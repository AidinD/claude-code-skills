---
name: catchup
description: Summarise what has changed on the current git branch and why - reads every changed file and its diff, then reports the themes and anything that looks incomplete. Use when the user says "/catchup", "what changed here", "bring me up to speed", or picks up a branch they have been away from.
---

Run `git diff main...HEAD --name-only` (or `git diff HEAD --name-only` if not on a feature branch) to get all changed files.

Then read each changed file and the diff. Summarize:
1. What files changed and what each change does
2. Any patterns or themes across the changes
3. Anything that looks incomplete, broken, or worth flagging

Be concise. This is a catch-up summary, not a code review - for an actual bug/quality pass on the same diff, use `/pr`'s review step instead.
