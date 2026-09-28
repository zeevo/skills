---
name: monitor-main-ci
description: Check the status of GitHub Actions workflows on the main branch.
user-invocable: true
---

1. Run `gh run list --branch main --limit 10` to get the most recent workflow runs on main.
2. If any runs are in progress, wait 20 seconds and re-check in a loop until they complete.
3. For any failed runs, run `gh run view <run-id>` to get details, then `gh run view <run-id> --log-failed` to show the failing log output.
4. Summarize the overall pipeline health: which workflows are passing, failing, or in progress, and the commit SHA each run is associated with.
5. If there are failures, call out the specific job and step that failed and suggest a fix if the cause is clear from the logs.
