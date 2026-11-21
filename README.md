# gh-workflows

Reusable GitHub Actions workflows (`workflow_call`) shared across `boma-marketing` repositories. They cover PR validation, publishing npm packages, and deploying serverless, SPA and Elastic Beanstalk services.

The workflows delegate most steps to composite actions in the private [`boma-marketing/gh-actions`](https://github.com/boma-marketing/gh-actions) repository (pinned to `v4`), checked out at run time using an SSH deploy key.

## Workflows

| Workflow | Purpose |
| --- | --- |
| [`pr_builder.yml`](.github/workflows/pr_builder.yml) | Install dependencies and run tests on pull requests |
| [`npm_publish.yml`](.github/workflows/npm_publish.yml) | Test and publish a package to GitHub Packages (`npm.pkg.github.com`) |
| [`sls_devtest.yml`](.github/workflows/sls_devtest.yml) | Build once, deploy a Serverless app to `dev` and `test` |
| [`sls_env.yml`](.github/workflows/sls_env.yml) | Build and deploy a Serverless app to a single named environment |
| [`spa_devtest.yml`](.github/workflows/spa_devtest.yml) | Build a SPA for `dev` and `test`, deploy to S3/CloudFront |
| [`spa_env.yml`](.github/workflows/spa_env.yml) | Build and deploy a SPA to a single named environment |
| [`svc_devtest.yml`](.github/workflows/svc_devtest.yml) | Build an Elastic Beanstalk service on arm64 and deploy to `dev` and `test` |

## Usage

Call a workflow from a job in your repository's workflow, referencing it by path and ref:

```yaml
name: PR

on:
  pull_request:

jobs:
  build-test:
    uses: boma-marketing/gh-workflows/.github/workflows/pr_builder.yml@main
    secrets:
      SHARED_ACTIONS_SSH_KEY: ${{ secrets.SHARED_ACTIONS_SSH_KEY }}
      GITHUB_PACKAGE_ACCESS_TOKEN: ${{ secrets.GITHUB_PACKAGE_ACCESS_TOKEN }}
```

Secrets are not inherited automatically; pass each one explicitly (or use `secrets: inherit` within the same organisation). Pin `@<tag-or-sha>` instead of `@main` if you need stable behaviour.

### Publish an npm package

Triggered by pushing a version tag. The tag must equal `v<package.json version>` and the tagged commit must be on `main`, otherwise the run fails before publishing.

```yaml
name: Publish

on:
  push:
    tags: ['v*']

jobs:
  publish:
    uses: boma-marketing/gh-workflows/.github/workflows/npm_publish.yml@main
    with:
      TEST_COMMAND: yarn test # optional, default shown
    secrets:
      SHARED_ACTIONS_SSH_KEY: ${{ secrets.SHARED_ACTIONS_SSH_KEY }}
      GITHUB_PACKAGE_ACCESS_TOKEN: ${{ secrets.GITHUB_PACKAGE_ACCESS_TOKEN }}
```

```sh
yarn version patch   # or edit package.json
git tag v1.2.3 && git push origin v1.2.3
```

The package is published with `yarn npm publish` using the job's own `github.token` (`packages: write`); the package access token is only used for installing and testing.

### Deploy a Serverless app

```yaml
jobs:
  deploy:
    uses: boma-marketing/gh-workflows/.github/workflows/sls_env.yml@main
    with:
      APP_NAME: my-app
      ENV_NAME: prod
    secrets:
      SHARED_ACTIONS_SSH_KEY: ${{ secrets.SHARED_ACTIONS_SSH_KEY }}
      GITHUB_PACKAGE_ACCESS_TOKEN: ${{ secrets.GITHUB_PACKAGE_ACCESS_TOKEN }}
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_ACCESS_KEY_SECRET: ${{ secrets.AWS_ACCESS_KEY_SECRET }}
      SERVERLESS_ACCESS_KEY: ${{ secrets.SERVERLESS_ACCESS_KEY }} # optional
```

## Inputs

| Workflow | Input | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `pr_builder` | `TEST_PARAMETERS` | no | `--passWithNoTests` | Arguments appended to `yarn test` |
| `pr_builder` | `TEST_COMMAND` | no | _empty_ | Extra command(s) run after `yarn test` |
| `npm_publish` | `TEST_COMMAND` | no | `yarn test` | Command run before publishing |
| `sls_devtest` | `APP_NAME` | yes | | Application name |
| `sls_devtest`, `sls_env` | `REQUIRED_LEGACY_SECRETS` | no | `true` | Whether the deploy action requires legacy secrets |
| `sls_env` | `APP_NAME`, `ENV_NAME` | yes | | Application and target environment |
| `spa_devtest` | `APP_NAME`, `DEV_CF_DISTRIBUTION_ID`, `TEST_CF_DISTRIBUTION_ID` | yes | | Application name and CloudFront distributions |
| `spa_env` | `APP_NAME`, `ENV_NAME`, `CF_DISTRIBUTION_ID` | yes | | Application, environment and CloudFront distribution |
| `svc_devtest` | `APP_NAME` | yes | | Application name |

## Secrets

| Secret | Used by | Description |
| --- | --- | --- |
| `SHARED_ACTIONS_SSH_KEY` | all | SSH deploy key with read access to `boma-marketing/gh-actions` |
| `GITHUB_PACKAGE_ACCESS_TOKEN` | all (required except in `pr_builder`) | Token with read access to private `@boma-marketing` packages |
| `AWS_ACCESS_KEY_ID` | `sls_*`, `spa_*`, `svc_devtest` | AWS access key ID for deployments |
| `AWS_ACCESS_KEY_SECRET` | `sls_*`, `spa_*`, `svc_devtest` | AWS secret access key |
| `SERVERLESS_ACCESS_KEY` | `sls_*` (optional) | Serverless Framework access key |

## Conventions

- Node.js 24 with Corepack enabled; Yarn (v4) is the package manager.
- Dependencies are installed via the shared `install-from-cache` action.
- Applications are expected to provide `yarn test` and `yarn build` scripts as relevant to the workflow used.
- `svc_devtest` expects `scripts/create-distributable-package.sh` in the calling repository and builds on `ubuntu-24.04-arm` so native modules match Graviton instances.
- `*_devtest` workflows deploy to `dev` and `test` in parallel after a successful build.

## Contributing

1. Branch from `main` and use [Conventional Commits](https://www.conventionalcommits.org/) (`feat(ci): …`, `fix(ci): …`).
2. Keep `workflow_call` inputs and secrets documented in the workflow file and in this README.
3. Changing inputs or secrets of an existing workflow is a breaking change for callers; add new inputs as optional with a default.
4. Open a pull request against `main`.

## Licence

[MIT](LICENCE)
