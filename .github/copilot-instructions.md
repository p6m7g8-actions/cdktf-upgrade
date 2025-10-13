# Copilot Coding Agent Onboarding


This repository is a **composite GitHub Action** that builds and deploys a CDKTF app. It also includes house-keeping workflows for labeling, PR title linting, auto-approve, and enqueueing PRs into GitHub’s Merge Queue after CI completes.

---

## High-level overview

- **Type:** GitHub Actions composite action repository.
- **Primary artifact:** `action.yml` (the Action that callers use via `uses: p6m7g8-actions/cdktf-deploy@<ref>`).
- **Languages & runtimes:** YAML + Bash on Ubuntu runners; Node.js (via a reusable setup action); AWS credentials through GitHub OIDC; OpenTofu CLI.
- **Key behaviors of the composite action:**
  - Checks out code, sets up Node, installs `cdktf-cli`, configures AWS creds via OIDC, installs OpenTofu, then runs `pnpm run deploy` with CDKTF/AWS env vars. **Important:** `pnpm run deploy` **must be defined by the _calling_ repo**; this repo does not contain an app to deploy.

### Layout map (edit here first)
- `action.yml` — the composite action steps and inputs (keep inputs stable unless coordinated with consumers).
- `.github/workflows/`  
  - `build.yml` — CI “no-op” that must pass and run on both `pull_request` and `merge_group`.
  - `auto-queue.yml` — enqueues the PR into Merge Queue when `Build` succeeds.
  - `auto-approve.yml` — auto-approves PRs for specific authors or label conditions.
  - `pull-request-lint.yml` — enforces Conventional Commit-style PR titles.
  - `pr-labeler.yml` — adds labels used by other automations.
- `.vscode/settings.json` — editor defaults for YAML formatting.

---

## How to build, validate, and avoid CI failures

### Always do this before opening a PR
1. **Keep the workflow name and job IDs stable.** The enqueue workflow triggers off the _workflow name_ `"Build"`. If you rename `build.yml`’s `name:` or change the job id `build`, update `auto-queue.yml` accordingly.  
2. **Do not change triggers lightly.** `build.yml` **must include** both `pull_request` and `merge_group` events so checks also run on merge-queue synthetic branches.  
3. **Conform PR title to the allowed types.** Use one of: `chore|ci|docs|feat|fix|major|perf|refactor|revert|style|test`. The PR-title workflow will fail otherwise.

### CI gates that must pass
- **Build**: fast “no-op” check that signals CI health and enables queueing. Runs on `pull_request` and `merge_group`.  
- **PR title lint**: validates the PR title format.  
- **Labeling & auto-approve**: labels are added automatically; some authors or labels are auto-approved.  
- **Auto-queue**: after `Build` succeeds on a PR, a `workflow_run` job runs `gh pr merge` to enqueue into the branch’s Merge Queue.

### Local and remote validation
- **This repo has no unit tests** and no application code to execute. Validation is via the GitHub workflows above.
- To test the composite action logic, use a **separate sandbox repo** that references this action with `uses: ./` (for local) or `uses: p6m7g8-actions/cdktf-deploy@<branch>` and provides:
  - a `package.json` with `"deploy"` script runnable by `pnpm run deploy`,
  - AWS OIDC-assumable role and required inputs: `aws_role`, `aws_session_name`, `cdk_deploy_account`, `cdk_deploy_region`.
- **Do not run `pnpm run deploy` in this repository**; it will fail because no CDKTF app or script is present here.

### Versions and tooling (as pinned)
- `cdktf-cli`: latest (installed at runtime).
- `aws-actions/configure-aws-credentials`: v5.1.0.
- OpenTofu: `1.10.6`.
- Node setup comes from a reusable action; follow its defaults unless coordination is required.

**Preconditions:** AWS role and permissions must exist in the calling environment.  
**Postconditions:** On consumer repos, a successful run implies the CDKTF deploy script completed without error.

---

## Project layout and architectural notes

### Composite Action (`action.yml`)
Steps, in order:
1. Checkout.
2. Node setup.
3. Install `cdktf-cli` and print version.
4. Configure AWS creds via OIDC.
5. Install OpenTofu.
6. Run `pnpm run deploy` with required env (`CDKTF_LOG_LEVEL`, `CDK_DEPLOY_ACCOUNT`, `CDK_DEPLOY_REGION`, `AWS_REGION`, `TERRAFORM_BINARY_NAME=tofu`).

**Do not** change input names without a coordinated version bump and consumer migration guidance.

### CI/Automation workflows
- **Build (`build.yml`)**
  - Triggers: `pull_request`, `merge_group`, `workflow_dispatch`.
  - Job `build`: echo “no-op” and exit 0. Keep as a quick required check.
- **Auto-queue (`auto-queue.yml`)**
  - Triggers on `workflow_run` of `"Build"` with `conclusion == success` and `event == pull_request`.
  - Enqueues exactly the triggering PR using `gh pr merge -R "$REPO" "$PR"` (relies on branch protection with Merge Queue and at least one required check).
- **PR Labeler (`pr-labeler.yml`)**
  - Adds `contribution/core` for specific author and then `auto-merge` when appropriate.
- **PR Title Lint (`pull-request-lint.yml`)**
  - Enforces Conventional Commit-style PR titles with a fixed list of `types`.
- **Auto-approve (`auto-approve.yml`)**
  - Auto-approves when not draft and either labeled `auto-approve` or authored by whitelisted users.

---

## Practical checklist for the agent

- Keep `Build` workflow name and `build` job id unchanged unless you also update `auto-queue.yml`.  
- Ensure `merge_group` remains in `build.yml` triggers.  
- When altering `action.yml`:
  - Maintain step ordering; changes may break consumer pipelines.
  - Keep OpenTofu and AWS credential steps intact unless there is a strong reason and coordinated rollout.
- When adding new inputs or env:
  - Provide sensible defaults or treat as required, and document them in `action.yml`.
- Trust this document first. Search the repo only if instructions are incomplete or appear incorrect.
