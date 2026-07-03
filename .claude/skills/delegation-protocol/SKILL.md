---
name: delegation-protocol
description: Protocol for delegating implementation, investigation or review work to Codex or to local agents (fast-worker, deep-reasoner, diff-reviewer). Use it whenever work is about to be delegated, a review is needed before a merge, a Codex command fails or hits quota, or the user asks to implement a spec, review a branch, or split work between models - even if they never say "delegate".
---

# Delegation protocol

Delegation is only cheaper than main-thread work if (a) the noisy work happens in someone else's context and (b) the task spec is smaller than the work itself. This protocol enforces both.

## 1. Task spec template (mandatory for any writing delegation)

```
GOAL: <one sentence, observable outcome>
ALLOWED FILES: <explicit paths or globs - the writer may touch nothing else>
NON-GOALS: <what must NOT change; include "no refactors" by default>
ACCEPTANCE: <verifiable criteria, bullet list>
VALIDATION: <exact commands: test / lint / build - from CLAUDE.md project commands>
RISK PATH: <yes/no, per CLAUDE.md risk paths. If yes: adversarial review is mandatory>
EXPECTED DIFF SHAPE: <files x approximate size - lets any reviewer spot drift instantly>
```

If VALIDATION cannot be filled, the project commands in CLAUDE.md are missing: ask the human to fill them before delegating. A spec without validation commands is decorative.

## 2. Routing table

| Work | Route | Why |
|---|---|---|
| Mechanical, fully specified | fast-worker agent | cheapest capable writer, isolated context |
| Scoped implementation with an approved spec | Codex rescue (delegate the task to Codex / codex-rescue), background | separate quota pool + cross-family perspective |
| One hard scoped question, no writing | deep-reasoner agent | premium reasoning without premium main thread |
| Escalation: deep-reasoner insufficient, or one decision's stakes justify premium spend | premium-reasoner agent - requires human authorization; include PREMIUM-APPROVED in the prompt | top-tier one-shot reasoning, spend-gated; premium tiers draw shared limits at a multiple of default models - verify with /usage |
| Pre-merge review (every branch) | `/codex:review --base <base> --background` | cross-family, read-only, cheap on quota |
| Risk-path change review | `/codex:adversarial-review --base <base>` plus a focus instruction from CLAUDE.md risk paths | decorrelated failure modes where bugs are most expensive |
| Codex exhausted, absent, or code must stay local | diff-reviewer agent | keeps the second pass; weaker decorrelation - it will say so |

## 3. Execution rules

- One writer per file per unit of work. Never run two writing agents on overlapping paths; if parallel work is unavoidable, use separate branches or git worktrees.
- Background long Codex jobs; collect with `/codex:status` then `/codex:result`. While a writer runs, do not start another writer on the same paths.
- Review at branch level (`--base`), never per micro-commit: each review re-establishes diff context - a fixed overhead - so amortize it per feature.
- Do not delegate tasks smaller than their spec: if writing the spec takes longer than writing the diff, do the work on the main thread.

## 4. Handling results

- Compare the result against EXPECTED DIFF SHAPE first; drift is cheaper to catch by shape than by reading.
- If the result deviates from the spec: reject with a delta list and re-delegate. Do not silently fix deviations on the main thread - that hides drift.
- Before moving on, summarize the decision (1-3 lines) into the relevant spec/ADR file under docs/, and keep the main context clean.

## 5. Cost guards

- Never enable the plugin's automatic review gate by default: it registers a Stop hook that loops Claude and Codex against each other and drains both quota pools. Supervised, high-stakes sessions only.
- If a Codex command fails on auth or quota: fall back per the routing table, and tell the human which pool is exhausted and when its window resets.
- Never route mechanical work to the most expensive available model. Escalation is by difficulty, decided by the human, not by quota remaining.
- Never add PREMIUM-APPROVED to a delegation prompt on your own initiative. The token exists so that premium spend always passes through the human.
