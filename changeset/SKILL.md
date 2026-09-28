---
name: changeset
description: Add a changeset to the current branch for changed packages.
user-invocable: true
---

The skill may be invoked with an optional bump type argument: `/changeset minor`, `/changeset patch`, `/changeset major`.

1. If the bump type was passed as an argument, use it. Otherwise ask the user: "major, minor, or patch?"
2. Run `git diff origin/main --name-only` to see which packages have changed.
3. Look at `.changeset/config.json` to understand the `fixed` groups — packages in the same fixed group always version together, so only include one representative package per group.
4. Write the changeset file directly to `.changeset/<short-description>.md` using this format:
   ```
   ---
   "package-name": minor
   ---

   short description of the change
   ```
   - Use a kebab-case filename matching the branch or feature (e.g. `add-shadcn-option.md`)
   - Only list one package per fixed group (changesets applies the bump to all in the group)
   - Do not use the interactive `pnpm changeset` CLI — write the file directly
5. Stage and commit the changeset file with message `add changeset`, then push.
