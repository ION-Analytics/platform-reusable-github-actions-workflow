# platform-reusable-github-actions-workflow

Reusable GitHub Actions workflows for releasing and deploying with [cdflow2](https://github.com/mergermarket/cdflow2).

| Workflow | Description |
|---|---|
| `.github/workflows/deploy.yaml` | Release and deploy to `aslive` then `live`, using static AWS access keys. |
| `.github/workflows/aws-cdflow2-deploy.yaml` | Release and deploy to any chained groups of environments, using AWS OIDC. |

## aws-cdflow2-deploy

Runs `tests` (optional) → `release` → deploys, authenticating to AWS with OIDC (no long-lived keys).

Environments are passed as groups. Within a group, each env deploys only after the previous env succeeds. Groups run in parallel.

```
prepare ─ tests ─ release
                     ├─ deploy-stage1: aslive, aslive_eu-west-2      (parallel)
                     └─ deploy-stage2: live, live_eu-west-2          (after all of stage 1)
```

### Usage

```yaml
name: deploy

on:
  push:
    branches: [main]

# Required: a reusable workflow cannot exceed the caller's permissions
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    uses: ION-Analytics/platform-reusable-github-actions-workflow/.github/workflows/aws-cdflow2-deploy.yaml@main
    with:
      release-role: arn:aws:iam::111111111111:role/github-release
      env-groups: '[["aslive","live"],["aslive_eu-west-2","live_eu-west-2"]]'
    secrets:
      DATADOG_API_KEY: ${{ secrets.DATADOG_API_KEY }}
      DATADOG_APP_KEY: ${{ secrets.DATADOG_APP_KEY }}
```

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `env-groups` | yes | | JSON array of arrays of env names, e.g. `[["aslive","live"]]`. Max 4 envs per group. |
| `release-role` | yes | | IAM role ARN assumed via OIDC for the `release` job. |
| `aws-region` | no | `eu-west-1` | Default AWS region; overridden per env by the `AWS_REGION` variable. |
| `run-tests` | no | `false` | Run `./test.sh` before releasing. |
| `plan-only` | no | `false` | Run `cdflow2 deploy --plan-only` in every env: shows the plan without applying it. The release still runs, since deploy needs a release version. |

### Secrets

| Secret | Required |
|---|---|
| `DATADOG_API_KEY` | yes |
| `DATADOG_APP_KEY` | yes |

### Environment configuration

Each name in `env-groups` is both a GitHub environment and the cdflow2 env name. Create the GitHub environments (Settings → Environments) and set these variables on each:

| Variable | Required | Description |
|---|---|---|
| `AWS_ROLE_ARN` | yes | IAM role assumed via OIDC for the deploy. |
| `AWS_REGION` | no | Overrides the `aws-region` input. |
| `CDFLOW2_DEPLOY_ARGS` | no | Extra arguments appended to `cdflow2 deploy`, e.g. `--config-param region=eu-west-2`. Split on whitespace, so values containing spaces are not supported. |

Example:

| Environment | `AWS_ROLE_ARN` | `AWS_REGION` | `CDFLOW2_DEPLOY_ARGS` |
|---|---|---|---|
| `aslive` | `arn:aws:iam::111111111111:role/github-deploy` | | |
| `live` | `arn:aws:iam::222222222222:role/github-deploy` | | |
| `aslive_eu-west-2` | `arn:aws:iam::111111111111:role/github-deploy` | `eu-west-2` | `--config-param region=eu-west-2` |
| `live_eu-west-2` | `arn:aws:iam::222222222222:role/github-deploy` | `eu-west-2` | `--config-param region=eu-west-2` |

Approval rules configured on a GitHub environment apply to that env's deploy job.

### AWS setup

Each IAM role needs a trust policy for the GitHub OIDC provider (`token.actions.githubusercontent.com`) with a `sub` condition matching the calling repo. If you scope `sub` to environments (`repo:org/repo:environment:aslive`), list every env that uses the role. The `release` job runs outside any environment, so its role needs a branch or ref condition instead.

### Notes and limitations

- `release` and all deploy jobs run on the self-hosted runner (`aws`, `linux`, `self-hosted`). `prepare` and `tests` run on `ubuntu-latest`.
- A stage waits for the whole previous stage. If one group fails at stage 1, later stages are skipped for every group. For fully independent chains, call the workflow once per group.
- The limit of 4 envs per group comes from four fixed `deploy-stageN` jobs, since GitHub Actions cannot generate jobs dynamically. Add another copy of the job to raise it.
