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
| Pre-merge review (every behavioral branch) | `/codex:review --base <base> --background` | cross-family, read-only, cheap on quota |
| Risk-path change review | `/codex:adversarial-review --base <base>` plus a focus instruction from CLAUDE.md risk paths | decorrelated failure modes where bugs are most expensive |
| Codex exhausted, absent, or code must stay local | diff-reviewer agent | keeps the second pass; weaker decorrelation - it will say so |

## 3. Execution rules

- One writer per file per unit of work. Never run two writing agents on overlapping paths; if parallel work is unavoidable, use separate branches or git worktrees.
- No agent reviews its own authorship. Author↔reviewer independence is the non-negotiable axis (cross-family is a bonus on top, not a substitute): the reviewer of a diff must be a different agent than who wrote it. A fresh critique pass by the *same* agent still catches lapses, so it is not worthless - but it shares the author's *systematic* blind spots (it writes the bug and waves it through for the same reason), so it does NOT discharge the review requirement. When authorship is split, review each portion by an agent that did not write it: Codex-authored code -> diff-reviewer (Sonnet) or human; Claude-authored code -> Codex adversarial.
- Review terminates by convergence, not by exhaustion. Every review produces fixes, and those fixes are new authorship nobody has reviewed - so independence recurses (diff -> review -> fixes -> review of the fixes -> ...). Read literally that never ends; two things make it converge. First, each fix-round review is *narrow*: the delta plus its dependency cone - what the fix touches and what depends on it - not the whole diff again, and not just the changed lines (a locally-correct fix can still break an already-approved caller, so the cone is the scope, not the line count). The reviewer verifies only that (a) the fixes resolve the findings they claimed to, and (b) they introduce nothing new, especially on risk paths. Second, the loop has a stop condition: halt when an independent pass over the *last authored delta* yields no new actionable findings - then the most recent authorship carries a green independent pass and you merge. It normally converges in 1-2 rounds because fix-rounds shrink - but shrinkage is why it is cheap, not why it terminates. What guarantees termination is the stop condition plus a round cap: if rounds ping-pong or grow instead of shrinking, that is diagnostic, not a tax - the change is ill-posed or the spec is wrong - so escalate to a human rather than loop.
- The independent pass is mandatory only where a defect can act without a human in the loop: code, executable config, hooks, migrations, anything on a risk path or with runtime/behavioral surface. There a bug crashes, corrupts, or leaks before anyone reads it, so the author's self-check never discharges the review. Pure prose with no runtime surface - docs, comments, formatting - sits below the line: any defect is mediated by whoever reads it, so the author may self-check and merge. This is what "behavioral" means in the §2 pre-merge row - every branch that changes behavior, not every commit. One caveat, because the threshold is "can the defect act unmediated", not "is it a .md": instruction docs (skills, CLAUDE.md, specs) shape behavior indirectly, so a wrong rule is a real defect - just a slow, human-mediated one - and when that guidance is high-stakes, take the independent pass voluntarily even though it is not mandated.
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
