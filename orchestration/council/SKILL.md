---
name: council
description: Run an adversarial council of AI advocates on an open question - opposed positions, a prosecutor who attacks all of them, and a judge who must pick one answer and preserve the strongest objection to its own choice. Use when the user says "/council", "let two AIs debate this", "council on X", or asks for a hard multi-perspective decision on a direction, design, feature, or research question where a single answer would be too easy to agree with.
---

# Council

An adversarial multi-agent pass on an open question. Its purpose is to reach a
decision that is harder to agree with than a single answer would have been.

**Not for:** implementation work, bug fixing, or anything with one correct
answer. Use it for direction, design, scope, "should we build this", "which
approach", or research where the failure mode is premature agreement.

## The one thing that makes this work

Two instances of the same model, given the same context and told to "discuss",
**agree**. You get politeness and mutual validation, not criticism. So the
adversarialism must be structural, never requested:

- **Opposed roles with stakes.** Each advocate's job is to WIN, not to be
  balanced. One advocate's job is always to kill the idea entirely.
- **A judge that scores, not a consensus.** Consensus is where the quality dies.
- **Forced self-criticism.** Each advocate writes the strongest case against its
  OWN position, and is told a weak self-critique makes its whole case less
  credible to the judge. That incentive is what makes it honest.
- **Concessions required.** A prosecution that concedes nothing reads as
  motivated and gets discounted.
- **Dissent preserved in the output.** Never smooth the counter-arguments away -
  the reader needs them in order to disagree with the verdict.

Balance is the judge's job. Nobody else's.

## Shape

Default (lean, 5 agents): **3 advocates + 1 prosecutor + 1 judge.**

Do NOT build an N-squared cross-examination matrix. An earlier 10-agent version
(3 advocates + 6 pairwise rebuttals + judge) died on the session quota with zero
output. Quota is the binding constraint here, not code. A single dedicated
prosecutor attacking every position gives the same adversarial pressure at a
fraction of the cost.

Scale down to **2 advocates + 1 prosecutor + 1 judge** for a quick pass, or when
the question is genuinely binary.

Use `Workflow` for this (the skill's instruction is the opt-in). Phases:
`Positions` -> `Prosecute` -> `Judge`.

## Where this can run (a worked example: an orchestrator's tool-restriction policy)

Run a council from a seat that is ALLOWED to fan out - e.g. the top-level
coordinating session, not a restricted worker seat.

Do **not** wire this into a sub-agent persona whose configuration explicitly
restricts which tools it may use. In one internal multi-agent orchestrator
tool, worker/coordinator sub-personas are deliberately blocked from spawning
further sub-agents - their tool-restriction config denies the
"spawn-a-sub-task" capability - because a coordinator seat is meant to dispatch
work to existing workers, not spawn its own on top of them. That guard was
added after an earlier runaway where a coordinator seat fanned out
uncontrollably. A council needs to fan out to run at all, so putting it inside
a seat like that either silently cannot work or defeats the guard. A "Council"
persona was in fact built into that restricted seat and removed for exactly
this reason.

If a council should ever be startable from such a restricted seat, the right
shape is to convene it through that orchestrator's own dispatch /
report-collection primitives (its tiered task-delegation machinery) rather
than through raw fan-out - a decision worth recording wherever that project
tracks its own architectural decisions.

## Ground the advocates in facts first

The strongest passes argue about the DECISION, not about what is true. Before
spawning anything:

1. Gather the verified facts into the shared context block, and label what is
   VERIFIED vs ASSUMED vs UNKNOWN.
2. Where the question touches a repo, let the agents read the actual code. The
   most valuable findings come from file:line reality, not reasoning - e.g. a
   council once killed the front-runner option by finding that the scheduler
   lived inside `app.whenReady()`, so the whole approach was mechanically
   impossible.

If the facts are not gathered, the council will argue from vibes and produce
confident nonsense.

## Roles

**Advocates** (different models - genuinely different failure modes; e.g. two
Opus and one Sonnet). Each gets its own brief and returns structured output:

- `position`, `case_for` (3-5 concrete arguments)
- `strongest_case_against_me` - "genuinely damaging, not a token caveat"
- `hidden_costs` - the costs the others will ignore
- `what_would_change_my_mind`

One advocate is always the killer: *"Your JOB is to kill this. Be ruthless about
opportunity cost - what does NOT get built because this did? Attack the premise,
not just the options."*

**Prosecutor** (advocates nothing, high effort). Attacks every position: what is
actually wrong, costs they ignored, assumptions smuggled in, how the plan fails
in month three rather than week one. Must also return, per target, genuine
`concessions` and a verdict of `survives` / `wounded` / `dead`, plus
`what_all_of_them_missed` - which is often where the real answer comes from.

**Judge** (high effort). Scores every option on axes chosen for the question -
typically: does it deliver the stated want, cost to build and maintain, what is
lost, and what is UNKNOWN rather than merely risky. Rules given to it:

- Name ONE recommendation. Hedging is a failure.
- Reproduce the strongest surviving objection to your own recommendation.
- If the honest answer is "do nothing yet", say so - the do-nothing advocate is
  not a strawman.
- Separate VERIFIED from assumed; list what must be checked before acting.
- Give the smallest next step actually worth doing.
- It may reject the entire option set and name a better one. That is a success,
  not a failure to follow instructions.

## Reporting back

Lead with the recommendation and the single fact that decided it. Then:

- What overturned a previously held view (including the user's, or your own).
- The judge's objection to its own recommendation, not softened.
- Anything the pass found that is actionable independently of the decision -
  verified bugs, unsupported claims, cheap cleanups. Spot-check these yourself
  before relaying them as fact; the council's file:line claims are usually right
  but are not verification.
- The question only the user can answer, if there is one.
- The cost. A pass is roughly 300-400k subagent tokens and 10-15 minutes. Say so,
  so the user can judge whether it was worth it.

Then record the verdict where the decision lives (the relevant ticket and/or
`DECISIONS.md`), so it is not lost in the transcript.
