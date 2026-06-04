# ci-deploy-action

[![quality](https://github.com/meblabs/ci-deploy-action/actions/workflows/quality.yml/badge.svg)](https://github.com/meblabs/ci-deploy-action/actions/workflows/quality.yml)
![type](https://img.shields.io/badge/github-Composite%20Action-blue?logo=github)
[![](https://img.shields.io/static/v1?label=MEBlabs&message=%E2%9D%A4&logo=GitHub&color=%23fe8e86)](https://github.com/sponsors/meblabs)

GitHub Action to setup and run the deployment script in MEBlabs standard CI

## How to use v5
v5 uses AWS OIDC (no access keys), sets up Docker Buildx for you, and forwards extra CLI flags to your `deploy/deploy.sh`.

### Minimal
```yml
name: deploy
on:
  push:
    branches: [release, staging]
jobs:
  deployment:
    runs-on: ubuntu-24.04-arm
    timeout-minutes: 20
    permissions:
      id-token: write
    steps:
      - name: deploy
        uses: meblabs/ci-deploy-action@v5
        with:
          token: ${{ secrets.MEBBOT }}
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
```

### With semantic-release
Run [meblabs/semantic-release-action](https://github.com/meblabs/semantic-release-action) first, then deploy. `deployment` waits for `release` (`needs: release`) but still runs even if release produced no new version (`if: always()`).

Keep permissions minimal: a `permissions: {}` deny-all baseline at the top, and grant `id-token: write` only on `deployment` (it needs OIDC for `configure-aws-credentials`). The `release` job needs no token permissions — `semantic-release-action` authenticates with the `MEBBOT` PAT, not the `GITHUB_TOKEN`.

```yml
name: Release

on:
  push:
    branches:
      - release
      - staging

permissions: {}

jobs:
  release:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Semantic Release
        uses: meblabs/semantic-release-action@v3
        with:
          token: ${{ secrets.MEBBOT }}

  deployment:
    runs-on: ubuntu-24.04-arm
    timeout-minutes: 20
    needs: release
    if: always()
    permissions:
      id-token: write
    steps:
      - name: deploy
        uses: meblabs/ci-deploy-action@v5
        with:
          token: ${{ secrets.MEBBOT }}
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
```

### Inputs
| name           | required | default     | description |
|----------------|----------|-------------|-------------|
| `role-to-assume` | yes    | —           | AWS IAM role ARN to assume via OIDC |
| `token`        | yes      | —           | GitHub token used for checkout and repo ops |
| `aws-region`   | no       | `eu-west-1` | AWS region |
| `node-version` | no       | `22.x`      | Node version for optional npm step |
| `npm`          | no       | `"false"`  | When `"true"`, runs `npm ci` and enables Node cache |
| `lfs`          | no       | `"false"`  | When `"true"`, runs `git lfs checkout && git lfs pull` |
| `deploy-args`  | no       | `""`       | Extra flags appended to `bash deploy.sh` after `-e $ENV` |

> Note: boolean-like inputs must be strings (`"true"`/`"false"`) due to GitHub composite action input typing.

### Behavior
- Environment name is derived from branch: `release` -> `production`, otherwise the branch name.
- Action checks out the repository at the current commit.
- AWS credentials configured via `aws-actions/configure-aws-credentials` with OIDC.
- Docker Buildx is set up via `docker/setup-buildx-action` (hash-pinned) before the deploy script.
- `jq` is ensured. If missing it is installed, otherwise skipped.
- Runs `deploy/deploy.sh -e "$ENV" <deploy-args>`.

### Migration notes v4 → v5
- Docker Buildx is now set up inside the action — remove the standalone `docker/setup-buildx-action` step from your workflow.
- Bumped pinned actions to current majors (`checkout`/`setup-node` v6, `configure-aws-credentials` v6), which run on the Node 24 runtime.
- No input or OIDC changes: existing `@v4` callers can switch to `@v5` as-is.

### Migration notes v3 → v4
- New input: `deploy-args` to forward arbitrary flags (e.g., `--version 1.2.3`).
- Safer tooling: `jq` install only when missing.
- Removed PR-only `ref` from checkout to avoid failures in push workflows.
- Examples show npm cache when `npm: "true"` and correct string defaults.

---

## How To Use v3 | v2.1 (legacy)
```yml
name: deploy
on:
  push:
    branches:
      - release
      - staging
jobs:
  deployment:
    runs-on: ubuntu-latest
    steps:
      - name: deploy
        uses: meblabs/ci-deploy-action@v2.0
        with:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          token: ${{ secrets.MEBBOT }}
          npm: true | false (default false)
```
