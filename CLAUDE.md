# <Project> - Orchestration Policy

## Model routing
- Default main model: Opus or Sonnet. Never use the most expensive available model for mechanical work.
- Escalating the main model (e.g. to Fable) is a human decision (`/model`), reserved for architecture, hard multi-module bugs, and design sessions. Suggest escalation when warranted; never assume it.
- Design sessions must end with a written spec in `docs/specs/` before any implementation starts.

## Delegation
Mechanics live in the `delegation-protocol` skill - use it for any delegation or review. Summary:
- Mechanical, fully-specified work -> fast-worker agent.
- Scoped implementation with an approved spec -> Codex rescue (if the Codex plugin is installed), background; otherwise fast-worker.
- One hard scoped question -> deep-reasoner agent.
- Beyond deep-reasoner -> premium-reasoner agent, ONLY with explicit human authorization per invocation: the orchestrator adds the token PREMIUM-APPROVED to the delegation prompt if and only if the human authorized premium spend for that question in this session. Never auto-delegate to it.
- Every writing delegation carries the full task spec (goal, allowed files, non-goals, acceptance, validation, expected diff shape).
- One writer per file. Parallel writers only on separate branches/worktrees.

## Review gate (manual, branch-level)
- Before merging a branch with runtime/behavioral surface (e.g. code, migrations, hooks): `/codex:review --base main --background` if the Codex plugin is installed; otherwise the diff-reviewer agent. Pure-doc changes with no runtime/behavioral surface may be self-merged; the `delegation-protocol` skill (§3) defines the threshold. (FILL IN: replace `main` if this repo's default branch differs.)
- Risk-path changes additionally get an adversarial pass (`/codex:adversarial-review`, or diff-reviewer with the focus text below).
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
