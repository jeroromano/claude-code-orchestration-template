# Changelog

Notable changes to this template. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions are annotated git tags following [Semantic Versioning](https://semver.org/).

## [Unreleased]

_Nothing yet._

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

[Unreleased]: https://github.com/jeroromano/claude-code-orchestration-template/compare/v0.3.1...HEAD
[0.3.1]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.3.1
[0.3.0]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.3.0
[0.2.1]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.2.1
[0.2.0]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.2.0
[0.1.4]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.4
[0.1.3]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.3
[0.1.2]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.2
[0.1.1]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.1
[0.1.0]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.0
