# <Project> - Orchestration Policy

## Model routing
- Default main model: Opus or Sonnet. Never use the most expensive available model for mechanical work.
- Escalating the main model (e.g. to Fable) is a human decision (`/model`), reserved for architecture, hard multi-module bugs, and design sessions. Suggest escalation when warranted; never assume it.
- Design sessions must end with a written spec in `docs/specs/` before any implementation starts.
- An APPROVED design or spec authored by the escalated top tier is AUTHORITATIVE for the cheaper model implementing it: implement to the letter, each point a checklist item - no silent simplification, no substituting a specified check with "equivalent" reasoning of your own. Flag any deviation to the human BEFORE deviating. Only overrides: an independent review catches a concrete error (repro or file:line), or an explicit human decision. Audit and review findings are NOT specs: they enter the finding-validation flow (skill §6) and are never auto-applied.
- Architecture, planning and specs are authored on the main Claude thread (Fable when escalated); never delegate them to Codex.
- Never use Codex Ultra or any external multi-agent orchestration mode - including the `ultra` reasoning effort, which is Ultra (automatic task delegation to sub-agents): this policy is the orchestration layer - Ultra would duplicate it.

## Delegation
Mechanics live in the `delegation-protocol` skill - use it for any delegation or review. Summary:
- Mechanical, fully-specified work -> fast-worker agent.
- Scoped implementation with an approved spec -> Codex rescue if the Codex plugin is installed (GPT-6.1 Sol - needs Codex CLI 0.159.0+ and account access; model unavailable -> fast-worker, never a rerun on the CLI default, per the skill §5; effort medium; high only for hard bugs or multi-module work), background; otherwise fast-worker. Exact command in the skill.
- Sol's primary role is independent audit of Claude-authored diffs; writing is secondary and always spec-bound. GPT-6 Astra - the costlier frontier tier - is reserved for the risk-path adversarial pass (unavailable -> GPT-6.1 Sol, disclosed); it never implements and is never the default.
- One hard scoped question -> deep-reasoner agent.
- Beyond deep-reasoner -> premium-reasoner agent, ONLY with explicit human authorization per invocation: the orchestrator adds the token PREMIUM-APPROVED to the delegation prompt if and only if the human authorized premium spend for that question in this session. Never auto-delegate to it.
- Every writing delegation carries the full task spec (goal, allowed files, non-goals, acceptance, validation, expected diff shape).
- Every delegated report opens with a provenance line (agent, model, effort - tiers and Codex caveats in the skill §4); the orchestrator closes each result with harness-reported token usage. No money figures; pool attribution is the human's check via /usage.
- One writer per file. Parallel writers only on separate branches/worktrees.

## Review gate (manual, branch-level)
- Every branch with runtime/behavioral surface (e.g. code, migrations, hooks) gets an independent review before merge. The reviewer is chosen by authorship - no agent approves its own diff:
  - Claude-authored -> `/codex:review --base main --background` if the Codex plugin is installed; otherwise the diff-reviewer agent. (FILL IN: replace `main` if this repo's default branch differs.)
  - Codex-authored -> the diff-reviewer agent or a human reviewer. Never Codex.
  - Mixed authorship -> each portion is reviewed by an agent that did not write it.
- Sol review effort: the highest the invocation path exposes - `max` via `direct` (the raw CLI pins it per invocation: `codex review --commit <sha> -c model_reasoning_effort="max"` - single-commit form, not a branch review; caveat in skill §2. The flagless `/codex:review` inherits the resolved Codex config), `xhigh` as the pinnable ceiling via `inline-task` (the companion's `--effort` flag rejects `max`; an omitted flag inherits the resolved config, which may sit higher). A pin is an exact selection and can LOWER effort: preflight the resolved config (unset counts as below - pin the floor; `ultra` is never inherited - pin the ceiling), pin only to raise a low one or to replace a banned `ultra`, and verify what actually ran - model, effort AND read-only profile - via the session rollout before acting on a verdict (mechanics: skill §5-§6). `max` is the highest ALLOWED effort (`ultra` sits above it and is banned); the risk-path adversarial pass runs at that same maximum, on GPT-6 Astra.
- Review transport: `auto` | `direct` | `inline-task` - this repo uses `auto`. `direct` is the two commands above; `inline-task` ships the locally-computed diff as a read-only background job on the plugin's companion task runtime instead - never via `/codex:rescue`, which returns no cancellable job ID (mechanics: delegation-protocol skill §6). On native Windows `auto` resolves to `inline-task`: the direct commands hang there - the Codex sandbox cannot spawn processes (verified on Windows 11, July 2026, plugin 1.0.6 / CLI 0.144.0; verified per-machine fix, 2026-08-08: PowerShell 7 installed outside WindowsApps, then restart the plugin's app-server, which caches the old PATH). Set `direct` once a probe review completes after the fix - it is the only path that can PIN `max` per invocation (inline-task reaches `max` only by inheriting it from the config), and the reviewer then verifies its hypotheses against the real repo instead of a pasted diff.
- Pure-doc changes with no runtime/behavioral surface may be self-merged; the `delegation-protocol` skill (§3) defines the threshold.
- Risk-path changes additionally get an adversarial pass, chosen by the same authorship rule: `/codex:adversarial-review` for Claude-authored diffs; for Codex-authored diffs, diff-reviewer (or a human) with the focus text below - never Codex.
- Never skip the second pass. Never enable the automatic review gate.

## Risk paths (FILL IN - replace with yours)
Changes here require: implementation by one agent, adversarial review by another, human approves the merge.
- Authentication & session handling
- Payments / billing
- Data migrations
- Anything that reads or writes user data
- <add project-specific ones>

Adversarial review focus text: "<one or two sentences telling the reviewer what breaking looks like in this project>"

## Data boundary (only if this is client / restricted code)
Codex runs on OpenAI infrastructure: delegated code ships there. If this repo has confidentiality constraints, default reviews to diff-reviewer and make Codex opt-in per task. Secrets, real-data fixtures and `.env*` never go into a delegated diff.

## Context hygiene
- Keep this file short: it is loaded every session. Long docs live in `docs/` and are read on demand.
- After each significant decision, write it into the relevant spec before continuing.

## Project commands (FILL IN - delegation specs are decorative without these)
- build: `<cmd>`
- test: `<cmd>`
- lint/format: `<cmd>`
- Stack notes (language, framework, key dirs): `<fill in 3-5 lines>`
