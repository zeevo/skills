---
name: gardener
description: Triage a project's open issues by trying to reproduce each one on the default branch, and close only those that are verifiably no longer reproducible. Then rebase your own open PRs that have fallen behind and been idle for a few hours. Use when asked to garden, prune, or clean up stale issues or PRs.
---

# gardener

Two stages, in order:
1. **Issues:** walk the open issues, try to reproduce each one on current code, and close the ones that are provably gone. Everything else is left untouched: no comments, labels, or edits on issues that stay open.
2. **PRs:** rebase your own open PRs/MRs that are behind their base branch, but only ones nobody has touched in a few hours.

Closing a real bug is far worse than leaving a fixed one open. When in doubt, leave it.

## Arguments

All optional:
- a ref to test against (default: the remote's default branch)
- issue numbers to check (default: every open issue)
- a cap on issues per run (default: 10)
- PR idle threshold (default: 3h)

## Setup

Detect the forge from `git remote get-url origin`: GitHub uses `gh`, GitLab uses `glab`. Stop if neither is available and authenticated.

```bash
git fetch origin
REF=<ref or origin's default branch>
WT=$(mktemp -d)/garden; git worktree add --detach "$WT" "$REF"
```

Do all testing in `$WT`. Never touch the user's working tree.

## Stage 1: issues

### Pick issues

List open issues, oldest first. Skip, without comment:
- feature requests, questions, discussions, tracking issues, anything that isn't a defect report
- issues with activity in the last 14 days
- issues with an open PR/MR linked to them
- issues labelled or described as won't-fix, blocked, upstream, or keep-open

### Check each issue

1. **Read everything:** body, comments, linked issues, attached logs. Pin down the repro steps, environment, and expected vs actual behaviour. If there's no concrete repro you can run, leave it open.
2. **Reproduce on `$REF`:** follow the steps literally in `$WT`, with the same config, flags, and inputs. Actually run it; reading code is not reproducing. Try the reporter's variants and anything commenters added.
3. **Prove the repro is faithful:** check out the last commit before the issue was filed (`git rev-list -1 --before=<created_at> $REF`) and run the same steps. If the bug doesn't show there either, your repro is wrong, so leave the issue open.
4. **Find the fix:** search merged PRs/MRs and commits that reference the issue (`#N`, the title, key error text), plus the issue's timeline cross-references. Bisect between the pre-issue commit and `$REF` if nothing turns up. Read any candidate fix and confirm it addresses this issue's cause, not something adjacent.
5. **Check for partial fixes:** make sure every symptom and variant in the thread is gone, not just the headline one.

### Close only when all of these hold

- the bug reproduced at the pre-issue commit (or, if that commit can't be built, a merged change clearly fixes this exact cause)
- it does not reproduce on `$REF`, including every variant from the thread
- you ran the steps and saw the result yourself

Then close it with one comment:

```
Gardening. Closing as this issue appears no longer reproducible.

- Reproduced at <old sha>: <one line result>
- Not reproducible at <ref sha>: <one line result>
- Fixed by <PR/MR or commit link>, if found
```

GitHub: `gh issue close <n> --reason completed --comment "<comment>"`. GitLab: `glab issue note <n> -m "<comment>"`, then `glab issue close <n>`.

When the issues are done, remove the worktree (`git worktree remove --force "$WT"`) and move on to stage 2.

## Stage 2: stale PRs

List open PRs/MRs authored by you:

- GitHub: `gh pr list --author @me --state open --limit 100 --json number,title,headRefName,headRefOid,baseRefName,updatedAt,isCrossRepository,mergeable`
- GitLab: `glab mr list --author=@me --output json`

### Pick PRs

Skip these and report why:
- the head branch is on a fork (you may not be able to push)
- it was updated within the idle threshold. This counts commits, comments, reviews, and label changes, which is what you want to be conservative about
- it is already up to date: not behind its base branch and not conflicting. GitHub: `gh api repos/<owner>/<repo>/compare/<base>...<headRefOid> -q .behind_by` is 0 and `mergeable` isn't `CONFLICTING`

Everything left needs a rebase.

### Rebase each one

Work in a fresh detached worktree at the PR's head sha. Never check out the PR branch itself (it may already be checked out somewhere).

```bash
WT=$(mktemp -d)/rebase
git worktree add --detach "$WT" <head sha>
git -C "$WT" rebase origin/<base branch>
```

If it conflicts:
- resolve it only when the answer is obvious from both sides (e.g. both sides added independent entries to the same list, or a generated file you can regenerate). Then `git -C "$WT" rebase --continue`
- otherwise `git -C "$WT" rebase --abort` and report the conflicting files. Don't guess

Push with a lease pinned to the sha you started from, so a push that landed in the meantime makes this fail instead of being overwritten:

```bash
git -C "$WT" push --force-with-lease=<head branch>:<head sha> origin HEAD:refs/heads/<head branch>
git worktree remove --force "$WT"
```

If the lease is rejected, skip that PR and report it.

## Never

- close an issue you couldn't run the repro for
- comment on, label, assign, or edit issues you leave open
- change code, except resolving obvious rebase conflicts
- push anything other than a rebased PR branch, and never with a plain `--force`
- rebase a PR by another author, or one on a fork
- open, comment on, edit, merge, or close PRs
- reopen, lock, or delete anything

## Finish

Make sure every worktree you created is removed.

Report two tables:
- issues: issue, closed or left open, one-line reason
- PRs: PR, rebased / skipped / conflict, one-line reason. For rebased PRs, note that CI re-runs and local checkouts of those branches are now stale (`git reset --hard origin/<branch>`)
