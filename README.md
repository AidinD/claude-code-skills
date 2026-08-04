# Claude Code skills

A curated sample of personal [Claude Code](https://docs.claude.com/en/docs/claude-code) skills — instruction files that teach Claude a specific workflow and when to trigger it automatically.

These 9 are pulled from a larger personal/work skill library and picked to show range: everyday dev workflow automation, multi-agent orchestration design, and small self-contained tooling. A couple reference "an orchestrator app I maintain" without naming it — that's a private project, kept out of this sample deliberately, not an oversight.

## What's here

### dev-workflow

| Skill | Triggers on | What it does |
|---|---|---|
| [`catchup`](dev-workflow/catchup/SKILL.md) | "what changed here", "bring me up to speed" | Summarizes a git branch's diff into themes and gaps — a catch-up, not a code review. |
| [`kickoff`](dev-workflow/kickoff/SKILL.md) | "start a new session for X", "kickoff X" | Drafts a self-contained brief, checks for duplicate sessions, recommends model/effort, hands off to a fresh session rooted in the right repo. |
| [`pr`](dev-workflow/pr/SKILL.md) | "/pr", "ship this branch" | Reviews the diff for bugs, commits, pushes, opens the PR with a summary and test plan. |
| [`skill-builder`](dev-workflow/skill-builder/SKILL.md) | "make a skill for this" | Turns a repeated workflow into a new skill via a short interview, then writes the file. |
| [`triage`](dev-workflow/triage/SKILL.md) | "clean up my sessions", "triage my board" | Surveys sessions and task board, proposes a cleanup batch — read-only, human approves every action. |

### orchestration

| Skill | Triggers on | What it does |
|---|---|---|
| [`council`](orchestration/council/SKILL.md) | "let two AIs debate this", "council on X" | Adversarial multi-agent debate — opposed advocates, a prosecutor attacking all of them, a judge that must pick one answer and name its own strongest objection. Built to defeat the failure mode where two LLM instances told to "discuss" just agree with each other. |
| [`ship-review`](orchestration/ship-review/SKILL.md) | "review this before I ship it", "/ship-review" | Adversarial pre-ship review pipeline: independent fresh-context reviewer, required E2E evidence, fix-or-escalate, stops short of pushing. Includes a 15-item failure catalog of real historical near-misses the pipeline was built to catch. |

### tooling

| Skill | Triggers on | What it does |
|---|---|---|
| [`gdrive-pdf-extract`](tooling/gdrive-pdf-extract/SKILL.md) | reading a view-only Google Drive PDF | Extracts text from copy-protected PDFs by jumping a virtualized viewer to force lazy-loaded pages to render. |
| [`summary-page`](tooling/summary-page/SKILL.md) | after a medium/large task, or "show as html" | Renders the end-of-task summary as a self-contained local HTML page instead of a wall of chat text. |

## How to read a SKILL.md

Each skill is a folder containing one `SKILL.md`:

```
---
name: skill-name
description: one-line pitch + trigger phrases — this is what Claude matches against
---

(the actual instructions Claude follows once triggered)
```

The frontmatter's `description` is what Claude Code uses to decide when to load the skill; the body is the workflow itself. No other files or tooling are required — a skill is just this one file.
