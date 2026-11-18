# lambda-build-deploy-reusable-wf

![Built with Copilot](https://img.shields.io/badge/Built_with-Copilot-brightgreen?logo=github)&nbsp;![Release](https://github.com/subhamay-bhattacharyya-gha/lambda-build-deploy-reusable-wf/actions/workflows/release.yaml/badge.svg)&nbsp;![Commit Activity](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-gha/lambda-build-deploy-reusable-wf)&nbsp;![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-gha/lambda-build-deploy-reusable-wf)&nbsp;![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-gha/lambda-build-deploy-reusable-wf)&nbsp;![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-gha/lambda-build-deploy-reusable-wf)&nbsp;![File Count](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-gha/lambda-build-deploy-reusable-wf)&nbsp;![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-gha/lambda-build-deploy-reusable-wf)&nbsp;![Top Language](https://img.shields.io/github/languages/top/subhamay-bhattacharyya-gha/lambda-build-deploy-reusable-wf)&nbsp;![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/ea51658500e2653e96ecab6e4e9ea6d4/raw/lambda-build-deploy-reusable-wf.json?)

A reusable GitHub Actions workflow that packages an AWS Lambda function from a folder in the calling repository, uploads the package to S3, and updates the function code. All environment-specific values are passed in as inputs, so the same workflow works across accounts, regions, and runtimes.

---

## Contents

- [What it does](#what-it-does)
- [Workflow: `build-deploy-lambda.yaml`](#workflow-build-deploy-lambdayaml)
- [Inputs](#inputs)
- [Outputs](#outputs)
- [Prerequisites](#prerequisites)
- [Example usage](#example-usage)
- [Repository layout](#repository-layout)
- [Releasing](#releasing)
- [Contributing](#contributing)
- [License](#license)

---

## What it does

For the function named by `function_base_name`, the workflow:

1. Checks out the calling repository.
2. Validates the inputs and confirms `<functions_root>/<function_base_name>/` exists.
3. Sets up Python at `python_version`.
4. Authenticates to AWS with GitHub OIDC by assuming `arn:aws:iam::<aws_account_id>:role/<aws_oidc_role_name>`.
5. Installs `requirements.txt` (if present) into a build directory.
6. Copies the function source over the dependencies, removes `requirements.txt` and `__pycache__`, and zips the result.
7. Uploads the zip to `s3://<s3_bucket_name>/<deployed name>/<deployed name>-<commit SHA>.zip`.
8. Runs `aws lambda update-function-code` on the deployed function with the same zip and waits until the function is updated.

The deployed name is `<function_base_name>-<environment>-<aws_region>`, for example `test-lambda-devl-us-east-1`. The source folder uses only the base name: `<functions_root>/<function_base_name>/`.

Each run writes the S3 URI and function name to the job summary.

---

## Workflow: `build-deploy-lambda.yaml`

The workflow is triggered by `workflow_call`. It is not run directly; a caller workflow invokes it with `uses:`.

File: [.github/workflows/build-deploy-lambda.yaml](.github/workflows/build-deploy-lambda.yaml)

### Inputs

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `function_base_name` | Yes | — | Lambda function base name. Must match a folder under `functions_root`. The deployed function is `<function_base_name>-<environment>-<aws_region>`. |
| `s3_bucket_name` | Yes | — | S3 bucket that stores the deployment packages. |
| `aws_region` | Yes | — | AWS region for OIDC authentication, S3, and Lambda. |
| `aws_account_id` | Yes | — | AWS account ID that contains the OIDC role. |
| `aws_oidc_role_name` | Yes | — | Name of the IAM role assumed through GitHub OIDC. Pass the name only, not the ARN. |
| `python_version` | No | `3.12` | Python version used to install requirements and build the package. |
| `functions_root` | No | `lambda` | Folder in the calling repository that holds one sub-folder per function. |
| `environment` | No | `ci` | GitHub environment the job runs in. Controls which environment secrets and protection rules apply. Also part of the deployed function name. |
| `runner` | No | `ubuntu-26.04` | Runner label for the job. |

### Outputs

| Name | Description |
| --- | --- |
| `s3_uri` | S3 URI of the uploaded Lambda package. |

### Permissions

The workflow requests `contents: read` and `id-token: write`. A reusable workflow can only receive permissions that the caller grants, so the caller must include both.

---

## Prerequisites

Before the workflow can run successfully:

- **Function folder:** each function lives at `<functions_root>/<function_base_name>/` in the calling repository. Put dependencies in `requirements.txt` in that folder.
- **Lambda function:** a function named `<function_base_name>-<environment>-<aws_region>` must already exist in the target account and region. The workflow updates its code and does not create it.
- **S3 bucket:** the bucket must exist and the OIDC role must allow `s3:PutObject` on it.
- **OIDC trust:** the IAM role must trust GitHub's OIDC provider for the calling repository.
- **IAM permissions:** the role needs `lambda:UpdateFunctionCode` and `lambda:GetFunction` (used by the wait step) on the function.

---

## Example usage

This caller reads its configuration from repository and organization variables and runs only from `workflow_dispatch`. Set these variables before running it:

| Variable | Scope | Used for |
|---|---|---|
| `AWS_ACCOUNT_ID` | Repository | `aws_account_id` |
| `AWS_ENVIRONMENT` | Repository | `environment`. Must match a GitHub environment name in the calling repository. |
| `AWS_REGION` | Repository | `aws_region` |
| `LAMBDA_PACKAGES_S3_BUCKET_NAME` | Repository | `s3_bucket_name` |
| `PYTHON_VERSION` | Organization | `python_version` |
| `AWS_OIDC_ROLE_NAME` | Organization | `aws_oidc_role_name` |

```yaml
name: Deploy Lambda (caller example)

run-name: Deploy Lambda ${{ inputs.function_base_name || 'test-lambda' }} by ${{ github.actor }}

on:
  workflow_dispatch:
    inputs:
      function_base_name:
        description: Lambda function base name. The deployed name is <base>-<AWS_ENVIRONMENT>-<AWS_REGION>.
        type: string
        required: false
        default: test-lambda

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    name: Build and deploy test-lambda
    uses: subhamay-bhattacharyya-gha/lambda-build-deploy-reusable-wf/.github/workflows/build-deploy-lambda.yaml@main
    with:
      function_base_name: ${{ inputs.function_base_name || 'test-lambda' }}
      s3_bucket_name: ${{ vars.LAMBDA_PACKAGES_S3_BUCKET_NAME }}
      aws_region: ${{ vars.AWS_REGION }}
      aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
      aws_oidc_role_name: ${{ vars.AWS_OIDC_ROLE_NAME }}
      python_version: ${{ vars.PYTHON_VERSION }}
      functions_root: lambda
      environment: ${{ vars.AWS_ENVIRONMENT }}
      runner: ubuntu-26.04

  summary:
    name: Show deployed package
    needs: deploy
    runs-on: ubuntu-26.04
    steps:
      - name: Print S3 URI
        run: 'echo "Deployed package: ${{ needs.deploy.outputs.s3_uri }}"'
```

The `s3_uri` output comes from the called workflow's `package` job. The caller reads it as `needs.<caller job>.outputs.s3_uri`, as the `summary` job does.

Pin `@main` to a release tag (for example `@v1.0.0`) in production callers.

---

## Repository layout

```text
.github/
  workflows/
    build-deploy-lambda.yaml   # The reusable workflow
    release.yaml               # semantic-release on push to main
    create-branch.yaml         # Creates a feature branch when an issue is assigned
    notify.yaml                # Slack notification on new issues and PRs
    claude.yaml                # Claude Code on @claude mentions
scripts/plugins/               # Custom semantic-release plugins
CHANGELOG.md                   # Generated by semantic-release
```

---

## Releasing

Releases are automated with [semantic-release](https://github.com/semantic-release/semantic-release). A push to `main` runs the `Release` workflow, which reads [Conventional Commit](https://www.conventionalcommits.org/) messages to decide the next version, updates `CHANGELOG.md`, tags the release, and publishes it on GitHub.

Use `feat:` for a new feature, `fix:` for a bug fix, and `feat!:` or a `BREAKING CHANGE:` footer for a breaking change.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## License

MIT. See [LICENSE](LICENSE).
