# Changelog

Notable changes to this template. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions are annotated git tags following [Semantic Versioning](https://semver.org/).

## [Unreleased]

Changes below address the independent review of v0.5.0: GPT-6.1 Sol in three inline-task chunks (rollout-verified `xhigh`, read-only, zero tool and sub-agent calls, Codex CLI 0.159.1 - the run doubled as the live probe of plugin 1.0.6 on the new CLI, and passed) plus a diff-reviewer cross-chunk pass; 3 majors, 7 minors, no blockers.

### Changed
- When GPT-6.1 Sol is unavailable, work falls back to the local agents - a rescue to fast-worker, a standard review to diff-reviewer - and never to a rerun without `--model`: the CLI default is whatever the resolved config names, which may be Astra, so that rerun could make Astra implement, and a standard review rerun that way always failed rollout verification, looping (skill §2/§5/§6, both READMEs, spec decision 4; CLAUDE.md and the example workflow state the rescue half). A risk-path pass already degraded from Astra, with Sol unavailable too, goes to diff-reviewer with the risk-path focus text (skill §5, spec decision 4).
- Astra never implements - rescues pin `gpt-6.1-sol` on risk paths too - and its degradation to GPT-6.1 Sol is stated in CLAUDE.md as well (CLAUDE.md, skill §2, both READMEs). Both interpretations the v0.5.0 spec flagged (writing effort `medium`/`high`; Astra reviews only) are confirmed by the human.

### Fixed
- The pin rule now carries its `ultra` exception everywhere it is stated ("pin only to raise a low config or to replace a banned `ultra`"): CLAUDE.md and both READMEs told the reader to pin only upward while also requiring the downward pin that replaces an inherited `ultra` (CLAUDE.md, both READMEs).
- The `thread_settings_applied` claim is stated per measured version - missing from every rollout measured from 0.153.4, 0.157.0 and 0.159.1 and from most from 0.154.0 - instead of the v0.5.0 entry's inaccurate "absent from 0.153 and later" cutoff; the rule (never require it, read `turn_context`) is unchanged (skill §6 step 5, spec decision 6).
- The READMEs scope config inheritance to the direct commands (the inline-task transport pins the model per pass) and name the key that steers each pass: the adversarial pass runs on `model`, `review_model` steers only `/codex:review` (plugin 1.0.6 source), as the skill's preflight now also says (skill §5, both READMEs).
- Astra's accepted-effort claim now traces to OpenAI's docs (Astra rejects `none`; GPT-6.1 Sol rejects `none` and `minimal`), and the spec drops a price comparison the no-prices rule bans (skill §5, spec).
- The inline-task deadline note records that its timings are GPT-5.6 Sol's, adds the GPT-6.1 Sol `xhigh` timings from this review (3m 56s - 5m 40s on 31-53 KB prompts) and marks Astra unmeasured (skill §6 step 5).

## [0.5.0] - 2026-09-29

### Changed
- Codex routing moves to the GPT-6 generation: GPT-6.1 Sol (`gpt-6.1-sol`) replaces GPT-5.6 Sol as the default model for reviews and spec-bound writing, at unchanged efforts (reviews at the per-path ceiling, writing at `medium`/`high`); GPT-6 Astra (`gpt-6-astra`), the costlier frontier tier, is reserved for the risk-path adversarial pass of Claude-authored diffs, degrading to GPT-6.1 Sol - disclosed in the report - when the account lacks it. Sol's "limited-access preview" caveat is replaced by its real gate: Codex CLI 0.159.0 or newer (older clients cannot see the model) plus account access (CLAUDE.md, skill §2/§5/§6, both READMEs, `llms.txt`, example workflow).
- The model joins the fail-closed effort trio: preflight checks the resolved `model` - and `review_model`, which overrides it for the built-in reviewer - dispatch pins it per pass where the path allows (inline-task `--model`), and the rollout must show the ordered model or its declared degradation (skill §5/§6). The recommended TOML pins `review_model` alongside `model` (both READMEs).
- Rollout verification reads the per-turn `turn_context` event (`model`, `effort`, `sandbox_policy`, `permission_profile`): the `thread_settings_applied` event the step named is absent from rollouts written by Codex CLI 0.153 and later (measured 2026-09-29), so the fail-closed check would have invalidated every verdict on a current CLI (skill §6 steps 5/7).

### Added
- The no-Ultra rule names its concrete key: the `ultra` reasoning effort, listed above `max` in the Codex catalog as "Maximum reasoning with automatic task delegation" - rollouts at `ultra` announce proactive multi-agent delegation and spawn sub-agents, rollouts at `max` are told not to. `max` is now "the highest allowed effort", not "the top of the enum"; a resolved `ultra` is never inherited (preflight pins the path's ceiling instead), never pinned, and a rollout at `ultra` or carrying any sub-agent call invalidates the verdict (CLAUDE.md, skill §2/§5/§6, both READMEs, `llms.txt`).
- `docs/specs/codex-gpt-6-refresh.md`: the approved spec for this change.

### Notes
- The Codex plugin is unchanged: openai/codex-plugin-cc 1.0.6 is still the latest release (checked 2026-09-29), so every companion-level statement stands. On the Claude side the agents pin family aliases, which already resolve to Fable 5.1, Opus 5.5 and Sonnet 5.5 - no instruction change; only the worked example's sample provenance line is refreshed from a live probe.

## [0.4.0] - 2026-08-09

### Changed
- Review effort ladder replaced by per-path ceilings: reviews run at the highest effort their invocation path exposes - `max` via `direct` (the raw CLI pins it per invocation: `codex review --commit <sha> -c model_reasoning_effort="max"` - a single-commit form; branch-level direct reviews pin `max` in the resolved config and use the flagless plugin command, which inherits it), `xhigh` as the pinnable ceiling via `inline-task` (the companion's `--effort` flag rejects `max`; an omitted flag inherits the resolved config, which may sit higher). `max` stops being an "exceptional, human-authorized escalation": it is the top of the API's effort enum (`none|minimal|low|medium|high|xhigh|max`), the risk-path adversarial pass runs at the same maximum because no higher rung exists, and the old "raise to `xhigh`, then set it back" lifecycle is deleted (CLAUDE.md, skill §2/§5/§6, both READMEs, `llms.txt`). Measured 2026-08-08/09, plugin 1.0.6 / Codex CLI 0.144.x.
- Inline-task chunk deadline raised from 15 to 25 minutes, PROVISIONAL, with exactly one same-length extension for a job still reported `running`: a measured `max` pass took 16m 27s on a 16 KB diff and would have been cancelled at the old default, discarding the blocker it found; the two `max` timings on record differ 3.7x at similar sizes, so the bound gets re-measured near the ~50 KB chunk ceiling (skill §6 step 5).
- The *configured* provenance tier now names the config file that actually resolves: the repo-local `.codex/config.toml` governs only when it exists (and the repo is trusted); otherwise the global `~/.codex/config.toml` does - "CLI default (unknown)" only when neither sets a value (skill §4; both READMEs state the underlying resolution order, without the tier vocabulary).

### Added
- Fail-closed effort compliance in the review gate: preflight the resolved config before dispatching (a pin is an exact selection and can LOWER effort - pin only to raise a low config to a floor), then verify the effort that actually ran from the session rollout (`~/.codex/sessions/<Y>/<M>/<D>/rollout-*-<session-id>.jsonl`, `thread_settings_applied`: real effort, model, and the permission profile evidencing the read-only sandbox), which promotes Codex provenance from *requested*/*configured* to *verified*. A rollout below the ordered effort - or one that cannot be read - invalidates the verdict: its findings are not classified and the review is re-dispatched (skill §4/§5/§6, CLAUDE.md).
- Typo warning on `model_reasoning_effort`: the Codex CLI does not validate the value locally - an invalid one passes parsing, prints in the run header, and dies mid-review at the API with HTTP 400 (skill §5, both READMEs).
- Top-tier design-authority rule in CLAUDE.md's model routing: an APPROVED design or spec authored by the escalated top tier is authoritative for the cheaper model implementing it - implemented to the letter, deviations flagged to the human BEFORE deviating; overridden only by a review-caught concrete error (repro or file:line) or an explicit human decision. Audit and review findings are not specs: they stay inside the finding-validation flow and are never auto-applied.
- The native-Windows sandbox breakage now documents its verified per-machine fix (2026-08-08): PowerShell 7 installed outside the WindowsApps store path, plus a restart of the plugin's app-server, which caches the old PATH. After a probe review passes, `direct` becomes available - the only path that can pin `max` per invocation, with a reviewer that verifies hypotheses against the real repository (CLAUDE.md, skill §6, both READMEs, `llms.txt`).
- `docs/specs/review-effort-ceilings.md`: the approved spec for this change.

### Fixed
- Ten findings and a nit from the independent review of this delta itself (GPT-5.6 Sol in two inline-task chunks at rollout-verified `max` - the review inherited `max` from the resolved config, demonstrating the inheritance rule with its own run - plus a diff-reviewer cross-chunk pass; 1 blocker, 6 majors, 1 minor, 2 concerns, all classified against the repo and fixed below, plus the nit - a conservative size estimate in the spec's expected-diff-shape table - which needed no fix): the spec's context no longer states a billing regime and its no-regime check is scoped to lines the delta adds; "the only path exposing `max`" corrected everywhere to "the only path that can pin `max` per invocation" (inline-task reaches `max` by config inheritance); the raw-CLI `codex review --commit` example is labeled a single-commit form across its full five-file footprint and no longer reads as a substitute for branch-level review; rollout verification defines per-path session-id correlation (companion jobs name the id in the job result; a direct review with no correlatable id is unverifiable, hence invalid - never resolved by newest-rollout); an unset or indeterminate configured effort counts as below the target (pin the floor); the design-authority rule binds only approved designs/specs, never audit findings; a rollout whose permission profile is not read-only also invalidates the verdict; transport wording is scoped to delivery selection; the CHANGELOG citation for the configured-tier fix no longer implies the READMEs use provenance-tier vocabulary.

## [0.3.1] - 2026-07-09

Changes below address the findings of an independent GPT-5.6 Sol audit of v0.3.0 (four findings, all verified against the plugin source, 1.0.6).

### Changed
- The canonical inline-task dispatch is now the plugin's companion CLI - `task --background --fresh --prompt-file <file> --json` with the model/effort pins - instead of `/codex:rescue --background`: the slash command backgrounds the Claude-side subagent and strips `--background` before the runtime `task` call, so it never returns the `task-*` job ID the deadline/cancel mechanics enforce against, and it re-materializes the prompt as a Bash argv (~32 KB Windows ceiling) on its way to the runtime - a chunk legal under the ~50 KB budget could fail in dispatch. The job ID is captured from the `--json` payload (no parseable ID = failed dispatch, fall back to diff-reviewer); read-only is structural (no `--write` flag) rather than prompt-interpreted; and the companion dependency is a declared internal-interface coupling, re-verified on plugin updates (skill §6, CLAUDE.md, both READMEs, example workflow, transport spec).

### Fixed
- The temp file carrying the audit prompt - and therefore the diff - is deleted the moment its dispatch returns, success or error, instead of never: the queued job persists its own copy of the prompt (skill §6 steps 3/5).
- The README's first-run and manual-review steps no longer point native-Windows users at typing `/codex:review` directly, which bypassed the `auto` transport and reproduced the very hang the transport exists to avoid; they route the review through the protocol and link the Windows section (both READMEs).

## [0.3.0] - 2026-07-09

### Added
- Review transport knob in `CLAUDE.md`'s review gate (`auto | direct | inline-task`, default `auto`) and the inline-task procedure as delegation-protocol skill §6: diff computed locally (`git merge-base` scope, so committed and uncommitted work both ship; mixed-authorship branches scoped to the Claude-authored paths) and embedded in a read-only, background `/codex:rescue` prompt (effort pinned per invocation: `high`, `xhigh` on risk paths; never `--write`, never `minimal` - Sol rejects it), a no-commands/no-files contract with a collision-safe delimiter, file-boundary splitting above ~50 KB (54 KB verified; single oversized files route to diff-reviewer), a per-chunk deadline enforced against recorded job IDs with `/codex:cancel` cleanup so no review job is left hanging, and mandatory Claude-side validation of every finding against the full repository (confirmed / discarded / needs design decision; the reviewer's original findings and verdict are preserved verbatim, disputed findings and blockers are cleared only by the human, fixes are never auto-applied). Spec: `docs/specs/review-transport.md`.

### Fixed
- The review gate no longer stalls on native Windows: `auto` routes reviews there through `inline-task` instead of `/codex:review` / `/codex:adversarial-review`, whose sandbox cannot spawn processes (every command exits -1 and the job hangs; verified on Windows 11, July 2026, plugin 1.0.6 / Codex CLI 0.144.0). Who reviews is unchanged - only the delivery of Codex-bound reviews moves; non-Windows platforms keep the previous behavior exactly.

## [0.2.1] - 2026-07-09

Changes below address the findings of an independent GPT-5.6 Sol audit of v0.2.0.

### Changed
- `diff-reviewer` now fails closed on unstated authorship: UNKNOWN instead of assumed Claude-family. The same-family caveat still fires, but no Codex routing is recommended until the author is stated - so a Codex-authored diff can no longer be routed back to Codex by an authorship assumption (skill §3, diff-reviewer, both specs).
- Degradation defined for the Sol rescue pin: GPT-5.6 Sol is a limited-access preview, so when the CLI rejects `--model gpt-5.6-sol` the protocol now says to rerun without `--model` (CLI default) or fall back to fast-worker (skill §2/§5, both READMEs, CLAUDE.md, example workflow).
- Provenance and spend claims downgraded from harness guarantees to observed behavior: the resolved model and per-delegation usage are quoted only when the runtime reports them, with explicit "model/usage not reported by harness" fallbacks (skill §4, both READMEs, provenance spec, example workflow, `llms.txt`). Premium-reasoner's cost-notice-before-provenance order is now a declared exception to the first-line rule instead of an undeclared contradiction.

### Fixed
- The `max` effort limitation is attributed to the per-invocation plugin path (`/codex:rescue --effort` and the documented `.codex/config.toml` values stop at `xhigh`), no longer to the Codex CLI as a whole (CLAUDE.md, skill, both READMEs, Sol routing spec).
- The README no longer implies reviews run on Sol out of the box: they run on the CLI's default model until the optional `.codex/config.toml` pin is added, and a project-level Codex config loads only in trusted repos.

## [0.2.0] - 2026-07-09

### Added
- GPT-5.6 Sol routing guidance across `CLAUDE.md`, the delegation-protocol skill and both READMEs: Sol is primarily the independent auditor of Claude-authored diffs (effort `high`; `xhigh` on risk paths; `max` only as an exceptional human-authorized escalation once the CLI supports it) and secondarily a spec-bound implementer via `/codex:rescue --fresh --background --model gpt-5.6-sol --effort medium <approved spec>` (`high` only for hard bugs or multi-module work). `.codex/config.toml` documented as optional configuration - the template still ships none. Codex Ultra explicitly ruled out: the template is already the orchestration layer.
- `docs/specs/gpt-5-6-sol-routing.md`: the approved spec for this change.
- Provenance and spend disclosure in delegated reports: every agent report opens with a provenance line (agent, the model actually resolved by the harness - quoted from the subagent's runtime context, exposing silently-skipped pins per invocation - and effort, pinned or inherited); diff-reviewer's line also names the diff's author, completing the independence trail. The orchestrator quotes harness-reported token usage per delegation plus a running total. Codex values are labeled requested (rescue flags) or configured (`.codex/config.toml`) - the Codex CLI echoes neither model nor usage; tokens are never converted to money, and pool attribution (plan quota vs API credits) stays with `/usage`. Spec: `docs/specs/agent-report-provenance.md`.

### Changed
- Reviewer selection is now authorship-aware in `CLAUDE.md`'s review gate and the skill's routing table (including the risk-path adversarial pass): Claude-authored -> Codex review; Codex-authored -> diff-reviewer or human; mixed -> each portion reviewed by an agent that did not write it. Codex never approves its own diffs. This resolves the ambiguity between `CLAUDE.md` (which routed every behavioral branch to `/codex:review`) and the skill's §3 independence rule.
- `diff-reviewer`: Codex-authored diffs added as an explicit trigger, and the same-family blind-spot warning now depends on the diff's author (it does not apply when the diff is Codex-authored - the local pass is then the cross-family one).

## [0.1.4] - 2026-07-04

### Added
- `docs/example-workflow.md`: a worked end-to-end example - task spec, routing decision, fast-worker report, independent review, fix round, convergence, merge.
- README (EN/ES): a "Who it's not for" paragraph, the expected shape of the `/agents` check after install, and a "First run" section linking the worked example.
- `AGENTS.md` (repo map) and `llms.txt` (start-here index): one-line pointers to the new example doc.

## [0.1.3] - 2026-07-03

### Removed
- The maintainer "Repository metadata" section from both READMEs (English and Spanish). It rendered on the public repo landing page but only served a one-time maintainer setup task (setting GitHub About/topics), which is complete; the recommended values remain in git history if ever needed again.

## [0.1.2] - 2026-07-03

### Added
- Discoverability and AI-readability files: `AGENTS.md`, `llms.txt`, `CHANGELOG.md`, `SECURITY.md`, `CONTRIBUTING.md`.
- README top orientation block (what it is / who it's for / what it solves / what's included) and a maintainer "Repository metadata" section, in both English and Spanish.

## [0.1.1] - 2026-07-03

### Changed
- Scoped `CLAUDE.md`'s review-gate wording to branches with runtime/behavioral surface, deferring the self-merge threshold definition to the `delegation-protocol` skill and removing a literal contradiction between the two.
- Normalized line endings to LF via `.gitattributes`.

## [0.1.0] - 2026-07-03

### Added
- Initial release: the `CLAUDE.md` orchestration policy, four subagents (`fast-worker`, `deep-reasoner`, `premium-reasoner`, `diff-reviewer`), and the `delegation-protocol` skill.
- Author-reviewer independence codified in the delegation protocol, with a convergence-based termination condition for review rounds.
- Review-mandatory threshold defined in the skill: an independent review is required only where a defect can act unmediated (runtime/behavioral surface); pure-doc changes may be self-merged.
- `diff-reviewer` constrained to diagnosis, not solution authorship, to preserve an independent second pass.

[Unreleased]: https://github.com/jeroromano/claude-code-orchestration-template/compare/v0.5.0...HEAD
[0.5.0]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.5.0
[0.4.0]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.4.0
[0.3.1]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.3.1
[0.3.0]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.3.0
[0.2.1]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.2.1
[0.2.0]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.2.0
[0.1.4]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.4
[0.1.3]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.3
[0.1.2]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.2
[0.1.1]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.1
[0.1.0]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.0
