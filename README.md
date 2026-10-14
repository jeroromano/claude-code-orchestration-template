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

Already keep an `AGENTS.md` (Codex reads it)? Claude Code skips that file whenever a `CLAUDE.md` exists - unless its Project instructions setting reads both (verified October 2026, Claude Code 2.1.299) - so add an `@AGENTS.md` line to `CLAUDE.md`: both tools then read the same conventions, with no copy to drift.

## First run

Once `/agents` lists the four workers, exercise the protocol once on something harmless:

1. Pick a small, fully-specified task - a unit test to add, a rename, a docstring pass.
2. Ask for it through the protocol: *"Delegate to fast-worker: \<task\>. Follow the delegation protocol."*
3. Three behaviors prove the template is live: a task spec (GOAL / ALLOWED FILES / ACCEPTANCE / VALIDATION...) appears *before* any delegation, the diff comes back with a report instead of being silently applied, and the report opens with a provenance line - naming the model the harness actually resolved when the subagent's context states it, or "model not reported by harness" when it does not (the honest fallback is part of the protocol, not a failure).
4. Before merging anything behavioral, ask for the independent review pass and check that the reviewer is not the author: `/codex:review --base main --background` for Claude-authored work (with the plugin; without it, Grok Build if you installed that optional fallback and the run is eligible - see [its section](#optional-grok-build-plugin-cross-family-fallback-reviewer) - else the `diff-reviewer` agent), the `diff-reviewer` agent for Codex-authored work. On native Windows, ask for the review *through the protocol* instead of typing that command yourself: the direct command hangs there, and the `auto` transport ships the same review as an inline task instead - see [Native Windows: reviews ship as inline tasks](#native-windows-reviews-ship-as-inline-tasks).

A complete worked example - task spec, routing decision, reviewer findings, fix round, convergence - lives in [docs/example-workflow.md](docs/example-workflow.md).

## The spend gate

`premium-reasoner` is a billing boundary, so it has layered guards instead of good intentions:

- **Authorization token.** It refuses to run unless the delegation prompt carries `PREMIUM-APPROVED`, which the orchestrator is instructed to add only on your explicit authorization. Fails closed: a refusal costs one line, not an analysis.
- **Bounded worst case.** `maxTurns` caps how long a runaway invocation can spend; `effort` is pinned in frontmatter so the cost profile is explicit configuration, not inherited ambient state.
- **Runtime disclosure.** Every report opens by telling you to verify which pool the invocation drew from (`/usage`). No rates or dates are hardcoded - billing regimes change faster than templates; artifacts should encode invariants and disclose regimes at runtime.

No Fable access? Edit one marked line in the agent (`model:`) or delete it - routing degrades cleanly. If you rename the agent or token, run `git grep -l PREMIUM-APPROVED` and update every file it returns in the same commit.

**Verify the pin actually resolved.** After install, check `/agents`: frontmatter model values are validated against your organization's model allowlist, and an excluded or unavailable value is silently skipped - the subagent then runs on the *inherited* model. The gate controls *when* premium runs; only `/agents` confirms *what* it runs on.

**Provenance and spend in every report.** Delegated reports open with a provenance line - the agent, the model the harness actually resolved (a *verified* value only when the subagent's own context states it - observed harness behavior, not a documented contract; the agent otherwise writes "model not reported by harness". When present, the quote exposes a silently-skipped pin per invocation, complementing `/agents`), and the effort tier (pinned or inherited). The orchestrator closes each result with the harness-reported token usage for that delegation and a running total - or "usage not reported by harness" when the runtime does not expose it; nothing is inferred. Codex lines are labeled *requested* (rescue flags) or *configured* (`.codex/config.toml`, or "CLI default (unknown)" when none exists) instead: the Codex CLI echoes neither model nor usage at runtime, and its token spend is visible only on your OpenAI-side usage page. No report converts tokens to money - rates rot; whether Claude tokens drew from plan quota or API credits stays a runtime check via `/usage`.

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
**Interface coupling, declared:** this template's routing table references `/codex:review`, `/codex:adversarial-review`, `/codex:rescue`, `/codex:status` and `/codex:result`, tested against [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) as of July 2026 (Codex CLI 0.144.x); re-checked September 2026 - plugin 1.0.6 is still the latest release, and GPT-6.1 Sol needs Codex CLI 0.159.0 or newer. If the plugin renames commands, update the skill's routing table. The inline-task review transport additionally drives the plugin's *internal* companion CLI (`scripts/codex-companion.mjs task --background --prompt-file --json`) - the only path that dispatches a review with a capturable, cancellable job ID; it is not a public command, so re-verify those flags and the JSON payload on every plugin update (verified against plugin 1.0.6; re-verified August 2026 - the companion's `--effort` flag caps at `xhigh` and rejects `max`, so only the direct path can pin it per invocation; an omitted flag can still inherit `max` from the resolved config).

Without the plugin nothing breaks: the protocol routes reviews to the local `diff-reviewer` instead - or first to Grok Build, if you installed that optional fallback and the run is eligible ([below](#optional-grok-build-plugin-cross-family-fallback-reviewer)) - by instruction in the routing table, not by interception; a `diff-reviewer` verdict will honestly note that same-family review is the weaker guarantee.

**Do not enable the plugin's automatic review gate as a default.** It registers a Stop hook that can loop Claude and Codex against each other and drain both quota pools. Review manually, at branch level: `/codex:review --base main --background` (replace `main` with your default branch if it differs; on native Windows the transport ships this same review as an inline task instead - next section).

### GPT-6.1 Sol and GPT-6 Astra: model and effort routing

With the plugin installed, this template routes Codex work to **GPT-6.1 Sol** in two roles, in this order, and reserves **GPT-6 Astra** for one pass:

- **Independent auditor (primary).** Sol reviews Claude-authored diffs. Effort: the highest each invocation path exposes - `max` (the highest allowed effort, and accepted by the TOML) on the direct path, where the raw CLI also pins it per invocation (`codex review --commit <sha> -c model_reasoning_effort="max"`, verified August 2026 - a single-commit form: for a branch-level review, pin `max` in your resolved config and use the flagless command); `xhigh` as the pinnable ceiling on the inline-task path, whose companion `--effort` flag rejects `max` while an omitted flag inherits your resolved config, which may sit higher. `max` is the highest ALLOWED effort: Codex lists an `ultra` level above it, but `ultra` is automatic task delegation to sub-agents - Codex Ultra, banned below. A pin is an exact selection, not a raise - it can lower a `max`-configured machine - so pin only to raise a low config or to replace a banned `ultra`, and verify what actually ran - model, effort and read-only profile - via the session rollout (skill §5-§6).
- **Risk-path adversarial reviewer: GPT-6 Astra.** The frontier tier, and the costlier one on your Codex limits, so it runs only the risk-path adversarial pass - security, payments, tenant isolation: the paths where extra reasoning changes the outcome - at the same per-path ceiling. The plugin's review commands take no model flag: the inline-task transport pins `--model gpt-6-astra` per pass, and on the direct path the protocol checks your resolved `model` first (the adversarial pass runs on `model`; `review_model` steers only `/codex:review`) - on a mismatch it ships that pass inline, or asks you to point `model` at Astra for that run and back afterwards. Astra reviews; it never implements. No Astra on your account? The pass degrades to GPT-6.1 Sol at the same effort, and the report says so.
- **Spec-bound implementer (secondary).** Sol writes only implementations of an already-approved spec - never architecture, plans, or the specs themselves; those stay with Claude (Fable when escalated) - and Astra never writes: rescues pin Sol, on risk paths too. Raise to `--effort high` only for hard bugs or multi-module work:

```
/codex:rescue --fresh --background --model gpt-6.1-sol --effort medium <approved spec>
```

GPT-6.1 Sol needs Codex CLI 0.159.0 or newer - older clients cannot see it, even on an eligible account - plus account access. If the CLI rejects `gpt-6.1-sol`, degrade instead of stalling - but never by rerunning without `--model`: your CLI default is whatever your config names, which may be Astra. The rescue goes to `fast-worker`, a standard review to the fallback reviewer (Grok Build when installed and the run is eligible, else `diff-reviewer`), and the report tells you when updating the CLI restores the route. Which model a review runs on depends on the transport: the inline-task transport pins it per pass; the direct commands take no model flag and inherit your configuration (below). Either way the rollout check verifies the model that actually ran and invalidates any other.

The reviewer is always chosen by authorship - no model approves its own diff: Claude-authored -> Codex review; Codex-authored -> the local `diff-reviewer` or a human; mixed -> each portion reviewed by an agent that did not write it.

`/codex:review` accepts no per-invocation model or effort flags - it inherits your RESOLVED Codex configuration: the repo-local `.codex/config.toml` when that file exists (and, per the Codex docs, only in repos the CLI trusts), otherwise your global `~/.codex/config.toml`. This template ships no `.codex/config.toml`, so out of the box the direct commands review on whatever your global config or CLI default resolves (the inline-task transport pins the model itself; its effort inherits when `--effort` is omitted); they run on Sol at the ceiling only once you pin it yourself. `review_model` overrides `model` for `/codex:review` (per the Codex config reference), so pin both - or leave `review_model` unset:

```toml
model = "gpt-6.1-sol"
review_model = "gpt-6.1-sol"
model_reasoning_effort = "max"
```

Effort has no per-review lifecycle to remember: with the ceiling pinned, the risk-path pass runs at the same maximum as every other review - only its model differs (Astra, above). Never set `ultra` here: the protocol refuses to inherit it. One warning - the CLI does not validate this value locally: a typo passes parsing, prints in the run header, and dies mid-review at the API with HTTP 400 (the rejection lists the accepted values - measured August 2026: `none|minimal|low|medium|high|xhigh|max`; `ultra` has since been added).

**Do not use Ultra.** Codex Ultra is multi-agent orchestration; this template is already the orchestration layer, so Ultra would duplicate it and multiply spend on both quota pools. The same goes for the `ultra` reasoning effort: a thread at `ultra` spawns sub-agents on its own, so the protocol treats a review that ran at `ultra` as invalid.

### Native Windows: reviews ship as inline tasks

On native Windows the Codex sandbox cannot spawn processes (verified on Windows 11, July 2026, plugin 1.0.6 / Codex CLI 0.144.0): every command a `/codex:review` or `/codex:adversarial-review` job attempts exits -1, and the job hangs "running" indefinitely instead of failing. The task path is unaffected - prompts that carry everything need no sandbox.

The template therefore makes the review transport configurable - one bullet in `CLAUDE.md`'s review gate, default `auto`:

- `auto` - `direct` everywhere except native Windows, where it resolves to `inline-task`.
- `direct` - the two commands above (the knob selects delivery only; the effort ceilings above apply on every transport).
- `inline-task` - Claude computes the diff locally, writes the audit prompt - with an explicit "analyze only this diff, run nothing, request nothing" contract - to a temp file, and dispatches it read-only in the background on the plugin's companion task runtime (`task --background --prompt-file --json`; model pinned per pass - GPT-6.1 Sol, or GPT-6 Astra for the risk-path pass; effort inherited from your resolved config by default - a pin only raises a low config to a floor, or replaces an `ultra` one, capped at the companion's `xhigh` flag ceiling; never `--write`, never `minimal` - Sol rejects it). It captures the `task-*` job ID from the dispatch and deletes the temp file immediately, splits diffs over ~50 KB at file boundaries, applies a deadline with cancellation against the captured ID so no review job is left hanging, and then validates every finding against the full repository (confirmed / discarded / needs design decision - preserving the reviewer's original findings and verdict verbatim) before reporting. The dispatch is deliberately *not* `/codex:rescue`: its `--background` backgrounds the Claude-side agent and returns no job ID to cancel against, and the prompt re-enters a ~32 KB Windows argv ceiling on its way to the runtime. Full mechanics: delegation-protocol skill §6.

Who reviews never changes - the transport only changes how a Codex-bound review is delivered. Non-Windows platforms keep the `direct` transport by default; the effort policy is transport-independent. A verified per-machine fix exists (August 2026): install PowerShell 7 outside the WindowsApps store path - the Store shim breaks process spawning inside the sandbox - and restart the plugin's app-server, which caches the PATH it was launched with. When a probe review on a trivial diff completes within its deadline (after the fix, or after a plugin/CLI update), set the knob to `direct`: the only path that can pin `max` per invocation, with a reviewer that verifies its hypotheses against the real repository instead of judging a pasted diff.

## Optional: Grok Build plugin (cross-family fallback reviewer)

```
/plugin marketplace add xai-org/grok-build-plugin-cc
/plugin install grok-build@xai-grok-build
/reload-plugins
/grok-build:check
```

Requirements: the Grok Build CLI on your PATH (its installer puts it in `~/.grok/bin`; Claude Code sessions started before the install will not see it), signed in (run `grok` once interactively, then verify with `grok models`), and Node.js for the plugin's bridge.

**One job only.** When the Codex route is unavailable - plugin absent, quota exhausted, model rejected, a review dispatch that fails or expires - the standard review of a Claude-authored diff goes to Grok before `diff-reviewer`, so the second pass stays cross-family instead of Claude reviewing Claude. It is a fallback, never a substitute for Sol: on a calibration probe (October 2026) over the same 8-file diff GPT-6.1 Sol had reviewed, Grok matched 4 of Sol's 10 findings in full and missed two of its three majors - with no false positive, and one valid finding Sol had missed. A run is eligible when the plugin's check reports ready, everything committed at the reviewed commit may go to xAI (below), and - on native Windows - you OK that run. An invalid Codex verdict is re-run on Codex, never handed to Grok. It never takes the risk-path adversarial pass (its review command documents no focus text), Codex-authored diffs (`diff-reviewer` already reviews those cross-family), or any writing.

**How the protocol runs it** (skill §7): it checks readiness - and on native Windows asks for your OK for that run, because Grok's read-only sandbox is not enforced there - snapshots your working tree, creates a disposable detached git worktree at the commit under review (tracked, committed files only: untracked files such as an untracked `.env` stay out, but anything tracked is in it) and drives the plugin's bridge script in the background (`grok-bridge.mjs review --background --base <base> --scope branch --cwd <worktree> --json`) under the same 25-minute deadline and single extension as Codex reviews. Effort: `high`, the bridge's cap, unless your Grok config's `models.default_reasoning_effort` already sits at `xhigh` (then it is inherited); never `--model`. Right after the run starts, it reads the tools Grok was offered and stops the run if any falls outside a measured read-only allowlist - a shell, editing or sub-agent tool (a check just after the start, not a barrier: it limits exposure rather than preventing a first call). Afterwards it reads Grok's session record to verify the read-only profile, the effort and that every tool call was flagged read-only, checks that the worktree's `git status` is clean and that yours matches the snapshot, validates each finding against the repository, reports tokens (never money) plus any call beyond reading and searching (a `web_fetch` URL, for instance), and removes the worktree. Any check that fails - or cannot be read - sends the review to `diff-reviewer`; a call not flagged read-only, or a `git status` of yours that no longer matches, is reported to you first, as an incident. These checks are tripwires, not a sandbox: they show what Grok called and that both `git status` outputs are as expected, not that nothing else happened.

**Interface coupling and known issues, declared** (verified October 2026 against plugin 0.2.1 and Grok Build CLI 1.0.52): `/grok-build:review` is not model-invocable, so the protocol drives the plugin's *internal* bridge script - re-verify its flags, job payload and session-record fields on every plugin or CLI update. Upstream issues open at that date: on native Windows the read-only sandbox is not enforced (#24) - Grok's docs say its review agent can run shell commands, although the run measured here was offered no shell or editing tool - and a worktree separates checkouts but confines nothing, hence the per-run OK and the tool allowlist above; `--model` hangs headless (#31); `import` fails (#21) and `critique` likely does (#35); the plugin's session hook grows Claude's env file on every start until Bash commands break (#18, fix proposed in PR #19); and it exports its data directory under the generic `CLAUDE_PLUGIN_DATA`, which other plugins' scripts - the Codex companion included - then inherit (#49). A plugin update overwrites any local fix you apply for the last two.

## Evaluated, not adopted: Gemini 4 Argon

Gemini 4 Argon was weighed as a third external reviewer in October 2026 and kept out for now. A consumer Google account reaches it only through Google Antigravity, whose terms let Google use your interactions to improve its products - with human access, no stated retention limit and no defined training opt-out - and Gemini CLI neither accepts consumer logins any more nor serves the model. The evaluation, the verified facts and the conditions to revisit (a no-training route, a scriptable CLI, enforced read-only, a calibration probe) live in [docs/specs/gemini-4-argon-reviewer.md](docs/specs/gemini-4-argon-reviewer.md).

## Security note

Codex runs on OpenAI infrastructure, and Grok Build - if you install that fallback - on xAI's: every delegated task or review ships the touched code there. Grok goes further: it reads the repository at the reviewed commit, so everything committed there can reach xAI - which is why the protocol dispatches it only when that whole tree may leave the machine. Never delegate secrets, `.env*`, credentials, customer data or real-data fixtures. For confidential or client repos, adapt the **Data boundary** section in `CLAUDE.md`: local `diff-reviewer` by default, Codex and Grok opt-in per task.

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
