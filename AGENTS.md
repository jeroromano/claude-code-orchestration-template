# AGENTS.md

Orientation for AI coding agents (OpenAI Codex, Cursor, and similar) working **on this repository**. It does **not** replace [`CLAUDE.md`](CLAUDE.md); that file is the operating policy and wins on any conflict. This file just helps you get oriented fast.

## What this repository is

A **documentation template** for orchestrating [Claude Code](https://code.claude.com). The deliverable is a set of instruction files - a policy (`CLAUDE.md`), four subagents in `.claude/agents/`, and one skill in `.claude/skills/` - that a developer copies into their own repo. There is no application code, no build step, and no runtime. The `Project commands` placeholders in `CLAUDE.md` exist for downstream users to fill in *their* repositories; this template itself has nothing to build or test.

## Map

| Path | Purpose |
|---|---|
| `README.md` / `README.es.md` | Human-facing docs. English is canonical; Spanish mirrors it. |
| `CLAUDE.md` | The orchestration policy - behavioral source of truth. |
| `.claude/agents/*.md` | The four subagents: `fast-worker`, `deep-reasoner`, `premium-reasoner`, `diff-reviewer`. |
| `.claude/skills/delegation-protocol/SKILL.md` | Delegation mechanics: task-spec template, routing table, review rules. |
| `docs/example-workflow.md` | Worked end-to-end example of the delegation protocol. Documentation only - not part of the install payload. |
| `llms.txt` | Compact machine-readable map of the project. |

## Working rules

- **Keep it declarative.** No scripts, dependencies, package managers, binaries, hooks, CI, or telemetry. Everything here is plain Markdown by design.
- **One source of truth.** Project-specific content (risk paths, validation commands, data boundaries) belongs only in `CLAUDE.md`. Don't copy it into the agents or the skill.
- **Keep the READMEs in sync.** A change to `README.md` that affects meaning must be mirrored in `README.es.md`. English wins on any discrepancy.
- **Preserve the gates.** Don't weaken or remove the spend gate (`PREMIUM-APPROVED`) or the author-is-not-the-reviewer rules. If you rename the token or an agent, run `git grep -l PREMIUM-APPROVED` and update every hit in the same change.
- **Surgical diffs.** Match the existing voice; no broad cosmetic rewrites.
- **Don't strip provenance.** Dated compatibility stamps (e.g., the Codex CLI version) are deliberate - update them when you re-verify, never delete them.

## Validation

No tests exist. Sanity-check with `git status --short`, `git diff --check`, and confirm any Markdown links you add resolve to real files.
