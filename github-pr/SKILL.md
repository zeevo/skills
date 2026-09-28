---
name: github-pr
description: Ship a single requested change as its own focused GitHub pull request. Use for each atomic change the user asks for — branch off the default branch, implement, verify it works, add a changeset if the repo uses them, commit in the repo's style, push, open a concise PR, keep it rebased and conflict-free, and monitor CI until it settles. Triggers on "make a PR for X", "ship this as a PR", "raise a PR", or any standalone change meant to land on its own. Not for exploratory work, throwaway experiments, or changes the user explicitly wants committed directly.
---

Ship exactly one atomic change as one focused, reviewable PR. Prefer several small PRs over one big one.

## Workflow

1. **One change, one PR.** Keep the PR to a single logical change. If the request bundles independent changes, split them into separate PRs (or check with the user which to do first). Don't let scope creep in — every changed line should trace to this request.

2. **Honor the project first.** Read `CLAUDE.md` / contributing docs before touching anything. They override the defaults below: commit-message style, PR body format, whether Changesets are used, branch naming, any "no AI attribution" rule. If the repo has a PR template, fill it out — but keep each section to a line or two.

3. **Branch off the up-to-date default branch — never commit to it.** Find the default branch (usually `main`), `git checkout <default> && git pull`, then `git checkout -b <short-kebab-name>` describing the change (e.g. `fix-auth-redirect`, `add-oxc-linter`).

4. **Implement surgically.** Only what the request needs. Match the surrounding code's style and idiom.

5. **Verify before pushing.** Run the checks the change warrants — build / lint / test — and, when the change has runtime surface, actually exercise it (run the app, hit the endpoint, scaffold the output, screenshot the UI) rather than trusting types alone. Don't push something you haven't seen work.

6. **Changeset (if the repo uses it).** If a `.changeset/` directory exists, add one — invoke the `changeset` skill or write the file directly. Pick the bump from the change: patch for fixes/docs, minor for features, major for breaking. Name the file for the change, and **never** name it `readme`/`README` — on a case-insensitive filesystem that clobbers `.changeset/README.md`.

7. **Commit and push** in the repo's convention (short, lowercase, no prefix if that's the house style). `git push -u origin <branch>`.

8. **Open the PR** with `gh pr create`: a short title and a *terse* body — one sentence, or at most 2–3 short bullets saying what changed and why. Assume the reviewer reads the diff; the body orients them, it doesn't restate the code. Skip headings, test plans, background sections, file-by-file walkthroughs, and any "Changes"/"Summary" scaffolding. Prefer no body over a padded one. Still verify the change (step 5); just don't transcribe the verification into the PR. Follow the repo's PR conventions if they say otherwise.

9. **Monitor CI.** Arm a *background* monitor with the Monitor tool — never a foreground poll loop — so results stream in as notifications while you keep working:

   ```bash
   prev=""
   while true; do
     s=$(gh pr checks <PR#> --json name,bucket 2>/dev/null) || { sleep 30; continue; }
     cur=$(jq -r '.[] | select(.bucket!="pending") | "\(.name): \(.bucket)"' <<<"$s" | sort)
     comm -13 <(echo "$prev") <(echo "$cur")
     prev=$cur
     jq -e 'all(.bucket!="pending")' <<<"$s" >/dev/null && break
     sleep 30
   done
   ```

   Set `persistent: true` and a specific `description` like `CI checks on PR #<PR#>`. Each printed line is one check settling; the loop exits when none are pending. Don't poll manually — a monitor event is not a user reply.

   If a check fails, inspect it (`gh pr checks <PR#>`, then `gh run view <run-id> --log-failed`), report the specific job/step, push a fix, and arm a fresh monitor (the previous one has already exited). To watch several PRs, use one monitor whose loop iterates the numbers and prefixes each line with `#<PR#>`.

   A job that fails in seconds with `steps=0` is infrastructure, not your code — check `gh api repos/<owner>/<repo>/actions/runs/<id>/jobs` for the step count before debugging the diff.

10. **Check mergeability.** Once the PR exists, confirm it can actually merge:

    ```bash
    gh pr view <PR#> --json mergeable,mergeStateStatus
    ```

    `mergeable: CONFLICTING` means conflicts — resolve them (see below). `mergeStateStatus: BEHIND` means the base moved — rebase. `UNKNOWN` means GitHub is still computing; re-check in a few seconds. Do this again before reporting, and any time CI has been running a while.

11. **Report** the PR URL, its CI status, and its mergeability. Leave merging to the user unless they've said to merge.

## Staying current with the base

Rebase liberally — a branch that's behind is a branch whose green CI is stale. Rebase whenever the base branch has moved, not just when GitHub blocks the merge:

```bash
git fetch origin
git rebase origin/<default>
git push --force-with-lease
```

Always `--force-with-lease`, never a bare `--force`. Rebase (don't merge the base in) so the PR stays a clean single-purpose diff, unless the repo's conventions say otherwise or someone else is committing to the branch — then merge instead so you don't rewrite their commits out from under them.

**Conflicts.** Resolve them yourself when the intent is clear: keep the base's version of anything unrelated to this change, keep yours for the lines this PR is actually about. Re-run the checks from step 5 after resolving — a conflict resolution is new code, and rebasing invalidates the CI run that was green. If the conflict is in code you don't understand or the correct resolution is genuinely ambiguous, stop and ask rather than guessing.

Changeset files rarely conflict but can duplicate — if a release consumed your changeset while the PR was open, drop the stale file instead of resurrecting it.

Re-arm the CI monitor after every force-push; the old run's results no longer describe the branch.

## Stacked PRs

When a change builds on an open PR, branch off that PR's branch and open the new PR with `--base <that-branch>`, stating the stack and merge order in the body. Rebase onto that base branch — not the default branch — and rebase the whole stack bottom-up when the default branch moves.

**The footgun:** GitHub only retargets a stacked PR to the default branch when its base branch is **deleted** at merge. If the base branch survives, merging the upper PR dumps its commits into the dead base branch — the merge *looks* fine but never reaches the default branch. Two rules:

- Tell the user the merge order and to **delete each branch on merge** (or enable auto-delete on the repo).
- After any stacked merge, verify the commits actually reached the default branch (`git log origin/<default>`); if one landed sideways, recover with a branch off the default branch that merges the stack tip, resolving conflicts toward the tip and dropping any changeset the earlier release already consumed.

## After it lands

When the user says it's merged, switch back to the default branch, pull, and delete the merged local branch before starting the next one.
