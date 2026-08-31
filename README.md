# overlay-scorecard-backend-module-dora-test

Test repository for Scorecard DORA metrics with GitHub

## Deployment test workflows

This repository includes GitHub Actions workflows to generate test deployment data through the GitHub Deployments API.

- `Create and Merge Deployment PR`:
  - Trigger: manual (`workflow_dispatch`)
  - Inputs:
    - `auto_merge` (boolean, default `true`)
    - `deployment_status` (`error`, `failure`, `inactive`, `in_progress`, `queued`, `pending`, `success`; default `success`)
    - `merge_delay_minutes` (number, default `5`)
  - Behavior: creates a test branch + PR, updates a single marker file (`deployment-simulations/latest.md`), applies `deployment-test`, and applies `deployment-status-<state>` label from input.
- `Create Test Deployment on PR Merge`:
  - Trigger: pull request merged into `main` with `deployment-test` label
  - Behavior: creates a test deployment under dummy 'production' environment and then sets deployment status.
  - Status rule: reads `deployment-status-<state>` labels on PR (`error`, `failure`, `inactive`, `in_progress`, `queued`, `pending`, `success`); defaults to `success` if none is set.
- `Mark Deployment Status`:
  - Trigger: manual (`workflow_dispatch`)
  - Inputs:
    - `deployment_id`
    - `status` (`error`, `failure`, `inactive`, `in_progress`, `queued`, `pending`, `success`)
  - Behavior: marks an existing deployment with selected status.

### Testing flow

1. Run `Create and Merge Deployment PR` with `auto_merge = true`, pick `deployment_status`, and set `merge_delay_minutes` (default `5`).
2. Confirm `Create Test Deployment on PR Merge` runs and creates deployment/status (only runs for PRs labeled `deployment-test`).
3. Optionally run `Mark Deployment Status` using the deployment id to add additional status events.

### Check deployments

Use these commands to verify that test deployments and statuses were created correctly.

```bash
# List recent deployments
gh api "repos/janus-qe/overlay-scorecard-backend-module-dora-test/deployments?per_page=20"

# Inspect one deployment by id
gh api "repos/janus-qe/overlay-scorecard-backend-module-dora-test/deployments/<deployment_id>"

# List status history for one deployment
gh api "repos/janus-qe/overlay-scorecard-backend-module-dora-test/deployments/<deployment_id>/statuses"
```
