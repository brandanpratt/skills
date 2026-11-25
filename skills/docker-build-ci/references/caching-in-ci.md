# Caching Docker builds in CI

A CI runner starts with an empty BuildKit cache, so without an external cache
backend every build runs every instruction. This reference compares the
backends and explains the CI-specific limits.

## Backends compared

| Backend | Configure | Use when | Limits |
|---|---|---|---|
| `gha` | `cache-from: type=gha` / `cache-to: type=gha,mode=max` | Default on GitHub-hosted runners | Shares the repository's GitHub Actions cache quota; entries are evicted when unused or over quota. Only the current branch, its base branch, and the default branch are readable. |
| `registry` | `cache-from: type=registry,ref=your-org/app:buildcache` / `cache-to: type=registry,ref=your-org/app:buildcache,mode=max` | Images are large, the gha quota is too small, or the cache must be shared across repositories or CI systems | Needs registry write access on every run that exports. Use a dedicated cache tag; never point it at a tag you deploy. |
| `local` + `actions/cache` | `cache-to: type=local,dest=${{ runner.temp }}/.buildx-cache,mode=max,reset=true` | You need explicit control over cache keys | `reset=true` (Buildx 0.35.0+) stops old blobs piling up in the directory. More moving parts than `gha`. |
| `inline` | `cache-to: type=inline` | Rarely; legacy setups | Supports only `mode=min`, so intermediate stages are never cached. Prefer `registry`. |

`mode=min` (the default) caches only layers in the final image. `mode=max`
also caches intermediate stages, which is where the expensive dependency
install and compile steps of a multi-stage build live. Use `mode=max` unless
cache size is a problem.

## Scopes

The `gha` backend stores cache under the scope `buildkit` by default. When one
workflow builds several images, each build overwrites the previous one's cache.
Give each image its own scope and use it in both directions:

```yaml
cache-from: type=gha,scope=api
cache-to: type=gha,scope=api,mode=max
```

GitHub Builder takes `cache-scope` and adds the platform to the scope for
distributed builds, so per-platform caches do not overwrite each other.

## Write restrictions

GitHub gives some triggers read-only cache access in the default-branch
context: `issue_comment`, `pull_request_target`, and `pull_request` with
`types: [closed]` after a merge. A `cache-to: type=gha` export there fails with
`error writing layer blob: failed to reserve cache`, after the image was
already built and pushed.

- In those workflows keep `cache-from` and drop `cache-to`.
- Warm the cache from a `push` workflow on the default branch.
- For post-merge builds, trigger on `push` to the target branch instead of
  `pull_request` with `types: [closed]`.
- Do not raise cache access for untrusted triggers. A writable cache shared
  with untrusted code lets that code plant layers that later trusted builds
  reuse (cache poisoning). GitHub Builder signs and verifies its cache entries
  with GitHub OIDC for this reason.

`ignore-error=true` on `cache-to` makes export failures non-fatal. It does not
grant write access, and it hides every export error, so use it only when the
cache is optional.

## Legacy cache service errors

GitHub shut down the v1 cache service API on April 15, 2025. The error
`This legacy service is shutting down, effective April 15, 2025` means the
runner's Buildx or BuildKit is too old. Minimums: Buildx 0.21.0, BuildKit
0.20.0, Docker Compose 2.33.1, and Docker Engine 28.0.0 when building with the
`docker` driver and the containerd image store. GitHub-hosted runners are
already current; on self-hosted runners, update them or let
`docker/setup-buildx-action` install a current Buildx.

## Cache mounts

BuildKit cache mounts (`RUN --mount=type=cache,target=/root/.npm`) live in the
builder, not in image layers, so no cache backend exports them and they start
empty on every CI run. Layer caching with `mode=max` already skips an unchanged
dependency-install step, which covers most projects. Only when the cache mount
itself must survive (for example, a compiler cache across source changes),
Docker Docs document the third-party
`reproducible-containers/buildkit-cache-dance` action combined with
`actions/cache`. It is not maintained by Docker; pin it to a commit SHA and
review it like any other third-party action before adding it.

## Verify the cache is used

On a second run with no source changes, the build log should show `CACHED` for
the dependency and compile steps. The build summary that `build-push-action`
writes to the job summary also reports cache utilization.
