# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Purpose

This repo publishes a reusable GitHub Actions workflow, `.github/workflows/build-deploy-lambda.yaml`. Other repositories call it with `uses:` to package a Python Lambda function from `<functions_root>/<function_base_name>/`, upload the zip to S3, and update the function code with `aws lambda update-function-code`.

The main artifact is the workflow. Any change to its inputs, outputs, or steps is a change to a public interface, so update [README.md](README.md) in the same change.

## Key files

- `.github/workflows/build-deploy-lambda.yaml`: the reusable workflow (`on: workflow_call`).
- `.github/workflows/release.yaml`: semantic-release on push to `main`.
- `.github/workflows/create-branch.yaml`, `notify.yaml`, `claude.yaml`: repo automation. Leave these alone unless the task names them.
- `scripts/plugins/`: custom semantic-release plugins, wired through `package.json` and `.releaserc.json`.

## Reusable workflow conventions

- Every environment-specific value (bucket, region, account, role, Python version, runner, environment) is an input. Do not hard-code them and do not read `vars.*` or `secrets.*` in the workflow. Callers own that configuration.
- `workflow_call` inputs only support `string`, `boolean`, and `number`. There is no `choice` type. Use `string` and validate in a step.
- A reusable workflow can only use permissions the caller grants. Keep `id-token: write` and `contents: read` declared, and say in the README that callers must grant them.
- Keep the job's `environment:` input-driven, with `ci` as the default.
- `function_base_name` is the base name. The source folder is `<functions_root>/<function_base_name>/`. The deployed Lambda name is `<function_base_name>-<environment>-<aws_region>`, built inside the workflow. Keep the folder check on the base name only.
- Keep the S3 key format `<deployed name>/<deployed name>-<GITHUB_SHA>.zip`, where the deployed name is `<function_base_name>-<environment>-<aws_region>`, unless the user asks to change it.
- The deploy step uses the zip built in that run, not the S3 object. Keep the two in sync if either changes.
- The deployed name is built inside the workflow from `function_base_name`, `environment`, and `aws_region`. Callers pass only the base name. Do not reintroduce a separate deployed-name input without asking.
- The caller pattern in the README reads `vars.*` in a `config` job and passes the values through job outputs. The environment comes from a `workflow_dispatch` choice. Keep the `vars.*` reads in the caller, not in the reusable workflow.
- When you add or rename an input, update all of these together: the `on.workflow_call` inputs, the "Print input variables" step, the README inputs table and caller example, and this file.
- The job summary is written once, in the final "Write job summary" step with `if: always()`. Add new summary rows there, not in the upload or deploy steps.
- Pin third-party actions to versions already used in this repo (`actions/checkout@v7.0.1`, `actions/setup-python@v7`, `aws-actions/configure-aws-credentials@v6`. Both run on Node 24, so avoid Node 20 actions). Runner: `ubuntu-26.04`.

## Validating changes

- Lint workflows with `actionlint` before committing. It was used to validate `build-deploy-lambda.yaml`.
- The workflow cannot be fully tested locally because it needs AWS OIDC, an S3 bucket, and an existing Lambda function. Say so in the summary if you could not run it end to end.

## Commits and releases

- Use [Conventional Commits](https://www.conventionalcommits.org/). Commit types decide the version bump: `feat:` is minor, `fix:` is patch, and `feat!:` or a `BREAKING CHANGE:` footer is major.
- Pushes to `main` run semantic-release. It updates `CHANGELOG.md` and creates a tag. Do not edit `CHANGELOG.md` by hand.
- Use `chore(deps)` or `chore(deps-dev)` for dependency bumps. Dependabot already uses that format.

## Documentation

- [README.md](README.md) is the public interface doc. Keep its inputs and outputs tables in sync with the workflow's `on.workflow_call` block.
- README examples use `@main`. Keep the note under them that production callers should pin a release tag.
