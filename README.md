# overlay-scorecard-backend-module-dora-test

Test repository for Scorecard DORA metrics with GitHub.

The workflows here generate synthetic pull requests, commits, merges, deployments and deployment
statuses so DORA metrics (deployment frequency, lead time, change failure rate, time to
restore) have data to measure.

## Workflows

### Create and Merge Deployment PR (`create-and-merge-deployment-pr.yml`)

- **Trigger**: Manual (`workflow_dispatch`) and `workflow_call`.
- **Behaviour**: Creates N intermediary PRs (each held open, committed to periodically, then
  squash-merged), then a final PR labelled for deployment.
- **Inputs**:

| Input                          | Type            | Default   | Notes                                                                                                                           |
| ------------------------------ | --------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `intermediary_pr_count`        | number          | `0`       | How many intermediary PRs to create and merge before deployment PR. Max **20**                                                  |
| `intermediary_pr_open_minutes` | number          | `1`       | How many minutes each intermediary PR should stay open before merge, `count * minutes` must be ≤ **40**                         |
| `deployment_status`            | choice / string | `success` | Deployment status, `error`, `success`, `failure`, `inactive`, `in_progress`, `queued`, `pending`                                |
| `merge_delay_minutes`          | number          | `0`       | Final deployment PR merge delay when `auto_merge_deployment_pr`. Max **55**. `0` will wait 3 seconds for mergeability to settle |
| `auto_merge_deployment_pr`     | boolean         | `true`    | Automatically merge final deployment PR                                                                                         |
| `run_label`                    | string          | `""`      | for `workflow_call` to distinguish the same caller                                                                              |

### Create Test Deployment on PR Merge (`create-test-deployment-on-pr-merge.yml`)

- **Trigger**: A pull request merged into `main` carrying the `deployment-test` label.
- **Behavior**: Creates a deployment against a dummy `production` environment, then sets its
  status. Reads the `deployment-status-<state>` label on the PR; defaults to `success`
  if none is present.

### Mark Deployment Status (`mark-deployment-status.yml`)

- **Trigger**: Manual (`workflow_dispatch`).
- **Inputs**: `deployment_id`, `status`.
- **Behavior**: adds an extra status event to an existing deployment.

### Cleanup Old Deployments (`cleanup-old-deployments.yml`)

- **Trigger**: weekly `schedule` (`0 4 * * 0`, Sunday 04:00 UTC) and manual
  (`workflow_dispatch`).
- **Inputs**: `older_than_days` (number, default `60`), `dry_run` (boolean, default `false`).
- **Behavior**: Pages through all deployments, and for each one created before the cutoff sets
  it `inactive` (required — an active deployment cannot be deleted) and then deletes it. A
  job summary table reports how many were seen, matched, deleted and failed.

### Simulate DORA (`simulate-dora.yml`)

Runs a DORA scenario by chaining three calls to `Create and Merge Deployment PR`:

| Segment | PRs merged         | Deployment status |
| ------- | ------------------ | ----------------- |
| 1       | PR-A, PR-B, D1     | `failure`         |
| 2       | PR-C (the fix), D2 | `success`         |
| 3       | PR-D, D3           | `success`         |

- Trigger: `schedule` (`0 3 */4 * *`, i.e. days 1, 5, 9, 13, 17, 21, 25, 29 at 03:00 UTC)
  and manual (`workflow_dispatch`).
- Input: `pr_open_minutes` (number, default `2`) — how long each PR stays open. Scheduled
  runs always use `2`.
- Runtime: roughly **19 minutes** with the defaults. The segments run sequentially
  (`needs:`), and a concurrency group prevents overlapping runs.

## Manual testing flow

1. Run `Create and Merge Deployment PR`, pick a `deployment_status`, and optionally set
   `intermediary_pr_count` / `intermediary_pr_open_minutes` / `merge_delay_minutes`.
2. Confirm `Create Test Deployment on PR Merge` runs and creates the deployment and status.
3. Optionally run `Mark Deployment Status` with the deployment id to add further status
   events.

## Check deployments

```bash
# List recent deployments
gh api "repos/janus-qe/overlay-scorecard-backend-module-dora-test/deployments?per_page=20"

# Inspect one deployment by id
gh api "repos/janus-qe/overlay-scorecard-backend-module-dora-test/deployments/<deployment_id>"

# List status history for one deployment
gh api "repos/janus-qe/overlay-scorecard-backend-module-dora-test/deployments/<deployment_id>/statuses"
```

## Check workflow runs

```bash
# List the workflows in the repository (id, name, path, state)
gh api "repos/janus-qe/overlay-scorecard-backend-module-dora-test/actions/workflows" \
  --jq '.workflows[] | {id, name, path, state}'

# List runs of one workflow (status, conclusion, commit and creation time)
# The workflow can be addressed by file name (as below) or by the numeric id from command above.
gh api "repos/janus-qe/overlay-scorecard-backend-module-dora-test/actions/workflows/simulate-dora.yml/runs?per_page=20" \
  --jq '.workflow_runs[] | {id, run_number, status, conclusion, sha: .head_sha, created_at, url: .html_url}'

# Inspect one run by id
gh api "repos/janus-qe/overlay-scorecard-backend-module-dora-test/actions/runs/<run_id>"
```

```bash
gh workflow list
gh run list --workflow simulate-dora.yml --limit 20
gh run view <run_id>
```
