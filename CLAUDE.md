# <Project> - Orchestration Policy

## Model routing
- Default main model: Opus or Sonnet. Never use the most expensive available model for mechanical work.
- Escalating the main model (e.g. to Fable) is a human decision (`/model`), reserved for architecture, hard multi-module bugs, and design sessions. Suggest escalation when warranted; never assume it.
- Design sessions must end with a written spec in `docs/specs/` before any implementation starts.
- Architecture, planning and specs are authored on the main Claude thread (Fable when escalated); never delegate them to Codex.
- Never use Codex Ultra or any external multi-agent orchestration mode: this policy is the orchestration layer - Ultra would duplicate it.

## Delegation
Mechanics live in the `delegation-protocol` skill - use it for any delegation or review. Summary:
- Mechanical, fully-specified work -> fast-worker agent.
- Scoped implementation with an approved spec -> Codex rescue if the Codex plugin is installed (Sol where the account has it - model-unavailable fallback in the skill §5; effort medium; high only for hard bugs or multi-module work), background; otherwise fast-worker. Exact command in the skill.
- Sol's primary role is independent audit of Claude-authored diffs; writing is secondary and always spec-bound.
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
- Sol review effort: high normally, xhigh on risk paths - set in `.codex/config.toml`, since `/codex:review` accepts no per-invocation model/effort flags; `max` only as an exceptional, human-authorized escalation when the invocation path exposes it (the current plugin path does not).
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
