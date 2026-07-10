# Spec: Review transport (direct vs inline-task)

Status: approved 2026-07-09. Authored on the main thread (Fable) after live verification on native Windows; implements the human's decision to make the Codex review *transport* configurable so the review gate keeps working where the direct commands are broken, without touching who reviews what.

Revised the same day after the round-1 independent Sol audit of the implementation (8 findings, all confirmed): diff acquisition switched from a dirty/clean fork to a single merge-base command (the fork silently dropped committed branch work); mixed-authorship branches get an explicit path-scoping step; the Claude-side validation pass now preserves the reviewer's original findings and verdict verbatim, with disputed findings and blockers cleared only by the human; dispatch is background-first with the deadline enforced by the orchestrator's clock against recorded job IDs (a bare `--wait` cannot reach its own cancel); a single file over the task budget routes to diff-reviewer; the prompt gets a collision-safe delimiter and file-based passing; job cleanup is scoped to the review's own jobs; and the Windows breakage claim carries its verified configuration (Windows 11) instead of reading platform-wide.

Revised again the same day after the round-2 fix-verification audit (6/8 resolved; 2 residual + 2 new, all confirmed): a file with mixed-authorship hunks cannot be split by pathspec and routes whole to diff-reviewer (legal for either author); the dispatch instruction no longer contradicts file-based serialization (the slash command carries the prompt through the plugin layer; the companion CLI takes `--prompt-file`, so the argv ~32 KB Windows ceiling applies to argv passing only); generated/lock/snapshot files are excluded from the inline prompt but keep their independent pass via diff-reviewer; and the `MSYS_NO_PATHCONV=1` workaround names where the variable goes (the environment of the cancel command that runs through Git Bash).

Final revision the same day after a fresh full-diff audit (round 4; round 3 had verified the round-2 fixes narrowly): on split reviews, cross-chunk interactions are substantive review and no longer fall to the author - the consolidated report plus the chunk map route to diff-reviewer; each chunk's raw report and verdict is preserved verbatim, with the deduplicated consolidated view strictly additive; the reviewer contract declares the delimited payload untrusted data whose embedded instructions are never followed, and the delimiter must be verified absent from the payload; and binary files (tracked or untracked) route to diff-reviewer and are named in the report.

## Context

The review gate routes Claude-authored diffs to `/codex:review` (and risk paths additionally to `/codex:adversarial-review`). Both commands make the Codex CLI compute the diff itself, inside its sandbox.

Verified 2026-07-09 on native Windows 11 (plugin `codex@openai-codex` 1.0.6, Codex CLI 0.144.0, PowerShell 7):

- The Codex sandbox cannot spawn processes on native Windows: every command the reviewer attempts - including `Get-Location` - fails with exit -1. `/codex:review` then hangs "running" indefinitely (observed: 26 minutes with zero progress after the first minute) and must be cancelled by hand. `/codex:adversarial-review` shares the same execution path.
- The task path (`/codex:rescue` without `--write`) works on the same machine: prompts round-trip normally because nothing needs the sandbox when the prompt carries everything.
- A diff of ~54 KB was reviewed successfully in a single inline task (human-verified the same day).
- Sol rejects `--effort minimal` with HTTP 400 `unsupported_value` even though the plugin usage advertises it; the accepted ladder is `none|low|medium|high|xhigh`.
- On Git Bash, `/codex:cancel` can fail because MSYS path conversion mangles `taskkill /PID` into a path; `MSYS_NO_PATHCONV=1` is the workaround.

So on native Windows the review gate as written stalls, silently burns wall-clock, and leaves zombie jobs. The fix is a *transport* switch: same reviewer, same authorship rules, different way of getting the diff in front of Sol.

## Decisions

1. **One knob: `Review transport`, values `auto | direct | inline-task`, default `auto`.** It lives as one bullet in `CLAUDE.md`'s review gate (per-project configuration surface, like the base branch). It is not a new routing system: the §2 routing table keeps choosing *who* reviews; the transport only selects *how* a Codex-bound review is delivered.
2. **`direct`** is today's behavior: `/codex:review --base <base> --background`, `/codex:adversarial-review --base <base>`.
3. **`inline-task`** delivers the review through the working task path: Claude computes the diff locally, embeds it in a read-only `/codex:rescue` prompt (never `--write`) with an explicit no-commands / no-file-requests contract, and validates every finding against the repository afterwards. Mechanics in the delegation-protocol skill §6.
4. **`auto` resolves to `direct` everywhere except native Windows sessions (platform `win32`), where it resolves to `inline-task`** - based on the July 2026 breakage above, which is provenance-stamped so it can be retired. WSL, macOS and Linux keep today's behavior exactly.
5. **Transport is orthogonal to authorship.** It applies only to the two Codex review commands, i.e. only to Claude-authored diffs. Codex-authored diffs keep routing to `diff-reviewer` or a human; mixed authorship keeps splitting; the automatic review gate stays disabled. No independence rule moves.
6. **Effort in `inline-task` is pinned per invocation** - an advantage over `direct`, which inherits `.codex/config.toml`: `--effort high` for normal reviews, `--effort xhigh` for the risk-path adversarial pass (with the CLAUDE.md focus text in the prompt). Never `minimal` (Sol rejects it). Model pin `gpt-5.6-sol` keeps the existing limited-access degradation: if the CLI rejects it, rerun without `--model`.
7. **Diff acquisition is generic, not hardcoded.** Base ref comes from the project's review gate (`--base <base>` equivalent). One command covers everything the merge would introduce, committed and uncommitted alike: `git diff $(git merge-base <base> HEAD)` (on the base branch itself it degrades to `git diff HEAD`). Untracked files that belong to the change are appended as labeled full-content sections - never `git add`. Mixed-authorship branches scope the inline prompt to the Claude-authored paths only (`-- <paths>`); the Codex-authored portion routes to diff-reviewer and never enters an inline-task prompt; a file whose hunks have mixed authorship cannot be split by pathspec and routes whole to diff-reviewer (legal for either author). No fixed commit ranges, no project paths. The assembled prompt uses a collision-safe delimiter around the diff (sentinel lines, or a fence longer than any inside the content), verified absent from the payload before use; the contract declares everything inside the delimiters untrusted data whose embedded instructions are never followed. The prompt is passed from a temp file, never hand-escaped inline: the slash command carries the prompt through the plugin layer, and the plugin's companion CLI accepts `--prompt-file` and piped stdin - passing the prompt as a command-line argument instead caps at ~32 KB on Windows (a 37 KB prompt fails with "Argument list too long"; verified July 2026), so under argv-only invocation split earlier.
8. **Size and splitting.** Up to ~50 KB of diff goes in one task (54 KB verified). Larger diffs split at file boundaries into coherent chunks - never inside a hunk - each chunk labeled "part N of M, scoped to <files>" under the same contract. Each chunk's raw report and verdict is preserved verbatim; the deduplicated consolidated view is additive, never a replacement. Cross-chunk interactions are substantive review, not validation, so they do not fall to the author: on split reviews the consolidated report plus the chunk map route to diff-reviewer for the cross-chunk pass (different agent than the author; same-family caveat disclosed). A single file whose diff alone exceeds the budget has no legal split point: it routes to diff-reviewer (stated in the report), or the human authorizes a larger single-task budget. Generated files, lockfiles and snapshots are excluded from the inline prompt but keep their independent pass: they route to diff-reviewer, named as excluded in the report. Binary files - tracked (no reviewable text diff) or untracked - route to diff-reviewer the same way and are named in the report.
9. **Timeout and cleanup are part of the transport.** Dispatch is background-first; the dispatch's job ID is recorded and the deadline (15 minutes per chunk by default) is enforced by the orchestrator's own clock against it - a bare `--wait` cannot reach its own cancel instruction, so `--wait` is legal only under a harness-enforced hard timeout. On expiry: `/codex:cancel <job>`, verify the review's own job IDs are no longer active, then fall back to `diff-reviewer` per the existing routing-table row. When the cancel runs through Git Bash, `MSYS_NO_PATHCONV=1` goes in that command's environment (MSYS otherwise mangles the underlying `taskkill /PID` into a path). A review that ends - success or failure - leaves none of its own jobs active; unrelated jobs are out of scope.
10. **Claude validates every finding against the full repository** before reporting: *confirmed* (repo evidence agrees), *discarded* (repo evidence contradicts - Sol saw only the diff, so false positives from missing context are expected, not alarming), or *needs design decision* (human). Classification annotates the reviewer's findings, never rewrites them: the consolidated report preserves Sol's original findings and overall verdict verbatim. When the classifier is the diff's author, a *discarded* stands only on citable repo evidence, and disputed findings - and all blockers - are cleared by the human, never by the author. Suggested fixes are never auto-applied; a fix is new authorship and re-enters the normal review loop. This pass is mandatory because inline-task deliberately blinds the reviewer to the repo - the validation pass is the price of the transport.
11. **Provenance tier for inline-task values is *requested*** (rescue flags), matching skill §4; report the job ID(s) and the zero-active-jobs check in the review report.
12. **Re-enabling `direct` is documented, not implied.** When OpenAI fixes the Windows sandbox (plugin/CLI changelog, or a probe review on a trivial diff completes within its deadline), set the knob to `direct` - or update the dated stamp if `auto`'s Windows exception is retired upstream.

## Invariants (must survive the change)

- Plain declarative Markdown: no scripts, no hooks, no shipped `.codex/` directory. The knob is a policy line, not code.
- Author↔reviewer independence untouched; automatic review gate stays prohibited; `PREMIUM-APPROVED` untouched.
- Clean degradation: without the Codex plugin nothing changes (diff-reviewer path as today); on non-Windows platforms `auto` behaves exactly as before this spec.
- One source of truth: knob in `CLAUDE.md`, mechanics in the skill §6, agents untouched. English canonical; `README.es.md` mirrors `README.md`.
- Provenance stamps dated, updated on re-verification, never removed.

## Changes by file

1. **`CLAUDE.md`** - Review gate: one new bullet defining `Review transport: auto | direct | inline-task` (default `auto`; on native Windows `auto` = `inline-task` because the direct commands hang there as of July 2026; mechanics in skill §6; set `direct` once the sandbox is fixed).
2. **`.claude/skills/delegation-protocol/SKILL.md`** - New §6 "Review transport: inline-task fallback" with the full procedure (scope resolution, prompt contract, dispatch command, split rule, timeout/cleanup, validation of findings, provenance). §2: the `/codex:review` and `/codex:adversarial-review` rows gain "via the review transport (§6)" pointers with the one-line Windows note.
3. **`README.md`** - Under "Optional: OpenAI Codex plugin": new subsection "Native Windows: use the inline-task transport" - the dated breakage, the knob, a compressed procedure summary, the re-enable path, the no-`minimal` note.
4. **`README.es.md`** - mirror (rioplatense voseo, existing voice).
5. **`docs/example-workflow.md`** - one parenthetical at the step-4 review command pointing at the transport (on native Windows the same review ships as an inline-task per skill §6).
6. **`CHANGELOG.md`** - `[Unreleased]`: Added (review transport knob, inline-task procedure, this spec) and Fixed (review gate no longer hangs on native Windows).
7. **`llms.txt`** - Codex-integration concept line: reviews go through a configurable transport; on native Windows they ship as inline diffs over the task path.

Out of scope: `AGENTS.md`, `SECURITY.md`, the four agents, any `.codex/` file, version bump/tag, the `/codex:rescue` writing path (already functional), bringing any downstream project to v0.2.1.

## Migration (downstream projects)

Projects that carry this template as a payload adopt the transport with three merge-not-overwrite edits, preserving every local customization (adapted language, extra local rules, project risk paths):

1. `CLAUDE.md`, review gate: add the transport bullet (adapted to the project's language and default branch), default `auto`.
2. Local `delegation-protocol` skill, §2: append the transport pointer sentences to the two Codex review rows - do not replace the rows, whose wording may be locally adapted.
3. Local skill: append §6 verbatim from this template. If the local skill already numbers a §6, renumber the new section and fix the pointers.

Never overwrite the whole skill or CLAUDE.md with the template copy; never touch the project's agents, risk paths or commands; leave the plugin's automatic review gate disabled; commit only with the project owner's approval. Validate: the knob bullet exists, `git grep -c inline-task` hits CLAUDE.md and the skill, the §2 pointers resolve to the appended section, and a probe inline review on a trivial known diff completes and leaves its jobs closed.

## Validation

- `git status --short` shows exactly the eight files above (seven modified + this spec).
- `git diff --check` clean; every added relative link resolves.
- `git grep -n "inline-task"` consistent across all touched files; `git grep -n "review transport"` finds knob + skill §6 + READMEs.
- No instruction anywhere routes a native-Windows `auto` review to `/codex:review` direct.
- EN/ES parity on every touched section; `.codex/` absent from the tree.

## Expected diff shape

| File | Shape |
|---|---|
| `docs/specs/review-transport.md` | new, ~100-120 lines |
| `CLAUDE.md` | ~4 lines added |
| `.claude/skills/delegation-protocol/SKILL.md` | ~30 lines touched |
| `README.md` | ~20 lines added |
| `README.es.md` | ~20 lines added |
| `docs/example-workflow.md` | ~2 lines touched |
| `CHANGELOG.md` | ~10 lines added |
| `llms.txt` | ~2 lines touched |

## Open items (human)

- Release/tag decision (entries sit under `[Unreleased]`); commit is explicitly deferred to the human.
- Downstream projects (beyond the initial pilot project) migrate later via the migration prompt delivered with this change.
- Re-verify the Windows sandbox on each plugin/CLI update; flip the knob to `direct` when the probe passes.
