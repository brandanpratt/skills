# Sources

This skill was checked against these sources on 2026-11-24. Re-check them when
a new major version of any action ships.

## Docker Docs

- [GitHub Actions overview](https://docs.docker.com/build/ci/github-actions/)
- [Cache management with GitHub Actions](https://docs.docker.com/build/ci/github-actions/cache/)
- [GitHub Actions cache backend](https://docs.docker.com/build/cache/backends/gha/)
- [Multi-platform image with GitHub Actions](https://docs.docker.com/build/ci/github-actions/multi-platform/)
- [SBOM and provenance attestations with GitHub Actions](https://docs.docker.com/build/ci/github-actions/attestations/)
- [Push to multiple registries](https://docs.docker.com/build/ci/github-actions/push-multi-registries/)
- [Using secrets with GitHub Actions](https://docs.docker.com/build/ci/github-actions/secrets/)
- [Test before push](https://docs.docker.com/build/ci/github-actions/test-before-push/)
- [`docker buildx imagetools inspect`](https://docs.docker.com/reference/cli/docker/buildx/imagetools/inspect/)

## Action and workflow repositories

| Repository | Version pinned in assets | Commit |
|---|---|---|
| [docker/github-builder](https://github.com/docker/github-builder) | v1.20.0 | `bb1eb9fb69408ac52395fb21f49c6b52d564529a` |
| [docker/build-push-action](https://github.com/docker/build-push-action) | v7.4.0 | `c3c9e263c25d99ce0380d002d59b67737d91b0dc` |
| [docker/setup-buildx-action](https://github.com/docker/setup-buildx-action) | v4.4.1 | `f87e5991a6d7451dcb8d9637bfbc97413f497069` |
| [docker/setup-qemu-action](https://github.com/docker/setup-qemu-action) | v4.4.0 | `99012661954931238ded8c8b007157a8430204e1` |
| [docker/login-action](https://github.com/docker/login-action) | v4.8.0 | `8aa5831fb8d48d8f76594c7ddde61ae58019f550` |
| [docker/metadata-action](https://github.com/docker/metadata-action) | v6.2.0 | `dc802804100637a589fabce1cb79ff13a1411302` |
| [actions/checkout](https://github.com/actions/checkout) | v7.0.1 | `3d3c42e5aac5ba805825da76410c181273ba90b1` |

The current majors of the docker/* actions (build-push v7, setup-buildx v4,
setup-qemu v4, login v4, metadata v6) switched to the Node 24 runtime, which
requires Actions Runner v2.327.1 or later. Their release notes list no input
renames that affect the workflows in this skill.

## Re-verifying a pin

```bash
# Latest release tag
gh api repos/docker/build-push-action/releases/latest --jq .tag_name

# Commit a tag points to (dereferences annotated tags)
gh api repos/docker/build-push-action/commits/v7.4.0 --jq .sha
```
