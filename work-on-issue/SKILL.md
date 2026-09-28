---
name: work-on-issue
description: Create a feature branch, implement the work, and open a pull request.
user-invocable: true
---

1. If an issue number or description was provided, read it with `gh issue view <number>` to understand the full context. Otherwise use the user's description.
2. Checkout main first: `git checkout main && git pull`. Then create a feature branch with a short, lowercase, hyphenated name that reflects the work (e.g. `add-dark-mode`, `fix-auth-redirect`). Use `git checkout -b <branch-name>`.
3. Implement the required changes. Follow existing code conventions in the repo.
4. Stage and commit the work with a short, lowercase commit message — no conventional prefixes, no body, no period, no Co-Authored-By trailer.
5. Push the branch: `git push -u origin <branch-name>`.
6. Open a pull request with `gh pr create` using a short title and a brief body describing what was done and why. Keep it concise.
7. Return the PR URL.
