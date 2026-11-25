# Verification Runbook

Use these checks to verify a GitHub Actions workflow that builds container
images. Steps 1 and 2 run locally against the workflow file. Step 3 needs an
image the workflow has pushed.

## 1. Lint the workflow

Run [actionlint](https://github.com/rhysd/actionlint) from the root of the Git
repository (the directory that contains `.github/`); outside a Git repository
it exits with status `3` and "no project was found". It checks workflow syntax, expression
types, and job references, and runs shellcheck on `run:` steps:

```bash
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.12 -color
```

Exit status `0` means no problems were found; `1` means it reported problems.
The image is pinned to a release tag; check
[actionlint releases](https://github.com/rhysd/actionlint/releases) for a newer one.

## 2. Review against the skill rules

Each command prints the lines that violate a rule; empty output passes. Set
`WF` to the workflow file under review.

```bash
WF=.github/workflows/image.yml

# Actions and reusable workflows not pinned to a 40-character commit SHA
grep -nE '^\s*(-\s*)?uses:\s*[^./][^@]*@' "$WF" | grep -vE '@[0-9a-f]{40}(\s|$)'

# Outdated majors of docker/* actions
grep -nE 'docker/(build-push-action|bake-action)@.*# v[1-6]\.|docker/(setup-buildx-action|setup-qemu-action|login-action)@.*# v[1-3]\.|docker/metadata-action@.*# v[1-5]\.' "$WF"

# Broad token permissions
grep -nE 'permissions:\s*write-all|contents:\s*write' "$WF"

# Untrusted-code triggers (review any match by hand: must not build PR code with secrets)
grep -nE 'pull_request_target|workflow_run' "$WF"

# Secrets referenced inside a build-args block (they land in provenance)
awk '/^[[:space:]]*build-args:/ {inb=1; ind=match($0,/[^ ]/); if ($0 ~ /secrets\./) print NR": "$0; next}
     inb { if ($0 ~ /^[[:space:]]*$/) next; i=match($0,/[^ ]/); if (i<=ind) inb=0; else if ($0 ~ /secrets\./) print NR": "$0 }' "$WF"

# Only a floating tag published
grep -nE 'tags:\s*\S+:latest\s*$' "$WF"
```

Then confirm by reading the workflow:

- [ ] Workflow-level `permissions: contents: read`; extra permissions only on
      the build job (`id-token: write` for GitHub Builder or OIDC,
      `packages: write` only for GHCR).
- [ ] Images are never pushed on `pull_request`, and registry login is skipped
      or harmless when secrets are absent (fork pull requests).
- [ ] Tags come from `docker/metadata-action` or GitHub Builder `meta-tags`,
      and include an immutable version or SHA tag.
- [ ] `cache-from`/`cache-to` use `type=gha` (or a registry cache), with
      `mode=max`, and a distinct `scope` per image when several are built.
- [ ] Build secrets use `secrets`, `secret-envs`, or GitHub Builder
      `build-secrets`, never `build-args`.
- [ ] Docker Hub logins use a personal access token secret, not a password.

## 3. Verify a pushed image

After a run on a trusted event (for example a push to `main`), inspect the
image in the registry. Replace the reference with one the workflow pushed:

```bash
IMAGE=your-org/app:sha-abc1234

# One entry per platform the workflow built
docker buildx imagetools inspect "$IMAGE"

# Provenance attestation present (empty output means missing)
docker buildx imagetools inspect "$IMAGE" --format '{{ json .Provenance }}'

# SBOM attestation present when sbom: true (empty output means missing)
docker buildx imagetools inspect "$IMAGE" --format '{{ json .SBOM }}'
```

For images built by GitHub Builder with signing enabled, run the commands from
the workflow's `cosign-verify-commands` output and confirm they succeed.

## 4. Confirm caching

Re-run the workflow without source changes. The build log should mark the
dependency and compile steps `CACHED`, and the job summary's build summary
should show cache utilization. If every step rebuilds, check the cache scope
and whether the trigger has cache write access (see
`references/caching-in-ci.md`).
