# Example GitHub Actions Databricks Deployment

## This folder contains a simple but production grade GitHub Actions workflow for Databricks Asset Bundles that:

*	Installs the latest Databricks CLI (via the official `databricks/setup-cli` action).
*	Authenticates securely with Databricks using [GitHub OIDC workload identity federation](https://docs.databricks.com/aws/en/dev-tools/auth/provider-github) (no long lived secrets).
*	On PRs to any branch except `main`, validates, plans, and deploys the bundle to the target that matches the PR’s base branch (e.g. PR → dev branch → deploy to dev target).
*	On push (merge) to `main`, validates, plans, and deploys to prod.
*	Fails the workflow if validation or deployment fails.

Why OIDC (a.k.a. “token federation”) instead of PATs or M2M client secrets? It’s the current best practice for CI on Databricks: short lived tokens, least privilege, no secrets to rotate, and policy scoping by repo/branch/environment. Databricks documents the GitHub OIDC flow and the exact env vars (DATABRICKS_AUTH_TYPE=github-oidc, DATABRICKS_CLIENT_ID, DATABRICKS_HOST) and the need for `id-token`: write permissions in your job.  ￼

## What each section does (and why)

*	`on.pull_request` with `branches-ignore`: ['main']: runs for PRs not targeting main. We’ll still run full deploys to dev/staging so reviewers can see real resources. ([GitHub’s event filters are documented here](https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions).  ￼)
*	`on.push` to `main`: fires after the merge actually lands on `main` (the moment you usually want to deploy to prod).
*	`permissions`: we request only what’s necessary: `id-token`: write (OIDC) and `contents`: read. Databricks’ OIDC for GitHub requires the `id-token` permission.  ￼
*	`prepare` job: centralizes the branch→target mapping and whether the PR is from a fork. It sets:
*	`env_name`: GitHub Environment we’ll bind to (dev/staging/prod).
*	`bundle_target`: the Databricks bundle target (same string as in your databricks.yml).
*	`is_fork`: to avoid deploying code from forks (safe default).
*	deploy job:
*	`environment`: `${{ needs.prepare.outputs.env_name }}` is important for two reasons:
	1.	It selects the `Environment` scoped variables/approvals in GitHub (so you can require reviewers for prod), and
	2.	If you configure your Databricks [federation policy](https://docs.databricks.com/aws/en/dev-tools/auth/env-vars) with subject type = `Environment`, the issued OIDC token’s subject will reflect `environment`:prod (tight scoping recommended by Databricks).  ￼
*	`env` sets Databricks auth for the CLI via unified authentication: `DATABRICKS_AUTH_TYPE`=`github-oidc`, `DATABRICKS_CLIENT_ID`, and `DATABRICKS_HOST` (or you can read host from the bundle target).  ￼
*	`databricks/setup-cli@main` installs the latest CLI (that action is the [Databricks maintained installer](https://github.com/databricks/setup-cli)). If you prefer pinning versions for reproducibility, replace @main with a released tag (e.g. `@v0.272.0`).  ￼
*	`bundle validate` checks config syntax and identity; the [CLI exits non zero on errors](https://docs.databricks.com/aws/en/dev-tools/cli/bundle-commands) so the job fails fast. (By design, validation may print warnings for unknown properties; warnings don’t fail the job on their own.)  ￼
*	`bundle plan` prints a no‑side‑effects diff of actions that would be taken; super useful in PR logs.  ￼
*	`bundle deploy --auto-approve --fail-on-active-runs` does the non‑interactive deploy and fails if jobs/pipelines are currently running in that target (guardrail to avoid mid‑run mutations).  ￼
*	`skip_forks` job: for external forks, we skip deployment (GitHub doesn’t pass OIDC/secrets to forked PRs by default). You can swap this for a “validate‑only” job if you want purely offline checks.

## First time setup you must do

1.	Create/enable a Databricks Service Principal and grant it permissions in each target workspace (e.g. job/pipeline create/update, catalog/schema rights as applicable).
2.	Enable OIDC (workload identity federation) for GitHub in your Databricks account:
	*	Create a federation policy that restricts issuer to https://token.actions.githubusercontent.com and scopes subject to your repo/organd (optionally) Environment (recommended). Databricks provides CLI examples for creating the policy.  ￼
	*	This makes it possible for your workflow to exchange GitHub’s OIDC token for a Databricks OAuth token without any stored secrets.  ￼
3.	In GitHub, define:
	*	Repository variable `DATABRICKS_SP_CLIENT_ID` = your Databricks service principal’s application/client ID (non‑secret).  ￼
	*	Environment variables (under Settings → Environments):
		*	Environment dev: `DATABRICKS_HOST`=https://dbc-dev-123.cloud.databricks.com
		*	Environment staging: `DATABRICKS_HOST`=https://dbc-stg-123.cloud.databricks.com
		*	Environment prod: `DATABRICKS_HOST`=https://dbc-prod-123.cloud.databricks.com
		*	(Optional) Set required reviewers or wait timers on the prod environment for an extra approval gate before the deploy job runs.

> If you cannot use OIDC yet, fall back to OAuth M2M: set DATABRICKS_AUTH_TYPE=oauth-m2m, provide DATABRICKS_CLIENT_ID (variable) and DATABRICKS_CLIENT_SECRET (as a secret), and keep DATABRICKS_HOST. This is supported by the CLI’s unified auth model, but favor OIDC when possible.
