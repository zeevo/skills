---
name: newt-hunt
description: Scaffold a random create-newt-app configuration, build a small toy app in it, and log bugs, friction, and improvement ideas. Use when asked to hunt for create-newt-app bugs or run a newt-hunt pass.
---

# newt-hunt

One pass: scaffold, build something small, record what broke. Findings go to a report; never file issues, push, or open PRs.

The argument is the path to a newt-app source checkout. If none was given, stop and ask for it. Call it `$SRC` below.

## 1. Update the source

```bash
git -C "$SRC" pull --ff-only
(cd "$SRC" && pnpm install && pnpm --filter create-newt-app build)
```

If the pull fails (dirty tree, diverged branch), stop and report it. Don't stash, reset, or switch branches.

## 2. Pick a config and scaffold

Run `node "$SRC/packages/create-newt-app/dist/index.js" --help` for the current flags. Pick one value per option at random. Use `--database postgres` only if `DATABASE_URL` is set.

Skip any combo already in `~/newt-hunt/runs/*/flags` from the last 24 hours.

```bash
RUN=~/newt-hunt/runs/$(date +%Y%m%d-%H%M%S); mkdir -p "$RUN"
git -C "$SRC" log -1 --format='%h %s' > "$RUN/commit"
APP=$(mktemp -d); cd "$APP"
node "$SRC/packages/create-newt-app/dist/index.js" toy <flags> 2>&1 | tee "$RUN/scaffold.log"
echo "<flags>" > "$RUN/flags"
```

## 3. Check the untouched scaffold

Free the ports first: `lsof -ti :3000,:3001 | xargs kill 2>/dev/null`. A leftover server on :3000 makes smoke results lie.

In `$APP/toy`, run `pnpm build`, `pnpm typecheck`, `pnpm lint`, `pnpm test`, then `"$SRC/scripts/smoke.sh" "<flags>" .`. Save each output under `$RUN/`.

## 4. Build a toy feature

Pick one: bookmarks, guestbook, habit tracker, link shortener. It must touch the database, sit behind auth, and use a Nest route when Nest is on. Follow the patterns the scaffold already uses. Keep it under an hour of work.

Re-run build, typecheck, lint, and test. Boot `pnpm dev`, then probe the new route with curl.

## 5. Report

Write `$RUN/findings.md`, one entry per finding:

```markdown
### <title>
- severity: bug | friction | improvement
- source: scaffold | my toy code
- commit: <contents of $RUN/commit>
- flags: <flags>
- repro: <commands>
- evidence: <error text or file:line>
```

Only `source: scaffold` entries matter. Before listing one, run `gh issue list -R zeevo/newt-app --state all --search "<keywords>"` and note any matching issue number.

## 6. Clean up

Kill anything on :3000 and :3001, then `rm -rf "$APP"`.

End with a three-line summary: commit and flags, finding count by severity, and the report path.
