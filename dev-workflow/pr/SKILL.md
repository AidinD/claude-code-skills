---
name: pr
description: Prepare and open a pull request - reviews the diff for bugs, stages, commits in the repo's own style, pushes, and opens the PR with a summary and test plan. Use when the user says "/pr", "open a PR", "make this a pull request", "ship this branch", or asks to turn the current work into a PR for review.
---

Follow these steps in order:

1. Run `git status` to see what's staged and unstaged
2. Run `git diff` to review all changes
3. Invoke the /code-review skill to review the diff for correctness bugs - fix anything obvious before continuing
4. Stage all relevant files (avoid secrets, binaries, or unrelated changes)
5. Commit if there are uncommitted changes, following the repo's commit message style from `git log --oneline -5`
6. Push the branch to origin
7. Create a PR using `gh pr create` with:
   - A short title (under 70 chars)
   - A body with: ## Summary (2-3 bullets), ## Test plan (checklist)
8. Return the PR URL

If anything looks risky or unclear, stop and ask before proceeding.

Step 3 is deliberately `/code-review` (bugs) rather than `/simplify` (cleanup): a
PR about to be read by someone else is the wrong moment to discover a defect, and
tidiness can land in a follow-up. Run `/simplify` first as a separate pass if the
diff is messy enough to be hard to review.
