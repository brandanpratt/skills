# Multi-platform images and attestations in CI

## Choosing how to build each platform

| Approach | Speed | Effort | Use when |
|---|---|---|---|
| GitHub Builder, `distribute: true` (default) | Native speed per platform, in parallel | Lowest: one `platforms:` input | Publishing images from GitHub-hosted Linux runners |
| One runner with QEMU (`setup-qemu-action` + `platforms:`) | Emulated platforms are much slower, especially compiled languages | Low | Small images, interpreted languages, or infrequent builds |
| Hand-written per-platform matrix with digest merge | Native speed | Highest: you maintain the merge job | You need individual actions and cannot use GitHub Builder |

### QEMU on one runner

```yaml
- uses: docker/setup-qemu-action@99012661954931238ded8c8b007157a8430204e1 # v4.4.0
- uses: docker/setup-buildx-action@f87e5991a6d7451dcb8d9637bfbc97413f497069 # v4.4.1
- uses: docker/build-push-action@c3c9e263c25d99ce0380d002d59b67737d91b0dc # v7.4.0
  with:
    platforms: linux/amd64,linux/arm64
    push: true
    tags: ${{ steps.meta.outputs.tags }}
```

### Hand-written matrix

If you must write the matrix yourself, the shape is:

1. A matrix job per platform, each on a native runner for that platform.
   Each job builds one platform and pushes it **by digest only**, with no tag
   (`outputs: type=image,name=your-org/app,push-by-digest=true,name-canonical=true,push=true`),
   and uploads its digest as a workflow artifact.
2. A merge job that `needs:` the matrix, downloads all digests, and creates the
   tagged manifest list:
   `docker buildx imagetools create -t your-org/app:1.2.3 your-org/app@sha256:<amd64> your-org/app@sha256:<arm64>`.

Give each platform its own cache scope (`scope=app-linux-arm64`) so the matrix
jobs do not overwrite each other's cache. This is the logic GitHub Builder
already implements; prefer it unless a requirement rules it out.

### Loading multi-platform images on the runner

The runner's default Docker Engine can push multi-platform images but cannot
`load: true` them. Either test a single platform with `load: true`, or enable
the containerd image store with `docker/setup-docker-action` and the
`"containerd-snapshotter": true` daemon feature.

## Attestations

| Attestation | Default with `build-push-action` | Turn on or change |
|---|---|---|
| Provenance (SLSA) | `mode=max` in public repositories, `mode=min` in private ones | `provenance: mode=max` |
| SBOM | Off | `sbom: true` |

Rules:

- Attestations are attached only when the image is pushed to a registry.
  `load: true` and the `docker` exporter discard them.
- Max-level provenance records build arguments. Never pass secrets as
  `build-args`; a public repository publishes them by default. If that already
  happened, rotate the secret: deleting the image tag does not remove copies
  others have pulled.
- GitHub Builder chooses the provenance level itself (max for public, min for
  private repositories) and signs attestation manifests when pushing. To force
  max provenance in a private repository, use individual actions.

## Verifying a pushed image

Run these against the registry, after the workflow pushed the image:

```bash
# Platforms in the manifest list
docker buildx imagetools inspect your-org/app:1.2.3

# Provenance attestation
docker buildx imagetools inspect your-org/app:1.2.3 --format '{{ json .Provenance }}'

# SBOM attestation
docker buildx imagetools inspect your-org/app:1.2.3 --format '{{ json .SBOM }}'
```

The first command should list one entry per platform the workflow built. The
provenance and SBOM commands print an empty value when that attestation is
missing. For an image signed by GitHub Builder, run the `cosign verify`
commands from the workflow's `cosign-verify-commands` output.
