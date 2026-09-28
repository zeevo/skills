---
name: hunter
description: Check out a project's default branch and aggressively hunt for bugs by using the project for real, the way a new user would, then log every bug, friction point, and improvement idea. Use when asked to hunt for bugs, run a hunter pass, or dogfood a project.
---

# hunter

One pass: check out the default branch, use the project for real, record what broke. Findings go to a report. Issues are filed only when the user asks, after the report is done; never push or open PRs.

Reading code is not hunting. Every finding must come from something you ran and watched fail.

## Arguments

All optional:
- a path to the project checkout (default: the current repo)
- a focus area, feature, or flag set to hammer (default: pick one at random)

## 1. Check out the default branch

```bash
cd "<path>"
git fetch origin
REF=$(git symbolic-ref --short refs/remotes/origin/HEAD)   # e.g. origin/main
WT=$(mktemp -d)/hunt; git worktree add --detach "$WT" "$REF"
RUN=~/hunter/runs/$(basename "$(git rev-parse --show-toplevel)")/$(date +%Y%m%d-%H%M%S); mkdir -p "$RUN"
git -C "$WT" log -1 --format='%h %s' > "$RUN/commit"
```

Do all work in `$WT`. Never touch the user's working tree, stash, reset, or switch their branch.

## 2. Learn how the project is meant to be used

Read the README, CLAUDE.md/AGENTS.md, CONTRIBUTING, docs, `--help` output, package scripts, and any project skill that covers running it. Work out:
- what kind of thing it is: CLI, library, web app, server, TUI, desktop app, generator
- the install and setup steps a new user follows
- the main user-facing entry points, flags, and config options

Also pull the project's existing issues so you know what is already reported. Use `gh` for GitHub remotes and `glab` for GitLab:

```bash
gh issue list --state all --limit 500 --json number,title,state,url,body > "$RUN/issues.json"
# GitLab: glab issue list --all --per-page 100 --output json > "$RUN/issues.json"
```

Skim the open ones. Don't spend the pass chasing bugs that are already tracked.

If setup needs secrets or services you don't have (API keys, databases, cloud accounts), use local substitutes where the docs offer them. Otherwise skip that surface and note it.

## 3. Set up like a new user

Follow the documented setup literally in `$WT`. Every step that fails, is missing from the docs, or needs a workaround is a finding. Then run the project's own checks (build, typecheck, lint, test) and save each output under `$RUN/`.

Before booting anything that listens on a port, check the port is free (`lsof -i :<port>`). A leftover process makes results lie.

## 4. Use it for real

Pick a realistic task a user would actually do with this project and do it end to end:
- CLI or generator: run it with a random mix of flags and inputs, then use what it produced
- library: write a small consumer program in a scratch dir that imports the built package the way the docs say
- web app or server: boot it, click through or curl every route, submit forms, sign up, log in
- TUI or desktop app: drive it through its main flows

Record every command in `$RUN/commands.log`.

## 5. Push on it

Once the happy path works, try to break it:
- edge inputs: empty, huge, unicode, spaces in paths, special characters, missing and duplicate values
- conflicting or unusual flag and config combinations
- running things twice, out of order, or concurrently
- interrupting mid-operation (Ctrl-C, killing the process) and resuming
- restarting after state exists: rerun on an existing dir, reboot a server with data in it
- stale or missing prerequisites: no network, wrong runtime version if easy to switch
- error paths: are messages accurate and actionable, do exit codes match

Chase anything suspicious until you have a minimal repro or have ruled it out.

## 6. Report

Write `$RUN/findings.md`, one entry per finding:

```markdown
### <n>. <title>
- severity: bug | friction | improvement
- source: project | my code
- commit: <contents of $RUN/commit>
- repro: <minimal commands>
- expected: <what should happen>
- actual: <what happened>
- evidence: <error text or file:line>
```

Only `source: project` entries matter. Re-run each repro once from a clean state before listing it; drop anything that doesn't reproduce.

Then dedupe against existing issues. For each finding, check `$RUN/issues.json` and search with a few keyword variations (`gh issue list --state all --search "<keywords>"`, or `glab issue list --all --search "<keywords>"`). Read the body of any likely match to confirm it is the same problem, not just similar words.
- open issue covers it: drop the finding from the report. List it under a `## Already tracked` section at the end as one line (`<title>: #<number>`) so the user can see it was hit.
- closed issue covers it and it still reproduces: keep it as a regression, and reference the closed issue in `evidence`.
- no match: keep it.

For every kept finding, also write a ready-to-file issue body in `$RUN/issues/<n>.md` (what happened, repro, expected, actual, commit). The finding's title is the issue title. Match the tone and labels of the project's existing issues.

## Never

- file issues or comment on existing ones unless the user asks after seeing the report
- push or open PRs
- change the project's code to fix what you found (a scratch patch to confirm a cause is fine, in `$WT` only)
- report something from reading code alone

## 7. Clean up

Kill any processes you started, remove scratch dirs, and `git worktree remove --force "$WT"`.

Keep `$RUN/` (report and issue drafts); it is what filing uses later.

End with a short summary: commit and what you exercised, finding count by severity (plus how many were dropped as already tracked), the report path, and a numbered list of kept findings. Then offer to file them, e.g. "Want me to file any of these? Say `all` or give numbers."

## 8. File on request

Only when the user asks. For each finding they pick:
1. Search issues once more, since one may have been opened during the pass. Skip it if a match exists and tell the user.
2. File from its draft: `gh issue create --title "<title>" --body-file "$RUN/issues/<n>.md"` (or `glab issue create --title "<title>" --description "$(cat "$RUN/issues/<n>.md")"`).
3. Add the issue URL to its entry in `$RUN/findings.md`.

Report the created URLs when done.
