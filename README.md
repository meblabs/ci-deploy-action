# ci-deploy-action
GitHub Action to setup and run the deployment script in MEBlabs standard CI

## How to use v4
v4 uses AWS OIDC (no access keys) and forwards extra CLI flags to your `deploy/deploy.sh`.

### Minimal
```yml
name: deploy
on:
  push:
    branches: [release, staging]
jobs:
  deployment:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - name: deploy
        uses: meblabs/ci-deploy-action@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          token: ${{ secrets.MEBBOT }}
```

### With options and extra args
```yml
- name: deploy
  uses: meblabs/ci-deploy-action@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    token: ${{ secrets.MEBBOT }}
    aws-region: eu-west-1            # default
    node-version: 22.x               # default
    npm: "true"                      # install via npm ci and enable node cache
    lfs: "false"                     # pull Git LFS objects when "true"
    deploy-args: --version 1.2.3  # forwarded to deploy.sh
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
- `jq` is ensured. If missing it is installed, otherwise skipped.
- Runs `deploy/deploy.sh -e "$ENV" <deploy-args>`.

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

## How To Use v2.0 (legacy)
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
```

## How To Use v1.0 [Script customization] (legacy)
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
      - name: Extract env
        run: |
          branch_name=${GITHUB_REF#refs/heads/}
          if [ "$branch_name" = "release" ]; then
              echo "##[set-output name=ENV;]$(echo production)"
          else
              echo "##[set-output name=ENV;]$(echo $branch-name)"
          fi
        id: extract_env

      - name: deploy
        uses: meblabs/ci-deploy-action@v1.0
        with:
            AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
            AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            AWS_REGION: eu-west-1
            token: ${{ secrets.MEBBOT }}
            deploy-script: bash deploy.sh -e ${{ steps.extract_env.outputs.ENV }}
```

## Git tag update
```sh
git tag -af "v2.0" -m "version 2.0"
git push -f --tags
```
