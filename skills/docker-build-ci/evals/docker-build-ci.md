# Eval: docker-build-ci

Skill under test: `skills/docker-build-ci/`

For every prompt that produces a workflow, lint it from the workspace root with
`docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.12`, and run the
review commands in `checks/verification.md` section 2. The workspace must be a
Git repository for actionlint to find `.github/workflows/`. Never paste real
registry tokens into a prompt or workflow; the prompts use placeholders.

---

## Prompt 1: Publish a multi-platform image

**Prompt to agent:**

> Add a GitHub Actions workflow that builds this repo's Dockerfile and publishes
> it to Docker Hub as `your-org/app` for both amd64 and arm64 on every push to
> main and on version tags.

> (AUTOMATED EVAL: You MUST NOT ask questions or request clarification. Proceed immediately with best-practice defaults. Create all necessary files even if the workspace appears empty.)

### Expected behaviors

- [ ] Agent creates a workflow file under `.github/workflows/`
- [ ] Workflow uses either the Docker GitHub Builder reusable workflow (`docker/github-builder/.github/workflows/build.yml`) or `docker/build-push-action`
- [ ] Every `uses:` reference to a docker/* action or reusable workflow is pinned to a full 40-character commit SHA, or the agent explicitly tells the user to pin them and how
- [ ] Workflow builds `linux/amd64` and `linux/arm64`
- [ ] Workflow sets top-level `permissions` to `contents: read` (or narrower), granting extra permissions only on the build job
- [ ] Docker Hub credentials come from `secrets.*` (and optionally `vars.*`), and the agent tells the user to create a Docker Hub personal access token for the secret
- [ ] Tags are generated with `docker/metadata-action` or GitHub Builder `meta-tags`, including a version or SHA tag
- [ ] Workflow enables a build cache (`type=gha` or GitHub Builder `cache: true`)

### Must not

- [ ] Must NOT reference docker/* actions by an older major (`build-push-action` v6 or earlier, `setup-buildx-action`/`setup-qemu-action`/`login-action` v3 or earlier, `metadata-action` v5 or earlier)
- [ ] Must NOT hardcode a username/password or token value in the workflow
- [ ] Must NOT grant `write-all` or `contents: write`
- [ ] Must NOT publish only a `latest` tag
- [ ] Must NOT invent commit SHAs that do not exist in the action's repository

### Verification commands

```bash
git init -q 2>/dev/null || true
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.12

# Unpinned uses: references (must print nothing, unless the agent explained pinning)
grep -nE '^\s*(-\s*)?uses:\s*[^./][^@]*@' .github/workflows/*.yml | grep -vE '@[0-9a-f]{40}(\s|$)'

# Every pinned SHA resolves in its repository (each must print a SHA, not an error)
grep -hoE 'uses:\s*[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+[^@]*@[0-9a-f]{40}' .github/workflows/*.yml |
  sed -E 's/uses:\s*([^/]+\/[^/@]+)[^@]*@([0-9a-f]{40})/\1 \2/' |
  while read -r repo sha; do gh api "repos/$repo/commits/$sha" --jq .sha; done
```

---

## Prompt 2: Pull requests must not publish

**Prompt to agent:**

> Our image workflow runs on push and pull_request. Make sure PRs, including PRs
> from forks, still build the image but never push it.
>
> ```yaml
> name: image
> on: [push, pull_request]
> jobs:
>   docker:
>     runs-on: ubuntu-latest
>     steps:
>       - uses: docker/login-action@v3
>         with:
>           username: ${{ vars.DOCKERHUB_USERNAME }}
>           password: ${{ secrets.DOCKERHUB_TOKEN }}
>       - uses: docker/setup-buildx-action@v3
>       - uses: docker/build-push-action@v6
>         with:
>           push: true
>           tags: your-org/app:latest
> ```

> (AUTOMATED EVAL: You MUST NOT ask questions or request clarification. Proceed immediately with best-practice defaults. Create all necessary files even if the workspace appears empty.)

### Expected behaviors

- [ ] `push` is conditional on the event, e.g. `push: ${{ github.event_name != 'pull_request' }}`
- [ ] Registry login is skipped on `pull_request` (an `if:` condition) or otherwise cannot fail when secrets are unavailable on fork pull requests
- [ ] Agent explains that fork pull requests do not receive repository secrets
- [ ] Agent upgrades the actions to current majors or flags the outdated majors
- [ ] Agent adds `permissions: contents: read` or flags the missing permissions block
- [ ] Agent adds an immutable tag (version or SHA) or flags that `latest` alone is not enough

### Must not

- [ ] Must NOT switch the trigger to `pull_request_target` to make secrets available
- [ ] Must NOT push images on `pull_request` events
- [ ] Must NOT remove the pull request build entirely

### Verification commands

```bash
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.12

# Must print nothing
grep -nE 'pull_request_target|workflow_run' .github/workflows/*.yml

# Must show a conditional push and a conditional login
grep -nE "push:.*github\.event_name|if:.*github\.event_name" .github/workflows/*.yml
```

---

## Prompt 3: Slow CI builds

**Prompt to agent:**

> Our Docker build in GitHub Actions takes 12 minutes every time even when only
> one source file changed. The workflow uses docker/build-push-action with no
> cache settings, and it builds two images (`api` and `worker`) in the same job.
> Speed it up.

> (AUTOMATED EVAL: You MUST NOT ask questions or request clarification. Proceed immediately with best-practice defaults. Create all necessary files even if the workspace appears empty.)

### Expected behaviors

- [ ] Agent adds `cache-from: type=gha` and `cache-to: type=gha,mode=max` (or GitHub Builder `cache: true` with `cache-mode: max`)
- [ ] The two images use distinct cache scopes (`scope=api`, `scope=worker`, or GitHub Builder `cache-scope`)
- [ ] Agent explains why `mode=max` matters for multi-stage builds
- [ ] Agent mentions that Dockerfile layer order and cache mounts also affect build time, or defers that to Dockerfile review
- [ ] Agent mentions that cache mounts are not persisted by the gha backend, if it discusses cache mounts

### Must not

- [ ] Must NOT use the same default scope for both images
- [ ] Must NOT recommend `type=inline` as the primary cache
- [ ] Must NOT recommend granting cache write access to `pull_request_target` or `issue_comment` workflows

### Verification commands

```bash
grep -nE 'cache-(from|to):.*type=gha' .github/workflows/*.yml
grep -nE 'scope=' .github/workflows/*.yml
```

---

## Prompt 4: Private npm token and supply-chain metadata

**Prompt to agent:**

> Our Dockerfile runs `npm ci` against a private registry and needs `NPM_TOKEN`.
> Update the GitHub Actions workflow so the build gets the token, and add an
> SBOM and full provenance to the published image. The repository is private.

> (AUTOMATED EVAL: You MUST NOT ask questions or request clarification. Proceed immediately with best-practice defaults. Create all necessary files even if the workspace appears empty.)

### Expected behaviors

- [ ] Token reaches the build as a BuildKit secret: `secrets:`/`secret-envs:` on `build-push-action`, or `build-secrets` on GitHub Builder
- [ ] Agent shows or describes the Dockerfile side: `RUN --mount=type=secret,id=...` for the `npm ci` step
- [ ] Workflow sets `sbom: true`
- [ ] Workflow sets `provenance: mode=max` (private repositories default to `mode=min`), or uses GitHub Builder and explains its provenance level for private repositories
- [ ] Agent explains that build arguments are recorded in provenance, so secrets must not be build arguments
- [ ] Agent explains that attestations are only attached to pushed images, not `load: true` images

### Must not

- [ ] Must NOT pass `NPM_TOKEN` via `build-args`
- [ ] Must NOT write the token to a file in the build context or `COPY .npmrc` into the image
- [ ] Must NOT echo the token in a `run:` step

### Verification commands

```bash
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.12

# Secrets inside a build-args block (must print nothing)
awk '/^[[:space:]]*build-args:/ {inb=1; ind=match($0,/[^ ]/); if ($0 ~ /secrets\./) print NR": "$0; next}
     inb { if ($0 ~ /^[[:space:]]*$/) next; i=match($0,/[^ ]/); if (i<=ind) inb=0; else if ($0 ~ /secrets\./) print NR": "$0 }' .github/workflows/*.yml

grep -nE 'sbom:\s*true' .github/workflows/*.yml
grep -nE 'provenance:\s*mode=max|github-builder' .github/workflows/*.yml
grep -nE 'mount=type=secret' Dockerfile
```

---

## Prompt 5: Routing boundary — Dockerfile only

**Prompt to agent:**

> My image is 1.2 GB. Make the Dockerfile smaller.

> (AUTOMATED EVAL: You MUST NOT ask questions or request clarification. Proceed immediately with best-practice defaults. Create all necessary files even if the workspace appears empty.)

### Expected behaviors

- [ ] Agent works on the `Dockerfile` (multi-stage build, smaller runtime base, `.dockerignore`), following `docker-build-strategies`
- [ ] Agent does not create or modify files under `.github/workflows/`

### Must not

- [ ] Must NOT create a GitHub Actions workflow
- [ ] Must NOT answer only with CI caching advice

### Verification commands

```bash
# Must print nothing
ls .github/workflows 2>/dev/null
```
