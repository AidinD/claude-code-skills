---
name: skill-builder
description: Turn a repeated workflow into a new skill via a short interview (what it does, when it should trigger, where it lives, what tools it needs), then write the SKILL.md file. Use when the user says "/skill-builder", "make a skill for this", "turn this into a skill", or agrees that a repeated workflow should become one.
---

Before starting the interview, check whether the repo or user has a skill-authoring
style guide (commonly named `SKILL-AUTHORING.md`, often kept alongside personal
Claude config or at the repo root) and read it if present - it holds the quality
bar the finished skill has to meet, and any pre-ship checks. If no such file
exists, skip this step and proceed with general good-skill judgment (clear
trigger description, minimal scope, tested example prompt). The interview below
gathers the raw material; that file, when present, decides whether what you write
is good enough to keep.

Two related tools exist, so pick deliberately rather than always reaching for
this one: the bundled `skill-creator` covers the heavier loop (test prompts,
evals, tuning the description for trigger accuracy) and is the better choice
when the skill matters enough to measure. This one is the quick
interview-and-write path.

Guide the user through creating a new skill via dialogue. Ask one question at a time:

1. **Name** — What should the skill be called? (becomes the /slash-command)
2. **Purpose** — What should it do in one sentence?
3. **Trigger** — When should someone use it? What's the situation?
4. **Scope** — Global (`~/.claude/skills/`) or project-specific (`.claude/skills/`)?
5. **Tools needed** — Does it need web search, bash, file reads, agents?
6. **Steps** — Walk through the workflow step by step

Once you have enough information, draft the skill file and show it to the user for review before writing it.

## IMPORTANT: file layout

Skills MUST be a **folder containing `SKILL.md`** — flat `.md` files in the skills directory are NOT picked up as slash-commands on this setup. Write to:
- `~/.claude/skills/<name>/SKILL.md` for global skills
- `.claude/skills/<name>/SKILL.md` for project-local skills

Use this frontmatter format (note the required `name` field):
```
---
name: <name>
description: <one-line description used for triggering>
---
```

Newly added skills only appear under `/` after the app/session is restarted.

After writing, confirm where the file was saved and that a restart is needed before `/<name>` shows up.
