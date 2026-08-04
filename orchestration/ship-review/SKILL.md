---
name: ship-review
description: Run an adversarial pre-ship review pipeline on the current repo's uncommitted changes (or most recent commits since upstream) - independent fresh-context review agent that also decides how critical the change is and which commands would catch a regression, required E2E verification evidence, auto-fix small issues vs. escalate ambiguous ones, ending with a recommendation instead of an auto-push. Use when the user says "review this before I ship it", "/ship-review", "run no-mistakes on this", or asks for an adversarial second-opinion pass on a change before committing/pushing.
---

# Ship review

Formalizes the manual pattern: implement -> independent adversarial review in
a FRESH context -> real E2E verification -> fix-or-escalate -> stop short of
pushing. Inspired by Kun Chen's "no-mistakes" (commit -> rebase -> peer-review
agent -> forced E2E test with evidence -> auto-fix/escalate -> lint -> push ->
open PR -> babysit CI). This skill covers steps 1-6 of that list (review
through fix-or-escalate) and deliberately stops before push/PR - see
"Why it stops before push" below.

This is a first-pass prototype, not a hardened spec. Treat the steps below as
a strong default, not a rigid script - use judgment where the repo or change
doesn't fit the mold.

## Where this can run

It dispatches subagents, so it needs the `Task`/`Agent` tool. In one orchestrator app
I maintain, a restricted-tools sub-agent persona has that tool explicitly denied, so
this skill cannot run from that seat - do not point a tool-restricted sub-agent at it.
Run it from a normal session, or have the restricted persona ask for one.

If the tool is unavailable, say so and stop. Reviewing the diff yourself in the same
context is not a degraded version of this skill; it is the exact thing it exists to
prevent, and every real defect found by using it came from a reviewer that had not
written the code.

## When there's nothing to review

Check `git status` and `git log @{upstream}..HEAD` (or `..main` if no
upstream). If both are empty, say so and stop - don't invent work.

## Step 1 - Scope the change

Identify what's under review:
- Uncommitted changes (staged + unstaged), if any exist. This is the common
  case - review before the user commits.
- Otherwise, commits on the current branch since it diverged from its
  upstream (or `main` if no upstream is set).

Run `git diff` (and `git diff --staged`) or `git log -p @{upstream}..HEAD` to
get the actual diff. Skim it yourself only enough to scope the review agent's
prompt (which files, what kind of change - UI, backend, config, docs-only) -
do not do the substantive review yourself. You (this session) most likely
wrote this code minutes ago; you are the wrong reviewer for it. That's the
whole point of dispatching a fresh agent in the next step.

## Step 2 - Independent review pass (fresh context, required)

Dispatch the review via the `Agent` tool - never review the diff yourself in
this same context. Prefer the `code-review` skill/agent if this environment
resolves it (check the current skill list for `code-review`; if present,
invoke it against the current diff instead of hand-rolling a prompt). If it
isn't available or doesn't fit (e.g. this isn't a GitHub PR, just a local
diff), fall back to the rubric below - it's adapted from this environment's
bundled `code-review` PR-review command's own approach:

Spawn a `general-purpose` (or `code-reviewer`-type, if available) agent with:
- The full diff (not just filenames - paste it or point at exact paths +
  line ranges).
- Explicit scope: correctness bugs, security issues, and regressions. Not a
  style pass (a separate `simplify` skill exists for reuse/cleanup - don't
  conflate the two).
- Instruction to report each issue with a confidence level (high / medium /
  low) and to explicitly exclude:
  - Pre-existing issues untouched by this diff.
  - Anything a linter/typechecker/CI would already catch.
  - Pedantic nitpicks a senior engineer wouldn't raise.
  - Intentional behavior changes that are clearly the point of the diff.
- Instruction to cite file path + line numbers for every finding, so you can
  verify without re-deriving it yourself.

If this session doesn't have a resolvable `code-review` skill/agent to
delegate to, say so explicitly in your final summary rather than silently
substituting - the user asked for this to be checked and reused, not
reinvented quietly.

### The reviewer also decides HOW MUCH checking this needed

Ask the same agent for two more things, in the same pass. Both are judgements the
author must not make about their own work:

1. **How much it costs to be wrong here** - one of:
   - `critical` - security, auth, data loss or corruption, money, or anything
     irreversible or outward-facing (releasing, publishing, deleting, spending).
   - `core` - state, persistence, or behaviour other work depends on.
   - `cosmetic` - visual or front-end only; a bug here is recoverable and finding it
     later is acceptable.

   Give the agent the diff and the touched paths and ask it to justify the level in
   one sentence. Do NOT tell it what level you had in mind - that is the whole point.

2. **Which commands would actually catch a regression here**, named as commands that
   can be run. Ask for these BEFORE showing it your own test suite, then compare. A
   command it proposes that you don't have is the most useful output of the entire
   review; a command you have that it didn't think to ask for is worth a second look.

**Why this belongs to the reviewer.** An author who picks their own level always has
a reason the work is less risky than it looks, and an author who picks their own
checks picks the ones their code passes. Both were observed on 2026-07-27: a
`cosmetic` label required no evidence at all and rendered under "Ready to stamp",
and a declared check whose command was `... || exit 0` produced a genuine green that
meant nothing. Neither is caught by asking the author to be more careful.

Record both on the review record (`criticality`, `checks`, and
`independentReview: {by, summary, findings}`). If the reviewer's level is HIGHER than
yours, take theirs - the asymmetry is deliberate, because being wrong about
`critical` costs more than being wrong about `cosmetic`. If it is lower, keep yours
and say why in the record.

## Step 3 - Require real E2E verification evidence

Do not accept "looks right" or "should work" as sufficient - this mirrors a
standing engineering rule worth adopting broadly (Bug Fixes & Testing:
reproduce/verify in a setting as close as possible to how the change is
actually experienced).

Pick evidence appropriate to the change:
- **UI-facing change**: use `preview_start` / `preview_screenshot` /
  `preview_snapshot` (or the project's own `run` skill if one exists) to
  actually load the affected screen and observe it, or describe a manual
  click-through you performed. A screenshot or an explicit step-by-step
  description of what was clicked and what was observed - not "the code
  looks correct."
- **Backend/CLI/script change**: run the actual command (build, test suite,
  the specific function/endpoint exercised) and capture real output. Paste
  the command and its output, not a description of what you expect it to
  print.
- **Config/infra-only change**: show the diff being applied to a real
  target where feasible (e.g. `terraform plan`, `wrangler dry-run`,
  a lint/validate command) - if no such check exists, say explicitly that
  no automated verification was possible and this needs a manual check.

If genuine E2E verification isn't possible in this environment (no live
target, destructive action, requires credentials you don't have), say so
explicitly rather than skipping the step silently - an honest "couldn't
verify, here's why" is acceptable; a silently-skipped verification step is
not.

## Step 4 - Fix or escalate

For each issue from Step 2 (and anything you noticed during Step 3's
verification):
- **Fix directly** when the fix is small, unambiguous, and low-risk (e.g. an
  off-by-one, a missing null check, a clearly-wrong condition, a typo'd
  config key). Apply it, then re-run the relevant piece of Step 3's
  verification to confirm the fix actually resolves it - don't just apply
  and assume.
- **Escalate instead of fixing** when the issue involves a design tradeoff,
  touches code you don't have full context on, is security-sensitive enough
  that a wrong autonomous fix is worse than a flagged one, or the "right"
  fix isn't obvious. List these clearly for the user with the file/line and
  why you didn't just fix it - mirrors the same fix-vs-flag distinction that
  should apply to unrelated issues noticed while testing.

Re-dispatch a fresh review pass (Step 2) after applying direct fixes if the
fixes were non-trivial - don't let the same context that just wrote the fix
also be the last word on whether the fix is correct.

## Step 5 - Summarize, don't push

This skill never runs `git push`, opens a PR, or auto-commits on the user's
behalf. End with a plain summary:
- What was reviewed (diff scope: uncommitted / N commits since upstream).
- What the independent review pass found, and its disposition (fixed /
  escalated / dismissed as false positive, with reasoning for dismissals).
- What E2E evidence was gathered (or why it couldn't be).
- **The level the reviewer set, and whether it differs from yours.** If it went up,
  say so plainly - that is the single most useful line in the summary.
- **Any check the reviewer proposed that you did not have.** Whether you added it or
  not, and why.
- A clear recommendation: e.g. "looks safe to commit and push" or "N
  escalated issues need your judgment before this ships."

Write the summary for someone who is NOT going to read the diff - that is the whole
point of the exercise. No unexplained error codes, abbreviations or internal names:
say what would have gone wrong for them and what it means now. "A board update was
silently lost when another program held the file" beats "EPERM on the atomic rename".

Leave the actual `git commit` / `git push` / PR creation to the user, or to
being explicitly invoked as a separate next step (e.g. the existing `pr`
skill). This is a deliberate scope boundary, not an oversight - see below.

## Why it stops before push

Per the human-gating principle already established for other orchestration
work (scale autonomy to blast radius - propose, don't auto-act, for anything
that mutates shared/external state): pushing and opening a PR
are more consequential and harder to undo than a local fix-and-verify loop.
A skill that silently pushes on your behalf the first time it has a bug in
its own judgment does real damage; a skill that stops at "here's what I found,
you decide" does not. If tightening this later, add an explicit opt-in flag
rather than making push the default.

## The failure list - check for these BY NAME every time

Every entry below is a real miss from this pipeline's own history, and each one
passed a review that felt thorough. Read them as a checklist for the review agent's
prompt, not as background. Ask about each explicitly; a reviewer who is not told to
look for these will report that the tests look good. Grouped by failure shape, not
chronology - the grouping is the fast way to skim this on a repeat read.

### Assertions that don't test what they claim to

1. **A guard asserted by NAME, not by ARGUMENT.** A check that greps for
   `removeWorktree(` passes when the options object flips from a safe mode to a
   forceful one, and a check that a call site has no `force: true` says nothing about
   the default flag inside the function. Two such mutations survived a suite that read
   as exhaustive (2026-08-03). For every safety option, assert the literal argument.
2. **A source-scan check that matches a comment.** Strip comments before asserting, or
   commenting the guarded line out leaves every check green.
3. **An assertion that cannot fail.** A ternary that yields `true`, a `.some()` over an
   empty array, an `expect` on a value the test itself just wrote. Mutation-test the
   test, not just the code.
4. **A guard closing the LAST review's finding, with no test of its own.** Of 8 guards
   that survived a 23-mutation matrix, the pattern was not random: they were the ones
   written under review pressure, where the reflex is to fix and re-report rather than
   to break the fix and watch. Every finding you close gets a mutation, not an
   argument.

### Fixing the mechanism without checking the symptom

5. **The mechanism changed, the symptom did not.** Persisting a value is not restoring
   it; writing a field nothing reads back fixes nothing. State the finding as the
   USER-VISIBLE symptom and re-check THAT, not the line you edited.
6. **A test seam ignored because imports are hoisted.** Setting `process.env.X_PATH`
   above a static `import` does nothing: ESM imports run first, and a module that
   resolves its path at import time has already read the ambient value. Use a dynamic
   import after the env var, and afterwards CHECK that no real data file was touched.
7. **A condition that short-circuits on absent data.** `x.branch && !isManagedBranch(x.branch)`
   skips the guard entirely when `branch` is null. Ask what each guard does with null,
   empty string, and a shape that did not exist when it was written.
8. **A filter whose input the app can no longer produce.** A test that hand-writes the
   state it filters on proves nothing about the app. For every predicate on a field,
   grep for who WRITES that field and confirm a live code path still does - a reshape
   that moves work from one mechanism to another silently empties every filter written
   against the old one. This emptied the same widget twice, a day apart, through two
   different mechanisms (2026-08-03).

### Green in isolation, wrong in context

9. **One file, not the suite.** A test that passes standalone can fail under the runner
   - and the reason is often that standalone it wrote to real state instead of its temp
   fixture. Run the whole suite before claiming green.
10. **A placeholder that lies.** "No sweep has run yet" while one is pending reads
    identically to the truth. A UI test that polls until the text settles catches it; one
    that reads the DOM synchronously passes on the lie.
11. **A new check that makes itself pass.** A test naming its own expected findings as
    strings counted as their caller, so everything it was looking for looked healthy.
    Run every new check against a deliberately broken world BEFORE believing a green
    result from it.

### Blast radius and irreversible actions

12. **One instance fixed, the class left open.** The same wrong assumption is usually in
    two or three places: two of two call sites, both dashboards, every filter written
    against the old data shape. After fixing one, grep for the pattern and say out loud
    how many you found.
13. **A repo-global command used for a scoped decision.** `git worktree prune` deregisters
    every absent worktree, not the one you decided about. Check the blast radius of each
    command, not just its intent.
14. **A fix that removes an independent second opinion.** Replacing git's own refusal
    (`--force`, `-D`) with your own check means your check is now the only thing standing
    there. If you do it, probe your check's blind spots deliberately - that is where the
    next data-loss bug is (2026-08-03: it was, within one round).
15. **A report that contradicts itself about a destructive action.** Two lists built in
    different passes ("removed" from observation, "kept" from a plan) drifted until the
    same branch appeared in both, and the kept count was inflated by exactly the number
    deleted. Assert the invariant between the lists, not each list separately.

## Order matters: review BEFORE you write the tests

Dispatch the independent pass as soon as the change works, not after a suite exists.
Every finding in the 2026-08-03 rounds was reachable by the method the author already
had - build a temp fixture and run it - but the suite tested the cases the author had
thought of, which is the definition of the blind spot a reviewer exists to fill. A
suite written first also makes the author defend it.

## Open questions / tradeoffs (flagging for review, not resolved here)

- **Rebase-onto-main step**: no-mistakes rebases before reviewing. This skill
  doesn't do that automatically - rebasing is itself a mutating, sometimes
  conflict-prone action, and doing it unprompted inside a "just review this"
  skill felt like scope creep. Worth reconsidering if staleness-vs-main turns
  out to cause real review blind spots in practice.
- **Which agent type to dispatch in Step 2** - SETTLED 2026-07-27 by using it for
  real, four times in one day. `general-purpose` on `opus` with the rubric above
  works well, and the single highest-value instruction to add is "verify by RUNNING a
  probe script, not by reading". The pass that found the most (12 findings, one fatal)
  was the one told to copy the module to a temp tree, disable a guard, and check
  whether the test suite noticed - mutation testing. Ask for that explicitly on
  anything whose tests are the thing being trusted.
- **How much the reviewer should be told** - SETTLED: as little as possible about your
  own conclusions. Passes given the criticality level up front agreed with it; passes
  asked to decide it independently disagreed usefully. Same for checks: ask before
  showing your suite.
- **Lint/docs pass**: no-mistakes includes a lint/docs step before push. Not
  included here - assumed to be covered by the repo's own CI/pre-commit hooks
  rather than duplicated in this skill. Add it back in if a given repo has no
  such automation.
- **No iteration cap**: Step 4's "re-dispatch review after fixing" could loop
  if the reviewer keeps finding new issues. Use judgment to stop and escalate
  to the user after ~2 rounds rather than looping indefinitely.
