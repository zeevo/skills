---
name: commit-and-push
description: Stage all changes, create a commit, then push to the current remote branch.
user-invocable: true
---

1. Run `git status` and `git diff` to understand what has changed.
2. Stage the relevant changed files (prefer specific files over `git add -A`).
3. Write a short, lowercase commit message — no conventional prefixes, no body, no period, no Co-Authored-By trailer. Examples: `more scaffolding`, `fix build`, `update styles`.
4. Commit and push to the current remote branch. If the branch has no upstream, use `git push -u origin <branch>`.
5. Report the commit SHA and confirm the push succeeded.
