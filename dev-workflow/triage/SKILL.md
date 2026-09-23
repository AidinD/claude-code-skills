---
name: triage
description: Survey the user's Claude Code sessions and Jot board, then PROPOSE a human-gated cleanup batch - idle/finished sessions to archive and stale Jot items to close/revisit - so finished work stops accumulating. Read-only: it proposes, it never archives or edits anything itself. Use when the user says "/triage", "clean up my sessions", "what can I archive", "triage my board", or wants a tidy-up pass over lingering sessions and stale tasks.
---

# Triage

Surveys two sources - the Claude Code session list and the Jot board - and
proposes a cleanup batch: sessions that look finished/idle (archive
candidates) and Jot items that look stale (close/revisit candidates). It is a
proposal you say yes/no to. It reads only; it takes no destructive or
state-changing action itself.

This exists to serve an "ephemeral sessions, not a durable fleet"
philosophy: the unit worth tending is WORK/GOALS (which live in your
task-tracking board, e.g. Jot), not a hoard of long-lived sessions. Left
alone, finished sessions and done-but-not-closed tasks pile up and dilute the
signal of what's actually in motion. Triage is the periodic sweep that keeps
the pile down - by surfacing candidates for you to approve, following a
general "notice and propose, never act" pattern (human gating scaled to
blast radius: the bigger the potential impact of an action, the more it
requires an explicit human yes).

Treat the steps below as a strong default, not a rigid script. Use judgment;
flag genuinely ambiguous cases for the user rather than guessing.

Written against Jot, the author's own personal open-source todo app - substitute
your own task tracker's data file and status vocabulary if you use something
else. The session-metadata path below (`%APPDATA%\Claude\...`) is Windows-specific;
adjust for your OS.

## What this skill can and cannot do (read first, and say it in the output)

- It **reads** the session metadata files and your task-tracking board's
  data file (e.g. Jot's `todos.json`), and **proposes** a batch.
- It **cannot** archive a session. Archiving flips an archived flag in the
  desktop app's own per-session metadata file - that is a write action
  belonging to that companion desktop app, not something a normal Claude
  Code session should perform. Even the desktop app itself only does it on
  an explicit user click. So the output here is a **proposal** the user (or
  a future desktop-app action) executes - not something this skill performs.
- It **can** offer to move approved Jot items (e.g. an agreed stale `review`
  item back to `open`, or a confirmed-finished item to `done`), because
  `todos.json` is a shared file this environment already edits under the
  jot-task-tracking discipline - but ONLY after the user approves that
  specific item, and following the safe-write flow below. Never as part of
  the survey itself.

Be honest about this in the final output - present it as "here's what I'd
archive/close, you decide", not as "I cleaned up your sessions".

## Step 1 - Survey the sessions (read-only)

Session metadata lives in per-device folders under the Claude desktop app's
data dir. Locate the directory that actually contains the session files:

- Root: `%APPDATA%\Claude\claude-code-sessions` (i.e.
  `~/AppData/Roaming/Claude/claude-code-sessions`).
- The files sit under a nested `<accountId>/<deviceId>/` folder. Don't
  hardcode those - walk the tree (a few levels deep) and use the directory
  holding the most session metadata files (the live one).
- Each session is stored as one JSON file, named for its unique session ID.
  Parse each and read these fields (a standard, stable set of fields present
  on every session record, so they're safe to rely on):
  - `sessionId`, `cliSessionId` (the transcript filename; falls back to
    `sessionId` for early sessions where they coincided)
  - `title`, `cwd`, `model`, `effort`
  - `isArchived` (already-archived sessions are NOT candidates - skip them)
  - `lastActivityAt` (fall back to `lastFocusedAt`, then `createdAt`, then 0
    - a reasonable derivation used consistently across this environment)
  - `completedTurns`, `createdAt`

Idle duration = `now - lastActivityAt`. This is the primary "finished/idle"
signal.

Optionally corroborate with the last message's role. The transcript is
`~/.claude/projects/<encoded-cwd>/<cliSessionId>.jsonl`, where the encoded
cwd is the cwd with every non-alphanumeric character replaced by a hyphen
(e.g. `D:\Repo\Tools\example-project` -> `D--Repo-Tools-example-project`). Transcripts can be
large - read only the tail (~96 KB) and scan backwards for the last line
whose `type` is `user` or `assistant`. An assistant-ended, long-idle session
is a strong "delivered a final answer, nothing pending" archive candidate; a
user-ended one may have been left mid-thought. This corroboration is
optional - if reading transcripts is slow or noisy across many sessions,
idle duration + title alone is enough to propose from; say which signals you
used.

Do NOT modify any session file. Not even to "fix" one that looks malformed.

## Step 2 - Survey the Jot board (read-only)

Read your task-tracking board's data file (e.g. Jot's `todos.json`) as
UTF-8. **Strip a leading UTF-8 BOM defensively** before parsing: if the first
character code is `0xFEFF`, drop it, then `JSON.parse`. (Jot's own app writes
without a BOM, but an editor or legacy tool may have added one, and a raw
parse throws on it.)

The shape:
- `categories`: `{ id, name, color }`.
- `todos`: `{ id, text, status, description, images, categoryId, tags,
  priority, deadline, parentId, createdAt, completedAt }`.
  - `status` is one of `open` | `in-progress` | `review` | `done`.
  - `priority`: LOWER number = more urgent (0 is the default/no-priority
    bucket; negatives are most urgent). Sort ascending when ordering.
  - `deadline`: epoch milliseconds, or null.
  - `parentId`: set on a subtask (Jot nests exactly one level deep); null on
    a top-level goal.
  - `createdAt` / `completedAt`: epoch ms.

For triage, the interesting items are the **open**, **in-progress**, and
**review** ones (a `done` item is already closed - not a candidate). For
each, note its category name, status, age (`now - createdAt`, and for a
`review` item how long it's been sitting), priority, and deadline.

Read each candidate's `description` before proposing anything about it - per
jot-task-tracking, backward moves and context notes live there (and `images`
may hold screenshots under `jot-images\...`). A task you're about to call
"stale" may have a description explaining exactly why it's parked.

## Step 3 - Correlate sessions with Jot (optional but valuable)

A session is a safer archive candidate when it has **no open linked Jot
work**. To link a session to a category, normalize both the session's `cwd`
folder name (preferred) or `title` and each category `name` - lowercase,
strip everything except `[a-z0-9åäö]` - and take a bidirectional substring
match of at least 3 characters (longest match wins). E.g. a session with cwd
`my-scheduler-app` matches a category named "Scheduler".

Use this to sharpen the session proposal:
- Idle session **with** open/in-progress/review work in its matched category
  -> weaker archive candidate (there's live work it may still be the home
  for). Flag it rather than confidently proposing archive.
- Idle session with **no** matched category, or a matched category whose work
  is all `done` -> stronger archive candidate.

If a session's cwd/title matches nothing, that's fine - it just means no Jot
correlation is available; fall back to idle duration + last-message role.

## Step 4 - Propose a reviewable batch (the core; human-gated)

Present two clearly separated lists. Order each so the most clear-cut
candidates are at the top and the ambiguous ones are visibly flagged, not
buried.

**Archive candidates (sessions):** for each, show title, cwd, idle duration
(e.g. "idle 6 days"), last-message role if you read it, matched Jot category
and its open-work count if any, and a one-line reason ("assistant-ended,
idle 8 days, no open linked work"). Suggested default heuristics, all
tunable and all worth stating so the user can adjust:
- Idle >= ~7 days AND (no matched category OR matched category has no open
  work) -> propose archive.
- Idle >= ~14 days regardless of Jot -> propose archive, but note any open
  linked work so the user isn't surprised.
- Assistant-ended + idle beyond a configurable inactivity window (roughly a
  day, by default) + no open linked work -> "delivered, nothing pending"
  archive candidate.

**Close/revisit candidates (Jot items):** for each, show category, text,
status, age, priority, deadline, and a one-line reason. Suggested defaults:
- A `review` item untouched for a long time (e.g. >= ~7-10 days) -> "sitting
  in review - confirm done, or send back with feedback?" Do NOT auto-move it
  to `done`; review is the user's to confirm (jot-task-tracking).
- An `open`/`in-progress` item with no activity for a long time (e.g. >= ~30
  days) and no upcoming deadline -> "long untouched - still relevant, or
  close it?"
- An item whose `description` already explains it's parked/blocked -> surface
  the reason, don't propose closing it.

Then let the user act **per item or as a batch** - "archive all the sessions,
skip these two Jot items", or item-by-item yes/no. Make declining trivial;
the point is a low-friction review, not a wall of forced decisions.

## Step 5 - Execute only what's approved, safely (and only the Jot half)

- **Sessions:** you cannot archive them from here. For the approved archive
  candidates, output them as a clear list the user can act on in the desktop
  app (or hand to a future automated archive action there). State plainly
  that this skill did not and cannot archive them itself.
- **Jot moves the user explicitly approved:** edit `todos.json` following the
  safe-write flow used everywhere in this environment - re-read the freshest
  file immediately before writing (the Jot app may have flushed its own edit
  since Step 2), make a minimal targeted change to just the approved item(s),
  write UTF-8 **without a BOM**, 2-space JSON (matching Jot's own writer), via
  a temp file + atomic rename so an interrupted write can't leave the file
  torn. Never move anything to `done` unless the user explicitly said that
  item is done - default backward/uncertain moves to `open` or `in-progress`
  and leave the reason in the item's `description`.
- Re-survey briefly after acting so the final summary reflects reality.

## Step 6 - Summarize

End with: how many sessions surveyed and how many proposed for archive (and
that archiving is the user's/the desktop app's action, not done here); how
many Jot items proposed for close/revisit and what the user approved; and
which cases you flagged as ambiguous rather than deciding. Keep it short.

## Open questions / tradeoffs (flagging for review, not resolved here)

- **Idle thresholds are guesses.** 7/14/30 days are reasonable defaults but
  unvalidated against how the user actually works. If per-feature ephemeral
  sessions become the norm, "idle 3 days" may already mean "finished" and the
  thresholds should tighten. Worth making these configurable - and worth
  reusing the same configurable inactivity window a companion desktop app
  might already define, rather than inventing a second notion of "idle".
- **Should it propose Jot review -> done moves at all?** The house rule is
  that `review` is the user's to confirm, never auto-done. This skill deliberately
  only proposes it and requires explicit per-item approval. If even proposing
  "confirm done?" feels like pressure to rubber-stamp, drop that suggestion and
  only ever propose review -> in-progress/open (send-back) plus flagging age.
- **Reading every transcript tail can be slow** across a large session list.
  The skill treats last-message-role as optional corroboration for that
  reason - idle duration + Jot correlation may be enough. If it turns out the
  role signal materially changes which sessions get proposed, make it
  mandatory instead.
- **Overlap with a companion app's own attention spotlight.** If a desktop
  companion app already computes a needs-attention score, triage is the
  inverse (needs-LESS-attention / can-be-put-away). If both ship, they should
  share the same session read and Jot correlation rather than diverging on
  definitions of "idle" and "linked work".
