# Claude Code Orchestration Template

**English** | [Español](README.es.md)

Model routing, spend gates and review gates for [Claude Code](https://code.claude.com) - with clean degradation when the optional pieces are missing.

**What it is.** A drop-in orchestration layer for [Claude Code](https://code.claude.com): a few specialized subagents, a reusable Claude Code skill, and one short policy file (`CLAUDE.md`). It routes each task to the cheapest capable model and keeps an independent reviewer between the author and the merge. Optionally pairs with the [OpenAI Codex](https://github.com/openai/codex-plugin-cc) CLI for cross-family AI code review.

**Who it's for.** Developers using Claude Code who want disciplined agent orchestration - deliberate task routing, isolated subagents, and review workflows - instead of ad-hoc prompting, with optional cross-model validation.

**Who it's not for.** If your tasks are small and your sessions short, writing a delegation spec costs more than doing the work - prompt directly instead; the protocol itself forbids delegating a task smaller than its spec. If you want an autonomous pipeline that ships code without a human decision, this template is the opposite by design: every gate ends in a human. And if you came for token savings, read **What this is not** below first - delegation usually raises the bill.

**What it solves.** Long multi-agent sessions drift and overspend. This template keeps your main session's context clean, routes mechanical work to cheap models and hard reasoning to expensive ones, and never lets a model approve its own diff.

**What's included:** four Claude Code agents (fast worker, deep reasoner, premium reasoner, diff reviewer), a delegation-protocol skill, spend gates, and manual review gates - all plain Markdown, no scripts or dependencies. See [What's in the box](#whats-in-the-box) for the full map.

**What this is not: a token-saving trick.** Delegation usually *increases* total tokens: every worker rebuilds its own context, and only a summary returns. Anthropic's cost guidance puts agent teams at approximately 7x the tokens of a standard session when teammates run in plan mode; ordinary subagent delegation sits below that, but the mechanism - and the direction - is the same. What you actually buy:

- **Context isolation.** Workers burn their tokens in their own context windows; only a summary returns. Your main session stays clean, which is what preserves decision quality over long sessions.
- **Pool arbitrage.** Work lands where it is cheapest: mechanical tasks on cheaper models, and optionally on a different provider's quota (OpenAI Codex) that you may already be paying for.
- **Decorrelated review.** Two models from the same family share blind spots. A reviewer from another family catches correctness bugs the author's family tends to miss.

If a workflow promises "70% savings", it is selling you a tweet. This one sells routing discipline: cheap model where the volume is, expensive model where the leverage is (planning and diff review), different family where independence matters.

## What's in the box

Each rule lives in the primitive whose loading semantics it matches - that placement is the design:

| File | Loads | Contents |
|---|---|---|
| `CLAUDE.md` | every session (fixed tax - kept short) | the policy: model routing, risk paths, review rules |
| `.claude/agents/fast-worker.md` (Sonnet) | on invocation | executes small, fully-specified tasks; smallest safe diff |
| `.claude/agents/deep-reasoner.md` (Opus) | on invocation | one hard scoped question in, one decision-grade analysis out; read-only |
| `.claude/agents/premium-reasoner.md` (top tier) | on invocation | the escalation rung; spend-gated, see below |
| `.claude/agents/diff-reviewer.md` (Sonnet) | on invocation | adversarial branch review before merge; read-only; local fallback |
| `.claude/skills/delegation-protocol/SKILL.md` | when delegating | task-spec template, routing table, quota fallbacks |

The agents are project-agnostic on purpose: everything project-specific (risk paths, validation commands, data boundaries) lives only in `CLAUDE.md` - one source of truth.

## Compatibility

Works wherever Claude Code runs: macOS, Linux, and Windows (native or WSL). The template is plain Markdown - no scripts, no binaries, no absolute paths, no platform assumptions. The example commands inside the agents (`git diff`, `grep`) execute in Claude Code's Bash tool, available on every platform (on native Windows it runs through Git Bash, which ships with Git for Windows). The only shell-specific content is the Uninstall section, which provides both bash and PowerShell blocks.

## Install

1. Click **Use this template** (or copy `CLAUDE.md` + `.claude/` into your repo root).
2. Fill the **Project commands** block in `CLAUDE.md` (build / test / lint). Every delegation spec anchors on them - without validation commands the whole protocol is decorative.
3. Replace the **Risk paths** placeholders with your real ones (auth, payments, migrations, user data...).
4. Restart your Claude Code session - subagent files load at session start - and verify with `/agents` that the four workers appear.

What `/agents` should show - the exact layout varies by Claude Code version, so treat this as the expected shape, not a promised rendering:

```text
Project agents (.claude/agents)
  fast-worker        sonnet
  deep-reasoner      opus
  premium-reasoner   fable
  diff-reviewer      sonnet
```

A missing name means the file did not load: check the path (`.claude/agents/*.md` at the repo root) and restart the session. A model value different from the one the file pins means your organization's allowlist silently excluded it - see [the spend gate](#the-spend-gate).

## First run

Once `/agents` lists the four workers, exercise the protocol once on something harmless:

1. Pick a small, fully-specified task - a unit test to add, a rename, a docstring pass.
2. Ask for it through the protocol: *"Delegate to fast-worker: \<task\>. Follow the delegation protocol."*
3. Two behaviors prove the template is live: a task spec (GOAL / ALLOWED FILES / ACCEPTANCE / VALIDATION...) appears *before* any delegation, and the diff comes back with a report instead of being silently applied.
4. Before merging anything behavioral, ask for the independent review pass - `/codex:review --base main --background` with the plugin, the `diff-reviewer` agent without it - and check that the reviewer is not the author.

A complete worked example - task spec, routing decision, reviewer findings, fix round, convergence - lives in [docs/example-workflow.md](docs/example-workflow.md).

## The spend gate

`premium-reasoner` is a billing boundary, so it has layered guards instead of good intentions:

- **Authorization token.** It refuses to run unless the delegation prompt carries `PREMIUM-APPROVED`, which the orchestrator is instructed to add only on your explicit authorization. Fails closed: a refusal costs one line, not an analysis.
- **Bounded worst case.** `maxTurns` caps how long a runaway invocation can spend; `effort` is pinned in frontmatter so the cost profile is explicit configuration, not inherited ambient state.
- **Runtime disclosure.** Every report opens by telling you to verify which pool the invocation drew from (`/usage`). No rates or dates are hardcoded - billing regimes change faster than templates; artifacts should encode invariants and disclose regimes at runtime.

No Fable access? Edit one marked line in the agent (`model:`) or delete it - routing degrades cleanly. If you rename the agent or token, run `git grep -l PREMIUM-APPROVED` and update every file it returns in the same commit.

**Verify the pin actually resolved.** After install, check `/agents`: frontmatter model values are validated against your organization's model allowlist, and an excluded or unavailable value is silently skipped - the subagent then runs on the *inherited* model. The gate controls *when* premium runs; only `/agents` confirms *what* it runs on.

## Optional: OpenAI Codex plugin (cross-family review)

```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

Requirements: Claude Code, Node.js 18.18+ (the plugin can install the Codex CLI via npm), and a ChatGPT account on any plan (including Free) or an OpenAI API key. Usage counts against **your Codex limits, not Claude's** - that separation is the arbitrage.

<!-- MAINTAINER NOTE (deliberate design - do not "fix"): the date and CLI version below are provenance,
not rot. Rot = a statement time makes FALSE (prices, quotas, regimes): those are banned from this repo.
Provenance = a statement time makes OLD (when compatibility was last verified): its whole function is to
let the reader judge staleness. Update the stamp when you re-verify; never remove it. -->
**Interface coupling, declared:** this template's routing table references `/codex:review`, `/codex:adversarial-review`, `/codex:rescue`, `/codex:status` and `/codex:result`, tested against [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) as of July 2026 (Codex CLI 0.142.x). If the plugin renames commands, update the skill's routing table.

Without the plugin nothing breaks: the protocol routes reviews to the local `diff-reviewer` instead - by instruction in the routing table, not by interception - and its verdict will honestly note that same-family review is the weaker guarantee.

**Do not enable the plugin's automatic review gate as a default.** It registers a Stop hook that can loop Claude and Codex against each other and drain both quota pools. Review manually, at branch level: `/codex:review --base main --background` (replace `main` with your default branch if it differs).

## Security note

Codex runs on OpenAI infrastructure: every delegated task or review ships the touched code there. Never delegate secrets, `.env*`, credentials, customer data or real-data fixtures. For confidential or client repos, adapt the **Data boundary** section in `CLAUDE.md`: local `diff-reviewer` by default, Codex opt-in per task.

## Uninstall

Everything here is declarative files inside your repo - no hooks, no daemons, no state outside the tree. Install surface = uninstall surface.

**If you merged this template into an existing `CLAUDE.md`, do not delete that file** - remove only the orchestration sections by hand (or restore your pre-merge version with `git checkout <ref> -- CLAUDE.md`). The commands below are safe only for a standalone install:

```bash
# bash / zsh / Git Bash
rm -f CLAUDE.md
rm -f .claude/agents/{fast-worker,deep-reasoner,premium-reasoner,diff-reviewer}.md
rm -rf .claude/skills/delegation-protocol
```

```powershell
# PowerShell (Windows) - brace expansion does not exist here, hence the explicit list
Remove-Item CLAUDE.md -Force
Remove-Item .claude\agents\fast-worker.md, .claude\agents\deep-reasoner.md, .claude\agents\premium-reasoner.md, .claude\agents\diff-reviewer.md -Force
Remove-Item .claude\skills\delegation-protocol -Recurse -Force
```

Restart the session; `/agents` should list none of them.

## Languages

English is canonical - this file wins on any discrepancy. A maintained Spanish translation lives in [README.es.md](README.es.md); Portuguese (or any other) translations are welcome via PR.

The instruction files (`CLAUDE.md`, agents, skill) are English **by design** and are not translated: Claude Code reads them in English and still converses with you in whatever language you use, so a single canonical instruction set costs nothing at the interaction layer - and prevents the worst kind of drift, where two translations of the same policy produce two different behaviors.

## Credits

Designed for [Claude Code](https://code.claude.com), optionally pairing with [OpenAI's Codex plugin for Claude Code](https://github.com/openai/codex-plugin-cc).

This repository is not affiliated with, endorsed by, or maintained by Anthropic or OpenAI.

## Disclaimer

Experimental template. It guarantees no token savings, no cost reduction, and no code quality by itself. A human reviews every diff before merge - the review gates exist to inform that human, never to replace them.

## License

MIT - see [LICENSE](LICENSE).
