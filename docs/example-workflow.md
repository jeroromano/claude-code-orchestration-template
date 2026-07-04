# Example workflow: one task through the protocol

This document walks one realistic task end to end through this template's delegation protocol: spec, routing, execution, independent review, a fix round, convergence, and human merge. It documents *this repository* - the template itself - and is not part of the payload a user copies into their own repo. That payload is `CLAUDE.md` plus `.claude/`; this file stays behind as reference.

## Setup

Assumptions for this walkthrough:

- The template is installed in a hypothetical TypeScript repo.
- The **Project commands** block in `CLAUDE.md` is filled: test is `npm test`, lint is `npx eslint .`.
- `/agents` lists the four workers: `fast-worker`, `deep-reasoner`, `premium-reasoner`, `diff-reviewer`.
- The Codex plugin is optionally installed; both branches are shown below where it matters.

## The task

The human asks: "Add unit tests for `slugify()` in `src/utils/slug.ts`: unicode input, empty string, repeated separators."

This routes to `fast-worker`: it is mechanical, fully specified (three named cases, one target function, one file to touch), and needs no design judgment. Contrast with a prompt like "rewrite `slugify()` to be Unicode-normalizing" - that asks for a behavior decision, not a test, and belongs on the main thread or with `deep-reasoner`, not with `fast-worker`.

## Step 1 - the spec

```
GOAL: Add unit tests for slugify() in src/utils/slug.ts covering unicode input, empty string, and repeated separators.
ALLOWED FILES: src/utils/slug.test.ts only.
NON-GOALS: No changes to slug.ts. No refactors of the test file's neighbors. No new dependencies.
ACCEPTANCE:
- A test file exists at src/utils/slug.test.ts.
- It covers: unicode input, an empty-string input, and repeated separators collapsing to one.
- All new tests pass under the project's test runner.
VALIDATION: npm test; npx eslint .
RISK PATH: no.
EXPECTED DIFF SHAPE: one new file, src/utils/slug.test.ts, roughly 40-60 lines.
```

If VALIDATION cannot be filled, stop and ask the human to fill the **Project commands** block in `CLAUDE.md` first - a spec without validation commands is decorative.

## Step 2 - routing

The first row of the routing table in [SKILL.md](../.claude/skills/delegation-protocol/SKILL.md) fires: "Mechanical, fully specified -> fast-worker agent." The task is scoped to one new file with named acceptance criteria and requires no architectural choice.

For contrast, the same task would route differently under different conditions:

- **Codex rescue** - if this were a broader, already-approved implementation spec (not just tests) and the Codex plugin were installed, it would run there in the background instead.
- **deep-reasoner** - if the ask were one hard scoped question with no writing (for example, "why does `slugify()` mis-handle combining diacritics" as investigation, not tests), it would go to deep-reasoner.
- **premium-reasoner** - only if the human authorizes premium spend and the orchestrator adds `PREMIUM-APPROVED` to the prompt for that specific invocation; the orchestrator never adds that token on its own initiative.

## Step 3 - execution

`fast-worker` returns a report, not a silently-applied diff. The exact wording varies by run, so treat the following as the expected shape, not a promised rendering:

- Changed files: the touched paths, which should be exactly `src/utils/slug.test.ts`.
- Validation results: pass/fail for each command in VALIDATION.
- Deviations from spec: normally "none".
- Out-of-scope observations: anything noticed but not touched - for example, "`slug.ts` has no JSDoc; unrelated to this task, not changed."

The orchestrator checks the diff against EXPECTED DIFF SHAPE first - one file, roughly 40-60 lines - because drift is cheaper to catch by shape than by reading the whole diff.

A deviation might look like: the worker also touched `slug.ts` to add a null check "while it was in there." That is outside ALLOWED FILES, even if well-intentioned, and the extra file alone breaks the expected shape before anyone reads a line. The orchestrator rejects the delta with a note of what changed and re-delegates with the scope restated - it does not quietly keep the extra fix, because that would hide drift from whoever reviews next.

## Step 4 - the review gate

Tests execute - a wrong assertion green-lights a bug with nobody in the loop - which is exactly the criterion SKILL.md §3 sets for the mandatory independent pass: a defect that can act unmediated. So this diff gets reviewed, and the reviewer must not be the author - `fast-worker` wrote the tests, so someone else has to check them.

Route: `/codex:review --base main --background` if the Codex plugin is installed; otherwise the `diff-reviewer` agent. Without the plugin, the reviewer's verdict will honestly flag that same-family review is the weaker guarantee - it says so rather than pretending otherwise.

This task's spec carries `RISK PATH: no`, so only this pre-merge pass applies. A change on a risk path - auth, payments, migrations, anything reading or writing user data, per `CLAUDE.md`'s Risk paths section - would additionally get an adversarial pass with the project-specific focus text from that section.

A finding might look like:

> CONCERN - the unicode test asserts what the current implementation produces rather than what the intended behavior is; if `slugify()` mishandles unicode today, the test would enshrine the bug.

## Step 5 - fix round and convergence

The fix is re-delegated narrowly: just the unicode test, not the whole file. The re-review covers only the fix plus its dependency cone - what the fix touches and what depends on it - not the whole diff again. Here that cone is small: the rewritten assertion and whatever the test file imports to build its expectation, so the second pass stays short.

The loop stops when an independent pass over the last authored delta yields no new actionable findings; then the human merges. That is usually round two: round one produces the CONCERN above, the narrow fix addresses it, and the round-two review finds nothing new to raise.

If rounds instead ping-pong or grow rather than shrink, that is diagnostic of an ill-posed change or a wrong spec - escalate to the human, never keep looping.

## When not to delegate

A one-line change whose spec would be longer than its diff goes on the main thread instead of through this protocol. Fixing a typo in a comment above `slugify()`, for instance, does not need a GOAL/ALLOWED FILES/ACCEPTANCE spec - writing that spec would take longer than fixing the typo.

Two cost guards worth restating here: never route mechanical work to the most expensive available model, and never add `PREMIUM-APPROVED` to a prompt on the orchestrator's own initiative.

## What this cost

Honest accounting: this flow used more tokens than writing three unit tests inline would have. Writing the three tests directly is one edit; this walkthrough took a spec, a worker turn, a reviewer turn, and a fix turn - four round trips instead of one. What it bought instead: context isolation (the main session never absorbed the worker's or reviewer's noise), an independent review before merge, and pool arbitrage if Codex handled the review (a separate quota pool from Claude's).

See the "What this is not" discussion in [README.md](../README.md) and the full mechanics in [SKILL.md](../.claude/skills/delegation-protocol/SKILL.md).
