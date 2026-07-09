# Spec: provenance and spend disclosure in delegated reports

Status: approved 2026-07-09. Revised the same day after independent review (APPROVE with concerns): both READMEs now label all three honesty tiers by name, carry the "CLI default (unknown)" fallback and point Codex spend at the OpenAI-side usage page; CLAUDE.md attributes the usage-closing action to the orchestrator explicitly. Companion to [gpt-5-6-sol-routing.md](gpt-5-6-sol-routing.md); ships in the same change-set.

## Context

The protocol now selects reviewers by authorship, but reports do not say which model actually did the work, at what effort, or what it cost. The human wants every delegated report to carry that: "written by Fable at x effort", plus token spend, plus API credits if API billing was used instead of weekly plan usage.

Facts verified on 2026-07-09 before designing this:

- **Claude Code subagents see their resolved model.** A live probe of `fast-worker` found this statement in its runtime context: "You are powered by the model named Sonnet 5. The exact model ID is claude-sonnet-5." The harness injects the *resolved* model - so an agent quoting it also exposes, per invocation, the case where an organization allowlist silently skipped the frontmatter pin (previously detectable only via `/agents`).
- **Claude Code subagents do NOT see their effort.** The same probe found no effort statement in context. Effort is reportable only as configuration: pinned in frontmatter (`premium-reasoner`: `high`) or "inherited (not visible at runtime)".
- **The orchestrator receives harness-reported usage per delegation.** Every Agent tool result carries `subagent_tokens`, `tool_uses`, `duration_ms`.
- **The Codex plugin runtime echoes neither model, effort, nor token usage.** Verified in codex-companion 1.0.5: `model`/`effort` are passed to `turn/start` but appear in no rendered output (task result, review result, `/codex:status`); no usage/token/credit field exists anywhere in its output.
- **Which pool tokens drew from (weekly plan quota vs API credits) is not visible programmatically** on either side. The template already handles this class of fact with runtime disclosure (`/usage`), never hardcoded rates.

## Decisions

1. **Provenance line, mandatory, first line of every delegated report.** Standard form:
   `Ran as: <agent-name> on <model name and ID exactly as the runtime context states them>; effort: <"pinned <value>" | "inherited (not visible at runtime)">`
   For diff-reviewer it is `Reviewed by: ...` and also names the diff's author (from the prompt, or "unstated - assumed Claude-family"), completing the independence trail in one line.
2. **Three honesty tiers, never conflated:**
   - *verified* - the model quoted from the subagent's own runtime context;
   - *requested* - Codex rescue's `--model`/`--effort` invocation flags (the CLI validates them but never echoes the runtime model);
   - *configured* - `/codex:review`'s inherited `.codex/config.toml` values, or "CLI default (unknown)" when the file is absent.
   The orchestrator composes the Codex lines, labeled with their tier.
3. **Fail-safe against self-belief.** Agents quote the context statement verbatim or write "model not reported by harness". Never answer from training-data identity.
4. **Spend reporting is an orchestrator duty** (agents cannot see their own token count): quote the harness-reported usage per delegation (`subagent_tokens`, tool uses, duration) and a running total per unit of work. Codex-side usage: state that the plugin runtime does not report it and point at the OpenAI-side usage page.
5. **Never convert tokens to money.** Rates are banned from this repo (they rot). Pool attribution (weekly plan quota vs API credits) stays a human runtime check via `/usage` - the same pattern as the premium-reasoner cost notice.

## Invariants

- All existing gates unchanged (`PREMIUM-APPROVED`, manual review gate, authorship-aware reviewer selection).
- No hardcoded rates, caps or dates; disclosure over computation.
- Agents stay project-agnostic; mechanics live in the skill; `CLAUDE.md` stays short.
- English canonical; `README.es.md` mirrors; plain declarative Markdown only.

## Changes by file

1. **`.claude/agents/fast-worker.md`** - report format gains the provenance line as item 1 (existing items renumbered), with the quote-or-"not reported" rule.
2. **`.claude/agents/deep-reasoner.md`** - output format gains the same provenance line as item 1 (existing items renumbered).
3. **`.claude/agents/premium-reasoner.md`** - Gate step 2 additionally requires the provenance line immediately after the cost notice; effort is reported as pinned (with a note to update the line if the frontmatter pin changes, same rule as the model-pin note).
4. **`.claude/agents/diff-reviewer.md`** - verdict format gains PROVENANCE as its first item: `Reviewed by: ... ; diff authored by: ...`.
5. **`.claude/skills/delegation-protocol/SKILL.md`** - §4 gains two bullets: the provenance line with its three tiers (verified / requested / configured) and orchestrator composition for Codex; the spend rule (harness usage per delegation + running total, Codex usage not reported, no money conversion, `/usage` for pool attribution).
6. **`CLAUDE.md`** - Delegation section gains one compact bullet: reports open with provenance; the orchestrator (never the agent, which cannot see its own token count) closes each result with harness-reported token usage; tiers and Codex caveats in the skill; no money figures; pools via `/usage`.
7. **`README.md`** - "The spend gate" gains a short "provenance and spend in every report" paragraph naming all three tiers (*verified* / *requested* / *configured*, with the "CLI default (unknown)" fallback) and pointing Codex-side spend at the OpenAI usage page; "First run" step 3 becomes three behaviors (spec first, report not silent diff, provenance line naming the resolved model).
8. **`README.es.md`** - mirrors both README changes.
9. **`docs/example-workflow.md`** - Step 3's expected report shape gains the provenance and usage items; Step 4 notes the verdict's reviewed-by/authored-by line.
10. **`CHANGELOG.md`** - one `[Unreleased]` Added entry.
11. **`llms.txt`** - the "Spend gates" concept line mentions the provenance line and harness-reported usage.

Out of scope: any attempt to compute costs, detect billing pool, or add tooling/scripts; the Codex plugin's own behavior.

## Validation

- `git status --short`: same tracked file set as the Sol change plus no new tracked files; `docs/specs/` gains this file (untracked).
- `git diff --check` clean; SKILL.md/README tables and fences intact.
- Grep: `"Ran as:"` appears in fast-worker, deep-reasoner, premium-reasoner; `"Reviewed by:"` in diff-reviewer; `"model not reported by harness"` in all four; no rate/price figures introduced (`git grep -nE '\$[0-9]'` clean).
- EN/ES parity on the touched sections.

## Expected diff shape (delta over the current working tree)

| File | Shape |
|---|---|
| 4 agent files | ~4-7 lines each |
| SKILL.md | ~6-10 lines |
| CLAUDE.md | ~2 lines |
| README.md / README.es.md | ~10-14 lines each |
| docs/example-workflow.md | ~4-6 lines |
| CHANGELOG.md | ~5-7 lines |
| llms.txt | ~1-2 lines |

## Open items (human)

- Same as the Sol spec: commit, optional cross-family pass, release/tag call.
- If a future plugin version starts echoing model/effort/usage, upgrade the Codex tiers from requested/configured to verified and update both specs' verified-facts sections.
