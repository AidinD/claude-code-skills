---
name: kickoff
description: >-
  Start a new work session via the orchestrator. Use when the user wants to
  kick off a new task/job in its own session — triggers like "start a new
  session for X", "kickoff X", "spin up a session for Z". Drafts a
  self-contained brief, checks for an existing session on the topic to avoid
  duplicates, recommends model + effort, and spawns the session rooted in the
  correct repo. Does NOT run the work itself — it hands off.
---

# Kickoff — orchestrated new-session handoff

You are acting as the user's orchestrator. The goal is to launch a *new*
session for a piece of work, correctly set up, without the user having to
hand-craft the prompt or remember to root it in the right repo.

Requires an orchestrator layer that exposes session-spawning and
session-search primitives (here: `spawn_task`, `search_session_transcripts`).
If your environment doesn't expose equivalents, drop steps 2 and 7 and just
draft the brief for the user to start manually.

## Steps

1. **Understand the task.** If the request is vague (no clear goal, scope, or
   target project), ask 1–2 sharp clarifying questions. Don't over-ask — a
   one-line task is fine if the target repo is clear.

2. **Check for duplicates FIRST.** Run `search_session_transcripts` on the core
   topic. If an existing session already covers it, surface it and ask whether
   to *resume that one* instead of starting fresh. Avoiding duplicate sessions
   is a primary reason this skill exists — it's easy to start new rather than
   resume, which fragments context.

3. **Determine the target repo (rooting).** Map the task to its repo so the new
   session loads that repo's CLAUDE.md, skills, and permissions. Maintain your
   own project registry (e.g. a table in your personal hub's CLAUDE.md, or
   memory) mapping project name → repo path, something like:
   - `<project-name>` → `D:\Repo\<...>\<project-name>`
   - Personal/general hub → `<path to your general working directory>`
   Look up the target project in that registry to get its path; confirm with
   the user if it's not there or you're unsure, and do not trust a stale entry
   — verify it exists before rooting to it. For work not tied to a specific
   repo (research, planning, misc tasks), the general hub is fine.

4. **Ask the rooting mode** (unless the user already said). Two options, real
   trade-off:
   - **spawn (default):** one-click handoff via `spawn_task` with `cwd` set to
     the target repo. Caveat: the spawned session runs in a **fresh git
     worktree** (isolated branch), NOT the working dir on `main`. Good for
     isolated new work; conflicts with a "work on main by default" convention —
     the user merges back later.
   - **manual (main, no worktree):** don't spawn. Give the user the brief plus
     a one-liner to start it themselves via the VS Code extension / CLI opened
     in the repo folder, which yields a normal session on `main`.

5. **Recommend model + effort** for this specific task and put it at the TOP of
   the brief. `spawn_task` cannot set these, so the user sets them at session
   start (`/model`, `/effort`, or the app). Rubric:
   | Task shape | Model | Effort |
   |---|---|---|
   | Mechanical, well-specified (edits, docs, file moves) | Sonnet | low–medium |
   | Normal feature work | Sonnet high / Opus medium | medium |
   | Architecture, ambiguous design, hard debugging, cross-system | Opus | high–xhigh |
   | Interactive back-and-forth | Opus (fast mode) | medium |
   Default toward the cheapest tier that fits — it's easy to over-use Opus+high
   for work that didn't need it.

6. **Write the brief.** Self-contained (the new session has none of this
   conversation's context). Include: the recommended model/effort line, the
   goal, essential context and file paths, constraints, and a done-criterion.
   Keep it tight — a brief, not an essay. Match whatever artifact-language
   convention the project uses (e.g. "code/docs in English").

7. **Hand off.**
   - spawn mode: call `spawn_task` with a verb-first `title`, the brief as
     `prompt`, a plain-English `tldr`, and `cwd` = target repo. Then tell the
     user to set the recommended model/effort when they open it.
   - manual mode: print the brief and the start command; do not spawn.

8. **Track in a todo board if applicable** — create an `open` task for it if the target project has a matching task list.

## Notes

- This skill hands off; it never does the work in the current session.
- Always prefer resume over new when a relevant session exists.
- Be honest about the worktree trade-off every time — never silently spawn a
  worktree when the user expected main.
