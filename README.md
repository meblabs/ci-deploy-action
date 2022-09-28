# ci-deploy-action
GitHub Action to setup and run the deployment script in MEBlabs standard CI

## How To Use v1.0 [Script customization]
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

## How To Use v2.0
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
