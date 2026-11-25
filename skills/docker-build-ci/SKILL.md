---
name: docker-build-ci
description: Use this skill when creating, reviewing, or fixing a GitHub Actions workflow that builds or publishes container images, even if the user just says they want to "build and push my image on every commit", "publish to Docker Hub or GHCR from CI", "make my image work on ARM / Apple Silicon", "speed up Docker builds in CI", or "add an SBOM or provenance". Covers choosing Docker GitHub Builder reusable workflows or individual docker/* actions, action pinning, least-privilege permissions, registry login, pull request behavior, tags and labels with metadata-action, GitHub Actions cache, multi-platform builds, build secrets, SBOM and provenance attestations, and test-before-push.
license: Apache-2.0
compatibility: GitHub Actions. Current majors are docker/build-push-action v7, docker/setup-buildx-action v4, docker/setup-qemu-action v4, docker/login-action v4, docker/metadata-action v6, and docker/github-builder v1; the docker/* action majors run on Node 24 and need Actions Runner v2.327.1 or later on self-hosted runners.
---

# Docker Build in CI

## Overview

This skill provides rules for building and publishing container images from
GitHub Actions. Apply it when the main artifact is a workflow under
`.github/workflows/` that runs a Docker build. It covers workflow structure and
supply-chain safety; what goes inside the `Dockerfile` belongs to
`docker-build-strategies`.

## When to use this skill

Activate this skill when:

- Creating a workflow that builds, tests, or pushes a container image
- Reviewing or hardening an existing Docker build workflow
- Adding multi-platform (`linux/amd64`, `linux/arm64`) images to CI
- Adding build caching, SBOM, or provenance attestations to a CI build
- Debugging CI-only failures: cache export errors, missing credentials on pull
  requests, or attestations missing from a pushed image

## Do not use this skill when

Do not use this skill when:

- The main task is writing or optimizing the `Dockerfile` itself — use
  `docker-build-strategies`
- The project has no Docker setup yet — use `docker-project-foundations` first,
  then return here for CI
- The CI system is not GitHub Actions (GitLab CI, CircleCI, Jenkins, Buildkite).
  This skill's Buildx rules on cache backends and attestations still apply, but
  its workflow syntax and action inputs do not; say so instead of translating
  GitHub Actions YAML
- The task is deploying a built image to a cluster or platform, rather than
  building and publishing it

## Core guidance

### Choose the build path

Docker maintains two supported ways to build in GitHub Actions. Pick one per
workflow; do not mix them in the same job.

| Use | When |
|---|---|
| **Docker GitHub Builder** reusable workflow (`docker/github-builder/.github/workflows/build.yml`, or `bake.yml` for Bake files) | Default for publishing images. It distributes multi-platform builds across native runners, signs SLSA provenance and cache entries with GitHub OIDC, and keeps build logic outside the caller's control, so a workflow step cannot tamper with what gets built. |
| **Individual actions** (`setup-buildx-action`, `login-action`, `metadata-action`, `build-push-action`) | The job must run other steps around the build: test the image before pushing, start service containers, prepare files or an SSH agent for the build, or run on self-hosted or non-Linux runners. A reusable workflow runs in separate jobs and cannot see the caller's workspace or agent. |

`assets/github-builder-workflow.yml` and `assets/build-push-workflow.yml` are
complete starting points for each path.

### Pin actions and reusable workflows

- Pin every third-party action and reusable workflow to a full commit SHA with
  the version as a trailing comment:
  `uses: docker/build-push-action@c3c9e263c25d99ce0380d002d59b67737d91b0dc # v7.4.0`.
  A tag such as `@v7` can be moved to different code after you review it; a
  SHA cannot.
- Use the current majors listed in this skill's `compatibility` field. Older
  majors (`build-push-action@v5`/`@v6`, `setup-buildx-action@v3`) target older
  Node runtimes and miss current fixes. When updating an existing workflow, keep the inputs that
  are already there unless the change log removed them.
- Do not invent SHAs. Resolve a tag to its commit with
  `gh api repos/docker/build-push-action/commits/v7.4.0 --jq .sha`, and add a
  Dependabot `github-actions` entry so pins stay current:
  ```yaml
  # .github/dependabot.yml
  version: 2
  updates:
    - package-ecosystem: github-actions
      directory: /
      schedule:
        interval: weekly
  ```

### Permissions

- Set `permissions: contents: read` at the workflow level, then grant more only
  on the job that needs it. The default `GITHUB_TOKEN` scope may be broad, and
  every step in the job can use whatever the token is allowed to do.
- `id-token: write` — required by GitHub Builder to sign attestations and cache
  entries, and by any OIDC registry login. Grant it only on the build job.
- `packages: write` — only when pushing to GitHub Container Registry (`ghcr.io`).
- Never grant `write-all` or omit `permissions` on a job that pushes images.

### Registry authentication

- Docker Hub: store the username as a repository variable
  (`vars.DOCKERHUB_USERNAME`) and a Docker Hub **personal access token** as a
  secret (`secrets.DOCKERHUB_TOKEN`). Never use the account password: a token
  can be scoped to read/write and revoked on its own.
- GHCR: username `${{ github.repository_owner }}`, password
  `${{ secrets.GITHUB_TOKEN }}`, plus `packages: write`. No personal token is
  needed.
- Individual actions: log in with `docker/login-action` before the build step.
  GitHub Builder: pass credentials in the `registry-auths` secret as a YAML list
  of `registry`, `username`, `password`; do not add a separate login step.
- Never `echo` a credential or pass one on a `run:` command line. GitHub masks
  known secrets in logs, but derived or transformed values are not masked.

### Pull requests and untrusted code

- Build on `pull_request` to validate changes, but push only from trusted
  events: `push: ${{ github.event_name != 'pull_request' }}`.
- Pull requests from forks receive no repository secrets and a read-only
  token. A workflow that logs in unconditionally fails on fork PRs; gate the
  login step with the same condition as `push`.
- Never build pull request code from a `pull_request_target` or `workflow_run`
  workflow that has secrets or write permissions. Those events run with the
  base repository's privileges, so a `RUN` instruction in the PR's Dockerfile
  could read the secrets and push a poisoned image.

### Tags and labels

- Generate tags and OCI labels with `docker/metadata-action` (GitHub Builder:
  `meta-images` and `meta-tags` inputs) instead of hand-building tag strings.
  It derives tags from the event (branch, PR number, semver Git tag, commit SHA)
  and adds `org.opencontainers.image.*` labels such as `source` and `revision`.
- A typical tag set:
  ```yaml
  tags: |
    type=ref,event=branch
    type=ref,event=pr
    type=semver,pattern={{version}}
    type=semver,pattern={{major}}.{{minor}}
    type=sha
  ```
- Do not publish only `latest`. Every image needs an immutable tag (version or
  commit SHA) so deployments can pin and roll back.

### Caching

- Use the GitHub Actions cache backend:
  `cache-from: type=gha` and `cache-to: type=gha,mode=max`. `mode=max` also
  caches intermediate stages, which is what makes multi-stage builds fast.
  GitHub Builder: `cache: true` and `cache-mode: max`.
- When one workflow builds several images, give each its own `scope=`
  (GitHub Builder: `cache-scope`); otherwise each build overwrites the previous
  one's cache.
- Triggers such as `issue_comment`, `pull_request_target`, and `pull_request`
  with `types: [closed]` get read-only cache access. Keep `cache-from` there,
  omit `cache-to`, and warm the cache from `push` on the default branch.
  Granting write access to untrusted triggers enables cache poisoning.
- `ignore-error=true` on `cache-to` makes an export failure non-fatal; use it
  only when the cache is optional, because it hides every export error.
- BuildKit cache mounts (`RUN --mount=type=cache`) are not saved to the GitHub
  Actions cache. See `references/caching-in-ci.md` before adding a workaround.

### Multi-platform images

- Prefer native runners over QEMU emulation: emulated `arm64` builds of
  compiled languages are often many times slower. GitHub Builder distributes
  one platform per native runner by default (`distribute: true`) and merges the
  manifest for you.
- With individual actions, add `docker/setup-qemu-action` before
  `setup-buildx-action` and set `platforms: linux/amd64,linux/arm64`, or split
  the build into a per-platform matrix and merge the digests (see
  `references/multi-platform-and-attestations.md`).
- The default runner Docker Engine cannot `load: true` a multi-platform image.
  Push it, or test a single platform locally (see "Test before push").

### Build secrets

- Pass credentials the build needs through BuildKit secrets, never `build-args`.
  Build arguments are recorded in provenance attestations, so a token passed as
  a build argument is published with the image.
- Individual actions: `secrets: |` with `"id=${{ secrets.NAME }}"`, or
  `secret-envs: id=ENV_VAR` for a value set by an earlier step. GitHub Builder:
  the `build-secrets` secret, as a YAML mapping of IDs to
  `${{ toJSON(secrets.NAME) }}` values.
- The `Dockerfile` reads each one with `RUN --mount=type=secret,id=<id>`; see
  `docker-build-strategies` for the Dockerfile side.

### Attestations

- `build-push-action` adds provenance by default: `mode=max` for public
  repositories, `mode=min` for private ones. Set `provenance: mode=max`
  explicitly to get full provenance in private repositories.
- Set `sbom: true` to add an SBOM; it is never added by default.
- Attestations exist only on pushed images. `load: true` and the `docker`
  exporter drop them, so verify attestations on the registry image, not the
  runner's local copy.
- GitHub Builder signs attestation manifests when pushing (`sign: auto`) and
  outputs ready-to-run `cosign-verify-commands`. Use that output to verify a
  signature rather than writing the certificate identity by hand.

### Test before push

When a job must test the image before publishing (individual-actions path):

1. Build a single platform with `load: true` and a local test tag.
2. Run the tests against that tag with `docker run --rm`.
3. Build again with `platforms:` and `push: true`. The second build reuses the
   first build's cache, so the tested platform is not rebuilt.

`assets/build-push-workflow.yml` implements this sequence.

## Related skills

- For the `Dockerfile` itself — multi-stage builds, layer order, non-root user,
  and `RUN --mount=type=secret` — use `docker-build-strategies`.
- For a project with no Docker setup yet, use `docker-project-foundations`.
- For Compose stacks used in CI integration tests, use `docker-compose-patterns`.
- For removing builders, caches, or images (`docker buildx prune`,
  `docker image prune`), use `docker-destructive-guardrails`.
- To check a flag or input against current Docker Docs, use `docker-docs`.

## References

- `references/caching-in-ci.md` — Cache backends compared (gha, registry,
  local, inline), scopes, write restrictions, and cache mounts in CI
- `references/multi-platform-and-attestations.md` — Native-runner matrix with
  digest merge, QEMU, verifying platforms, SBOM, and provenance on a pushed image
- `references/sources.md` — Docker Docs pages and action repositories this
  skill was checked against, and how to re-verify pinned versions

## Assets

- `assets/github-builder-workflow.yml` — GitHub Builder workflow: multi-platform,
  cache, SBOM, signed provenance, Docker Hub and GHCR, no push on pull requests
- `assets/build-push-workflow.yml` — Individual-actions workflow: test before
  push, metadata tags, gha cache, SBOM, max provenance, gated login

## Checks

- `checks/verification.md` — Lint a workflow with actionlint, review it against
  this skill's rules, and verify a pushed image's platforms and attestations.
