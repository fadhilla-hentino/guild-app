# Deploy Extend App

A GitHub Actions workflow that builds your Extend app's container image, pushes it to the app's registry, and deploys it — on every push to `main`, or on demand.

It runs [AGS CLI](https://github.com/AccelByte/accelbyte-ags-cli) on a standard GitHub-hosted runner. The workflow is a plain YAML file you copy into your own repository and edit.

```
push to main  →  build image  →  push to app registry  →  deploy  →  wait for rollout
```

Workflow file: `.github/workflows/deploy-extend-app.yml`

> **New to GitHub Actions?** A *workflow* is this YAML file. GitHub reads it automatically and runs it on the events listed at the top — here, a push to `main` or a manual trigger. Each run executes on a *runner*: a fresh, temporary Linux machine GitHub provides, so nothing is installed on your own computer and every run starts from a clean slate. A workflow is made of *jobs*, and each job is a list of *steps* that run top to bottom. You don't install or host anything — commit the file and GitHub takes it from there.

## Prerequisites

- **An Extend app that already exists.** This workflow deploys to an app, it does not create one. Create the app first in the Admin Portal or with AGS CLI.
- **A `Dockerfile` in your repository root.** The build context is the repository root (`--work-dir .`).
- **An IAM client.** [Create an IAM client](https://docs.accelbyte.io/gaming-services/modules/foundations/identity-access/authorization/manage-access-control-for-applications/) with client type `confidential` and assign the required permissions listed below. Keep a copy of the `Client ID` and `Client Secret`.

    - For AGS Private Cloud customers:
        - `ADMIN:NAMESPACE:{namespace}:EXTEND:REPOCREDENTIALS` [READ]
        - `ADMIN:NAMESPACE:{namespace}:EXTEND:APP` [READ]
        - `ADMIN:NAMESPACE:{namespace}:EXTEND:DEPLOYMENT` [CREATE]
    - For AGS Public Cloud customers:
        - Extend > Extend app image repository access (Read)
        - Extend > App Management (Read)
        - Extend > Deployment Management (Create)

  These are the minimum permissions the workflow needs: read the app, mint registry credentials, create a deployment. Don't widen them by copying a permission set from other Extend tooling — this credential lives in a repository indefinitely, and the list above deliberately cannot create, update, or delete the app.

## Setup

### 1. Add the workflow file to your repository

The workflow must live at `.github/workflows/deploy-extend-app.yml` on your repository's default branch. If you started from the sample app, it's already there — skip to the next step. If you're adding deployment to your own repo, copy that file into the same path and commit it. GitHub only picks up a workflow once the file is committed to the branch, so it won't appear under the **Actions** tab until then.

### 2. Add repository variables

In your GitHub repository, go to **Settings → Secrets and variables → Actions → Variables**.

| Variable | Description | Example |
| --- | --- | --- |
| `AGS_BASE_URL` | Your AGS environment base URL | `https://dev.yourstudio.accelbyte.io` |
| `AGS_NAMESPACE` | The namespace the app lives in | `yourgame` |
| `EXTEND_APP_NAME` | The name of the existing Extend app | `guild-service` |

### 3. Add repository secrets

In your GitHub repository, go to **Settings → Secrets and variables → Actions → Secrets**.

| Secret | Description |
| --- | --- |
| `AGS_CLIENT_ID` | Confidential client ID |
| `AGS_CLIENT_SECRET` | Confidential client secret |

The first step of the workflow checks all five and fails with a specific message naming anything that is missing, so a misconfigured repository fails in about ten seconds rather than part-way through a deploy.

### 4. Push to `main`

That's the whole setup. The workflow runs on every push to the **`main`** branch (and on manual runs — see [Manual deploy and rollback](#manual-deploy-and-rollback)). If your repository's default branch has a different name, either rename it to `main` or change `branches: [main]` at the top of the workflow to match — otherwise nothing will trigger.

## Configuration

Tunable values live in the `env:` block at the top of the workflow.

| Variable | Default | Description |
| --- | --- | --- |
| `AGS_VERSION` | `0.5.1` | AGS CLI version to install. Drives both the download URL and the cache key. |
| `WAIT_LIMIT` | `600` | Seconds to wait for the rollout before giving up. |
| `WAIT_INTERVAL` | `10` | Seconds between rollout status polls. |
| `IMAGE_TAG` | commit SHA | Tag applied to the built image. |

> **On pinning.** `AGS_VERSION` fixes the CLI version rather than tracking the latest release. Upgrading is then a deliberate one-line edit, not something that lands in your pipeline unannounced.

### Concurrency and timeout

Two job-level safety settings sit outside the `env:` block:

- **`concurrency`** serializes deploys per branch, with `cancel-in-progress: false`. If a second push lands while a deploy is still running, it waits for the first to finish rather than racing it — two rollouts to the same app can't overlap, and an in-flight rollout is never cancelled part-way through (which could leave the app half-updated).
- **`timeout-minutes: 30`** caps the whole job. `WAIT_LIMIT` only bounds the rollout step; this covers the case where the image build itself hangs, so a stuck run can't run up to GitHub's 6-hour default before it's killed.

## How AGS CLI gets installed

Three steps handle this: cache, install, and PATH.

### Cache AGS CLI

```yaml
- name: Cache AGS CLI
  id: cache-ags
  uses: actions/cache@v4
  with:
    path: ~/.ags-cli
    key: ags-${{ env.AGS_VERSION }}-${{ runner.os }}-${{ runner.arch }}
```

Every workflow run starts on a clean runner, so without a cache the CLI archive is downloaded from GitHub Releases on every single deploy. Caching it means that download happens only on the first run and after a version bump — not on every deploy — so a slow or unavailable GitHub Releases endpoint won't fail an otherwise-cached deploy.

`~/.ags-cli` is where the install step puts the binary (see below).

The cache key is worth understanding, because it is what makes version upgrades work without any manual cache clearing:

- **`AGS_VERSION`** — a cache entry is immutable once written; GitHub will not overwrite an existing key. Putting the version in the key means bumping `AGS_VERSION` produces a new key, which misses, which triggers a fresh install of the new version. Leave the version out and you would be pinned to whatever binary was cached first, forever.
- **`runner.os` / `runner.arch`** — the cached artifact is a compiled binary. A Linux x86_64 build is not valid on any other target. If you ever add a matrix or move to a different runner image, the key changes with it.

The step sets `steps.cache-ags.outputs.cache-hit` to `'true'` only on an exact key match.

### Install AGS CLI

```yaml
- name: Install AGS CLI
  if: steps.cache-ags.outputs.cache-hit != 'true'
  run: |
    set -euo pipefail
    export CARGO_HOME="$HOME/.ags-cli"
    mkdir -p "$CARGO_HOME"
    curl --proto '=https' --tlsv1.2 -LsSf \
      "https://github.com/AccelByte/accelbyte-ags-cli/releases/download/v${AGS_VERSION}/accelbyte-ags-cli-installer.sh" \
      | sh
```

This runs only on a cache miss — a first run, a version bump, or an expired cache. On a hit it is skipped entirely.

- **`CARGO_HOME`** — the installer is a `cargo-dist` shell installer, which installs into `$CARGO_HOME/bin`. Overriding it redirects the binary into `~/.ags-cli/bin` instead of the default `~/.cargo/bin`, so it lands inside the directory the cache step covers. The two paths have to agree or the cache does nothing.
- **`--proto '=https' --tlsv1.2`** — refuse any protocol downgrade and set a TLS floor. Standard hygiene for a `curl | sh` install.
- **`-f`** — fail on an HTTP error response. Without it, a 404 body gets piped straight into `sh`, which is both useless and unsafe.
- **`-L`** — follow redirects. GitHub release assets redirect to a CDN.
- **`v${AGS_VERSION}`, not `latest`** — the download URL pins an exact version (the "On pinning" note above explains why).

### Add AGS CLI to PATH

```yaml
- name: Add AGS CLI to PATH
  run: echo "$HOME/.ags-cli/bin" >> "$GITHUB_PATH"
```

This one runs unconditionally, and it has to be its own step: a write to `$GITHUB_PATH` only affects *later* steps, not the step doing the writing, and the install step doesn't run on a cache hit — so PATH setup can't live inside it.

## Runner state and authentication

Three environment variables exist to make the CLI behave on a clean runner.

| Variable | Why |
| --- | --- |
| `AGS_NO_KEYCHAIN=1` | Hosted runners have no OS keychain — no macOS Keychain, no Windows Credential Manager, no running Linux Secret Service. Setting this goes straight to file-based token storage instead of attempting a keychain call that will fail. |
| `AGS_PROFILE=default` | Resolved before the CLI's first-run profile setup, so it works on a clean runner regardless of what is on disk. Without it the CLI fails with `No active profile`. |
| `AGS_HOME=$RUNNER_TEMP/ags-home` | Puts CLI state — including the access token — in a temp directory **outside the repository**. |

> **Why `AGS_HOME` matters more than it looks.** The image build uses the repository root as its Docker build context. Any CLI state written inside the repository would be visible to the build and could end up baked into an image layer. Pointing `AGS_HOME` at `$RUNNER_TEMP` keeps the token out of the build context entirely. Don't move it into the workspace.

Authentication is the standard client-credentials flow:

```yaml
- name: Authenticate
  env:
    AGS_CLIENT_ID: ${{ secrets.AGS_CLIENT_ID }}
    AGS_CLIENT_SECRET: ${{ secrets.AGS_CLIENT_SECRET }}
  run: ags auth login --grant client-credentials --no-input
```

The credentials are passed as environment variables scoped to this single step, not as command-line flags — flag values show up in process listings and are easier to leak into logs. `--no-input` makes the CLI fail rather than prompt if anything is missing.

## Build and push

```yaml
- name: Build and push image
  run: |
    ags extend image-upload \
      --namespace "$AGS_NAMESPACE" \
      --app "$EXTEND_APP" \
      --image-tag "$IMAGE_TAG" \
      --work-dir . \
      --dockerfile Dockerfile \
      --platform linux/amd64 \
      --login \
      --retry-limit 2
```

One command builds the image and pushes it to the app's own registry. `--login` handles registry authentication using the session from the previous step, so there is no second set of registry credentials to manage.

- **`--platform linux/amd64`** is deliberate. If you build the same image on an Apple Silicon machine without this flag you get an `arm64` image, which starts fine locally and then fails on the cluster with `exec format error`. Pinning the platform in CI means the image you deploy is the image the cluster can run.
- **`--image-tag` defaults to the commit SHA**, which gives every deploy an immutable, traceable tag. This is also what makes rollback trivial (below). Avoid `latest` here; it destroys both the audit trail and the rollback path.

## Deploy and wait

```yaml
- name: Deploy and wait for rollout
  run: |
    ags extend deploy-app \
      --namespace "$AGS_NAMESPACE" \
      --app "$EXTEND_APP" \
      --json "$(printf '{"imageTag":"%s"}' "$IMAGE_TAG")" \
      --wait --wait-limit "$WAIT_LIMIT" --wait-interval "$WAIT_INTERVAL" \
      --api-scope admin --api-version v5 \
      --format json --no-input --yes
```

`deploy-app` takes the image tag in a JSON request body (`--json`) rather than as a flag.

`--wait` polls until the rollout reaches a terminal state or the wait limit expires. The step captures the exit code and maps it to a human-readable result:

| Exit code | Result | Meaning |
| --- | --- | --- |
| `0` | deployed | Rollout completed and the app is healthy. |
| `3` | failed | Rollout reached a terminal failure. Check the app logs before retrying. |
| `6` | timed out | `WAIT_LIMIT` elapsed with the rollout still in progress. |
| other | failed | Something else went wrong — a bad request, an auth failure, a network error. |

> **Exit `6` is not a rollback.** A timeout means CI stopped watching, not that the deployment stopped. The rollout may still succeed a minute later. Check the app's actual status before redeploying, or you risk stacking a second deployment on top of one that was about to finish. If your app is legitimately slow to start, raise `WAIT_LIMIT` rather than treating timeouts as normal.

Either way, the final step writes a job summary — app, namespace, image tag, result, and a hint for codes `3` and `6` — so the outcome is visible on the run page without opening the logs.

## Adding tests

There's a commented placeholder between configuration and install:

```yaml
- name: Test
  run: make test
```

Anything that exits non-zero there stops the workflow before any image is built or pushed. Uncomment it and point it at your test command.

## Manual deploy and rollback

The workflow also has a `workflow_dispatch` trigger. Under **Actions → Deploy Extend App → Run workflow** you can choose a ref and optionally override the image tag.

**To roll back:** select the last known-good commit or tag as the ref and leave the image tag input empty. The workflow checks out that ref, rebuilds it, tags it with that commit's SHA, and deploys it.

> **Watch the image-tag override.** The tag input does not choose *what* to build — it only labels whatever the selected ref contains. Running against `main` with an older SHA typed into the tag field will build current `main` and push it under a misleading tag. Use the ref selector for rollback; use the tag input only when you want a non-SHA label such as a release tag.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `Missing repository variable …` or `Missing secret …` | The config check found something unset. The message names which one and where to set it. |
| `No active profile` | `AGS_PROFILE` or `AGS_HOME` was removed from the `env:` block. Both are required on a clean runner. |
| Image builds, app never becomes healthy | Usually a crash at startup — a missing environment variable or config the app needs at boot. Check the app logs in the Admin Portal. |
| `exec format error` in the app logs | Architecture mismatch. Confirm `--platform linux/amd64` is still present and that your base image has an `amd64` variant. |
| Auth fails at the login step | Confirm the IAM client is **Confidential**, not Public, and that `AGS_BASE_URL` points at the right environment. |
| CLI version bump doesn't take effect | The cache key includes `AGS_VERSION`, so this normally resolves itself. If the key was edited, clear the entry under **Actions → Caches**. |
| Deploy rejected as unauthorized | The IAM client is missing a permission. Compare it against the list in the prerequisites — the error response names the permission string it expected. |

Run `ags doctor` locally against the same base URL and client to check config, auth, and connectivity independently of CI.

## Security notes

- `AGS_CLIENT_SECRET` is a long-lived credential in a repository. Rotate it periodically and on personnel changes.
- Grant only the permissions listed in the prerequisites, scoped to one namespace. If the secret is compromised, the blast radius is whatever you granted it — and that list deliberately cannot delete the app.
- Keep `AGS_HOME` outside the workspace — see [Runner state and authentication](#runner-state-and-authentication) for why.
- Never move credentials from `secrets` into `vars`. Repository variables are not masked in logs.
- The `permissions: contents: read` block at the job level is intentional — this workflow never needs write access to the repository.