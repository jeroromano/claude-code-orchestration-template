# Spec: GPT-5.6 Sol routing

Status: approved 2026-07-09. Revised the same day after independent review (round 1): the authorship rule now explicitly covers the risk-path adversarial pass, and the plugin-presence gate stays explicit in CLAUDE.md's delegation bullet. Implements the human's routing decisions for GPT-5.6 Sol without breaking the template's clean degradation when the Codex plugin is absent.

Revised again 2026-07-09 after the post-release Sol audit of v0.2.0: the rescue pin gains a model-unavailable degradation (Sol is a limited-access preview - rerun without `--model` or fall back to fast-worker); decision 8's unstated-author default is replaced by fail-closed UNKNOWN (assuming Claude-family could route a Codex-authored diff back to Codex on a risk path); the `max` limitation is re-attributed from the Codex CLI to the per-invocation plugin path; the README no longer implies reviews run on Sol without the optional config pin, and notes the trusted-repo condition for a project-level Codex config.

## Context

The optional Codex integration now runs GPT-5.6 Sol. The template must state Sol's role, its effort ladder, and - critically - fix an existing ambiguity: `CLAUDE.md`'s review gate routed *every* behavioral branch to `/codex:review` regardless of who authored it, while the delegation-protocol skill (§3) already required Codex-authored code to be reviewed by diff-reviewer or a human. Read together, CLAUDE.md could send a Codex-authored branch to Codex for approval.

Interface facts, verified against openai/codex-plugin-cc 1.0.5 and Codex CLI 0.144.0 (July 2026):

- `/codex:rescue` accepts `[--background|--wait] [--resume|--fresh] [--model <model>] [--effort <none|minimal|low|medium|high|xhigh>]`. No `max` effort exists in this version.
- `/codex:review` accepts only `[--wait|--background] [--base <ref>] [--scope ...]` - no per-invocation model or effort flags. It inherits the Codex CLI configuration (`.codex/config.toml`).

## Decisions

1. **Roles.** Claude (Fable when escalated) owns architecture, planning and specs; these are never delegated to Codex. Sol's primary role is independent audit of Claude-authored diffs; its secondary role is implementing already-approved specs.
2. **Review effort ladder (Sol).** `high` for normal important reviews; `xhigh` for risk paths; `max` only as an exceptional, explicitly human-authorized escalation *when the invocation path exposes it* (the plugin's `--effort` parser and the documented TOML values stop at `xhigh` in the verified versions). Since `/codex:review` takes no flags, effort is set via `.codex/config.toml`.
3. **Writing effort ladder (Sol).** `medium` by default; `high` only for hard bugs or multi-module work. Canonical command:
   `/codex:rescue --fresh --background --model gpt-5.6-sol --effort medium <approved spec>`
4. **No Ultra.** Codex Ultra is multi-agent orchestration; this template is already the orchestration layer. Using it would duplicate the system and multiply spend on both pools.
5. **Reviewer chosen by authorship.** Claude-authored -> Codex review; Codex-authored -> diff-reviewer (Claude) or human; mixed -> each portion reviewed by an agent that did not write it. Codex never approves its own authorship. The rule extends to the risk-path adversarial pass: a Codex-authored risk-path diff takes its adversarial pass with diff-reviewer (or a human) using the CLAUDE.md focus text, never `/codex:adversarial-review`.
6. **Automatic review gate stays disabled.** Unchanged.
7. **No active `.codex/config.toml` in the template.** Documented as optional configuration only:
   ```toml
   model = "gpt-5.6-sol"
   model_reasoning_effort = "high"
   ```
8. **Author-aware diff-reviewer.** Codex-authored diffs become an explicit trigger; the same-family blind-spot warning becomes conditional on the diff's author. When authorship is not stated in the prompt, diff-reviewer treats it as UNKNOWN and fails closed: the same-family caveat fires AND no Codex routing is recommended until the author is stated (revised 2026-07-09 - the original "assume Claude-family" default could have routed a Codex-authored diff back to Codex on a risk path).

## Invariants (must survive the change)

- Clean degradation: every Sol reference sits inside an "if the Codex plugin is installed" branch; without the plugin the routing is exactly today's local routing.
- `PREMIUM-APPROVED` gate untouched; automatic review gate stays prohibited; `max` effort is human-gated per invocation, like premium spend.
- Author↔reviewer independence only strengthens.
- One source of truth: policy in `CLAUDE.md` (kept short), mechanics in the skill, agents project-agnostic. English canonical; `README.es.md` mirrors `README.md`.
- Plain declarative Markdown: no scripts, no hooks, no shipped `.codex/` directory. Provenance stamps are updated on re-verification, never removed.

## Changes by file

1. **`CLAUDE.md`**
   - Model routing: add two bullets - (a) architecture, planning and specs are authored on the main Claude thread (Fable when escalated) and never delegated to Codex; (b) never use Codex Ultra or any external multi-agent orchestration mode (this policy is the orchestration layer; Ultra would duplicate it).
   - Delegation: add a bullet stating Sol's primary role (independent audit of Claude-authored diffs; writing is secondary and spec-bound). Rework the "Scoped implementation" bullet to name Sol and its effort rule (medium; high only for hard bugs or multi-module work), pointing at the skill for the exact command and keeping the plugin-presence condition explicit ("if the Codex plugin is installed").
   - Review gate: make reviewer selection authorship-aware (Claude-authored -> `/codex:review --base main --background`, else diff-reviewer; Codex-authored -> diff-reviewer or human, never Codex; mixed -> each portion reviewed by a non-author). The risk-path adversarial bullet follows the same authorship rule: `/codex:adversarial-review` only for Claude-authored diffs; diff-reviewer (or a human) with the focus text for Codex-authored ones. State the Sol review effort ladder and that effort is set via `.codex/config.toml` because `/codex:review` takes no per-invocation flags. Keep: the pure-doc self-merge threshold deferred to skill §3, the `main` FILL-IN note, "Never skip the second pass. Never enable the automatic review gate."
2. **`.claude/skills/delegation-protocol/SKILL.md`**
   - §2 routing table: "Scoped implementation" row carries the canonical rescue command and the medium->high escalation rule, plus "Sol implements specs; it never authors them". Split the pre-merge review row into Claude-authored (`/codex:review`, note it takes no model/effort flags and inherits `.codex/config.toml`, suggested pin gpt-5.6-sol at high - see README) and Codex-authored (diff-reviewer or human - never Codex; no agent approves its own authorship). Risk-path row: split by authorship - Claude-authored keeps `/codex:adversarial-review` with `model_reasoning_effort = "xhigh"` via `.codex/config.toml` for that review (`max` only as an exceptional human-authorized escalation when the CLI supports it); Codex-authored routes to diff-reviewer (or a human) with the CLAUDE.md focus text, never Codex. Fallback row: add "or the diff is Codex-authored" to the conditions.
   - §3 independence bullet: add that every review delegation prompt must state the diff's author (the reviewer's blind-spot disclosure depends on it; diff-reviewer assumes Claude authorship when unstated).
   - §5 cost guards: add (a) never use Codex Ultra or any multi-agent Codex mode - this protocol is already the orchestration layer, Ultra duplicates it and multiplies spend on both pools; (b) effort follows the routing table - Sol reviews at high (xhigh on risk paths) and writes at medium (high only for hard bugs or multi-module work); `max` is never a default: exceptional, human-authorized per invocation, and only when the CLI supports it.
3. **`.claude/agents/diff-reviewer.md`**
   - `description`: add the Codex-authored trigger first: use before any merge whenever the diff was authored by Codex (Codex never reviews its own work), plus the existing triggers.
   - Procedure step 1: also take the diff's author from the prompt; if unstated, assume Claude-family authorship and say so in the verdict.
   - Replace "Known limitation" with an authorship-conditional section: Claude-authored -> same-family warning as today plus the `/codex:adversarial-review` recommendation for risk paths; Codex-authored -> you ARE the cross-family pass, no same-family caveat, and never route the diff back to Codex; mixed -> apply the caveat per portion and flag any portion whose author would also be its reviewer.
4. **`README.md`**
   - New subsection under "Optional: OpenAI Codex plugin" titled around model/effort for GPT-5.6 Sol: Sol's two roles (auditor of Claude-authored diffs first, spec-bound implementer second, never the architect); the reviewer-by-authorship rule; both effort ladders; the canonical rescue command; the note that `/codex:review` has no per-invocation model/effort flags and inherits `.codex/config.toml`, with the optional two-line TOML snippet from Decision 7 (explicitly not shipped by the template; raise to `"xhigh"` for a risk-path review); and the "do not use Ultra" rule with its one-line rationale.
   - "First run" step 4: make the reviewer choice authorship-aware.
   - Interface-coupling stamp: refresh to Codex CLI 0.144.x (re-verified July 2026).
5. **`README.es.md`** - mirror all README.md changes in Spanish (rioplatense voseo, existing voice). Keep the translation-sync stamp at July 2026.
6. **`docs/example-workflow.md`**
   - Step 2, "Codex rescue" contrast bullet: name the concrete command with Sol at effort medium.
   - Step 4: add the authorship note - fast-worker (Claude family) authored, so Codex review is the cross-family independent pass; had Codex rescue authored the diff, review would route to diff-reviewer or a human instead, because Codex never reviews its own authorship.
7. **`CHANGELOG.md`** - under `[Unreleased]`: Added (GPT-5.6 Sol routing guidance: auditor role with high/xhigh/max-gated ladder, spec-bound writer via the rescue command at medium/high, optional `.codex/config.toml` documentation, no-Ultra rule; this spec file) and Changed (reviewer selection now authorship-aware across CLAUDE.md, the skill and diff-reviewer, resolving the CLAUDE.md<->skill ambiguity about who reviews Codex-authored code; diff-reviewer gains the Codex-authored trigger and its same-family warning is now conditional on the author). No version bump.
8. **`llms.txt`** - diff-reviewer line: add that it is the designated reviewer for Codex-authored diffs (not only the fallback). Codex-integration concept line: reviews are cross-family for Claude-authored diffs (GPT-5.6 Sol); Codex-authored diffs route to the local diff-reviewer instead.

Out of scope: `AGENTS.md`, `SECURITY.md`, any `.codex/` file, version bump/tag, any behavior of the premium-reasoner gate.

## Validation

- `git status --short` shows exactly the nine files above (eight modified + this spec).
- `git diff --check` clean.
- `git diff --stat` matches the expected shape below.
- Every relative Markdown link added resolves to an existing file.
- `git grep -n "gpt-5.6-sol"` returns consistent spelling everywhere it appears.
- No remaining instruction routes reviews author-blind to `/codex:review`.
- EN/ES parity on every touched section; `.codex/` absent from the tree.

## Expected diff shape

| File | Shape |
|---|---|
| `docs/specs/gpt-5-6-sol-routing.md` | new, ~100-140 lines |
| `CLAUDE.md` | ~15 lines touched |
| `.claude/skills/delegation-protocol/SKILL.md` | ~20 lines touched |
| `.claude/agents/diff-reviewer.md` | ~15 lines touched |
| `README.md` | ~30 lines touched |
| `README.es.md` | ~30 lines touched |
| `docs/example-workflow.md` | ~10 lines touched |
| `CHANGELOG.md` | ~12 lines added |
| `llms.txt` | ~3 lines touched |

## Open items (human)

- Optional cross-family pass (`/codex:review`) on this change before commit - it is instruction-doc guidance, below the mandatory line but high-stakes.
- Release/tag decision (entries sit under `[Unreleased]`).
- Confirm the `gpt-5.6-sol` model slug against your Codex account when first invoking it.
