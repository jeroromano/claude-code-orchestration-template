# Spec: Grok Build as the optional cross-family fallback reviewer (+ the `ultra`-on-direct wording fix)

Status: approved 2026-10-13 (all four flagged points confirmed by the human as written: Grok only on
the standard review; worktree on every platform; main-thread implementation; Sol review of the change,
optional Grok second pass). Authored on the main thread (Opus 5.5) from the
human's decision to adopt Grok Build as a fallback reviewer once a calibration probe passed, plus the
measurements below (Grok Build CLI 1.0.52, plugin grok-build 0.2.1, Windows 11). Does not supersede any
decision of earlier specs; it adds one route to the review fallback chain and fixes one wording gap a
probe finding exposed.

Revised 2026-10-13, same day, after the independent review of the implementation (GPT-6.1 Sol in two
inline-task chunks - rollout-verified `xhigh`, read-only, zero tool and sub-agent calls, Codex CLI
0.162.1 - plus a diff-reviewer cross-chunk pass): 3 majors, 6 minors, 1 nit, no blockers; one minor
discarded on evidence. Folded in: the worktree no longer claims to confine anything (decision 3); Grok's
data scope is the whole committed tree, not the diff (decision 4); the human chose, for native Windows
where the sandbox is not enforced, a per-run OK plus a tool control - offered tools checked against a
measured allowlist at start, every call checked for the read-only flag afterwards, the user's working
tree's `git status` compared with a pre-dispatch snapshot (decision 3); the README "First run" step, `llms.txt` and the
model-unavailable wording now name the full fallback chain; `critique` is "likely" broken and the review
command "documents" no focus text. The probe's tool-call count is corrected from 88 (duplicated event
lines) to 22, and the session-record facts below now name `updates.jsonl`, `tool_definitions.json` and
`signals.json`.

## Context

Today, when the Codex route is unavailable (plugin absent, quota exhausted, model rejected, a review
dispatch that fails or expires), the standard review of a Claude-authored diff falls back to
diff-reviewer - a Claude-family reviewer of a Claude-authored diff. The template already calls that the
weaker guarantee: author/reviewer independence holds, cross-family decorrelation does not. Grok Build is
a third model family with an official Claude Code plugin, so it can keep the fallback cross-family.

Facts verified on 2026-10-13 (point-in-time observations with version stamps):

- **Plugin and CLI.** Official plugin `xai-org/grok-build-plugin-cc` 0.2.1 (marketplace `xai-grok-build`,
  plugin `grok-build`; upstream last push 2026-08-04). Commands: `/grok-build:review|critique|delegate|
  import|runs|show|stop|check`. Hooks: SessionStart/SessionEnd only - no Stop hook, so no automatic review
  loop to guard against. Grok Build CLI 1.0.52, signed in with a grok.com account.
- **The review command is not model-invocable.** `commands/review.md` sets
  `disable-model-invocation: true`, so an orchestrator drives the plugin's bridge script directly -
  `scripts/grok-bridge.mjs review` - the same declared-internal-interface pattern the Codex inline-task
  transport uses with the Codex companion.
- **Bridge interface (plugin 0.2.1, source-read and exercised live).** `review [--wait|--background]
  [--base <ref>] [--scope auto|working-tree|branch] [--model <m>] [--effort low|medium|high] [--cwd <dir>]
  [--json]`. `--background --json` returns a `review-*` `jobId`; `show <job>` prints the result;
  `stop <job>` cancels; `check --json` reports `ready`, CLI, and auth. The review runs Grok's built-in
  `explore` agent headless with `sandbox: "read-only"` and auto-approval inside it.
- **Effort.** The bridge's `--effort` allowlist stops at `high`. The Grok model catalog lists
  `low|medium|high|xhigh` for grok-4.7 (default `high`), and the Grok config key
  `models.default_reasoning_effort` sets the default - so, like the Codex companion, the flag cap is the
  wrapper's, and an omitted flag inherits the resolved config. Inheriting `xhigh` through the bridge is
  NOT verified live.
- **Model.** `--model` hangs headless on CLI 1.0.52 (upstream issue #31): never pass it. The review runs
  on the CLI default model (grok-4.7 on the probe).
- **Run record.** Each run leaves a session record under `~/.grok/sessions/<encoded cwd>/<session-id>/`.
  The job record's `threadId` is that session id (the result also prints it as "Grok session ID").
  `summary.json` carries `current_model_id`, `reasoning_effort`, `sandbox_profile` and `agent_name`;
  `usage.json` carries token counts (and a cost field - never reported, per the no-money rule);
  `tool_definitions.json` lists the tools the run was offered; `updates.jsonl` carries every tool call
  as a `tool_call` update with its `rawInput` and `_meta."x.ai/tool"` (`name`, `kind`, `read_only`);
  `signals.json` summarizes `toolCallCount` and `toolsUsed`; `events.jsonl` has only `tool_started`
  lines by name. This is the Grok equivalent of the Codex rollout check.
- **Read-only is not enforced on native Windows** (upstream issue #24, CLI 1.0.52). Grok's docs say the
  `explore` agent can run shell commands; the bridge's run on the probe was offered none - its
  `tool_definitions.json` listed `read_file`, `list_dir`, `grep`, `web_fetch`, `image_gen`, `image_edit`,
  `image_to_video`, `reference_to_video`, `enter_plan_mode`, `exit_plan_mode`, `ask_user_question`. A
  detached worktree shares the repository's objects and refs and confines no path. The bridge's review
  prompt says "Review only; do not modify files" but carries no untrusted-data contract.
- **Known upstream breakage (0.2.1 / 1.0.52).** `import` is broken (#21); `critique` likely is (#35 -
  `--json-schema` rejected headless; the review path sends no schema); #18 - the SessionStart hook appends
  to the Claude env file on every start/resume/compaction until Bash commands overflow (unmerged fix:
  PR #19); #49 - the hook exports its data dir under the generic `CLAUDE_PLUGIN_DATA`, so other plugins'
  Bash-run scripts (the Codex companion included) inherit grok-build's data dir. The probe ran with the
  fixes for #18 and #49 applied to the installed plugin.
- **Calibration probe (2026-10-13).** Bridge review, `--effort high`, of `v0.4.0...v0.5.0` of this
  repository - the exact diff GPT-6.1 Sol reviewed for v0.5.0 (8 files, ~95 KB of diff, one pass, no
  chunking) - run inside a disposable detached worktree. Completed in ~9 minutes. Session record:
  grok-4.7 (usage attributes grok-4.7-build), effort `high`, `sandbox_profile` read-only, 22 tool calls
  (13 read_file, 5 grep, 4 list_dir - every one flagged read-only, no sub-agent), worktree clean
  afterwards. Result: 3 major +
  1 minor against Sol's 3 major + 7 minor. Four of Sol's ten matched in full and one in part; two of
  Sol's three majors were missed; no false positive; one finding Sol did not report, still present in
  v0.5.1 (decision 6). Read: Grok is a viable cross-family fallback with lower recall than Sol on this
  sample - a fallback, never a replacement.

## Decisions

1. **Route.** Grok Build becomes an OPTIONAL step in the review fallback chain, used only for the
   standard pre-merge review of a Claude-authored diff when the Codex route is unavailable:
   Codex (GPT-6.1 Sol) -> Grok Build, when its plugin is installed and `check` reports ready and the
   code may leave the machine -> diff-reviewer. "Codex route unavailable" covers every case that today
   lands on the §2 fallback row for a standard review: plugin absent, quota exhausted, model rejected
   (§5), a failed or expired dispatch (§6 steps 3 and 5). An INVALID Codex verdict still re-dispatches
   Codex first (§6 step 7) - Grok is not a shortcut around verification.
2. **Out of scope for Grok** (each stays on its current route):
   - The risk-path adversarial pass: the review command documents no focus text and `critique` is likely
     broken (#35), so the pass keeps its §5 chain (Astra -> Sol -> diff-reviewer with the risk-path focus text).
   - Codex-authored diffs: diff-reviewer or a human, unchanged - that route is already cross-family.
   - Writing: no Grok delegate route; rescues still fall back to fast-worker.
   - Complementary diff-reviewer routes inside an inline-task review (cross-chunk pass, oversized or
     binary files, mixed-authorship files) are not fallbacks and do not move.
3. **Mechanics** (new skill §7), fail-closed like §5-§6:
   - **Readiness and consent:** `node <bridge> check --json` must report `ready: true`; anything else ->
     diff-reviewer. On native Windows, where the read-only sandbox is not enforced, the run needs the
     human's explicit OK each time (their risk call - human decision after the review); elsewhere the
     sandbox is Grok's documented behavior, not measured here.
   - **Isolation, with stated limits:** snapshot the user's working-tree `git status`, then run in a
     disposable DETACHED git worktree at the commit under review, never in the working tree - on every
     platform. The worktree holds the tracked files of that commit only (untracked files, gitignored ones
     included, stay out; a tracked file is in, whatever the ignore rules say);
     uncommitted work is committed first by the human's decision, or routes to diff-reviewer. On the base
     branch itself, the base is the ref the change starts from (e.g. the last release tag). A worktree
     separates checkouts and confines nothing; only an enforced sandbox confines the run - the tool
     control below detects violations and stops or invalidates, it does not prevent them.
   - **Tool control:** as soon as the job carries its session id, every tool in the session's
     `tool_definitions.json` must be in the allowlist measured on the probe (the eleven tools listed in
     the facts above); a shell, editing or sub-agent tool stops the job at once -> diff-reviewer. The check runs just after
     the start, not as a barrier (nothing pauses the run), so it limits exposure; the post-run audit still
     covers every call.
   - **Dispatch:** `node <bridge> review --background --base <base> --scope branch --cwd <worktree>
     --json` (+ the effort rule below), capturing the `review-*` `jobId`; no parseable job id = failed
     dispatch -> diff-reviewer. Never `--model` (#31).
   - **Effort:** preflight `models.default_reasoning_effort` in the resolved Grok config: `xhigh` ->
     omit `--effort` (inherits it); unset, lower, or indeterminate -> pin `--effort high`, the bridge's
     cap. Same pin-can-lower logic as §5.
   - **Deadline:** 25 minutes, one extension of the same length only for a job still running, then
     `node <bridge> stop <job>` and diff-reviewer - the §6 step 5 rule.
   - **Verify what ran** (session record named by the job's `threadId`): `sandbox_profile` read-only,
     `reasoning_effort` at least the ordered value, every `tool_call` in `updates.jsonl` allowlisted and
     flagged `read_only: true` (missing = false) with `signals.json` agreeing, an empty
     `git status --porcelain --untracked-files=all` in the worktree, and the user's working tree equal to
     the snapshot. Any miss, or an unreadable record, makes the verdict INVALID -> diff-reviewer (no Grok
     re-dispatch); a call not flagged read-only or a mismatched user `git status` is also an incident
     reported to the human first. These are tripwires, not proof: matching `git status` outputs do not
     show unchanged contents (a further change to an already-modified file, or a write to an ignored one,
     leaves them equal). The model is reported as read from the record; there is no ordered Grok model because
     the bridge cannot pin one.
   - **Findings:** validated and classified per §6 step 6 (confirmed / discarded / needs design
     decision), never auto-applied. Grok reads the real repository, so context-starved false positives
     should be rarer than on a pasted diff - but classification still applies.
   - **Report:** provenance tier *verified* from the session record (model, effort, sandbox profile,
     session id, job id); every call other than `read_file`/`grep`/`list_dir` with its `rawInput` (a
     `web_fetch` URL can carry data out); spend as tokens from `usage.json` - never its cost field; the
     worktree removed and the job no longer running.
   - **Mixed authorship:** the bridge has no path scoping, so Grok sees the whole branch; its pass
     discharges only the Claude-authored portion - the Codex-authored portion still goes to diff-reviewer
     (§3).
4. **Data boundary.** Grok Build runs on xAI infrastructure and reads the repository at the reviewed
   commit, not just the diff: everything committed there can ship to xAI, so Grok is dispatched only
   when that whole tree may leave the machine (revised after the review). Same rules as Codex
   otherwise - secrets, `.env*`, credentials, real-data fixtures never delegated; confidential repos
   default to diff-reviewer and opt in per task.
5. **Interface coupling, declared.** The bridge script is an internal plugin interface (verified against
   plugin 0.2.1, October 2026): re-verify flags, the job payload and the session-record fields on every
   plugin or CLI update. The README states the upstream breakage above with its stamps, the native
   Windows shell caveat, and that a plugin update overwrites any local fix of #18/#49.
6. **G3 fix (from the probe, still present in v0.5.1).** SKILL §5(a) says a resolved `ultra` is replaced
   by "pin the ceiling (... `max` on the raw CLI)", and CLAUDE.md's review bullet says "`ultra` is never
   inherited - pin the ceiling" next to the raw-CLI example - but the only raw-CLI pin the template
   documents, `codex review --commit <sha> -c ...`, reviews ONE commit. State the branch remedy where the
   `ultra` rule lives: on the direct path, a branch review under a resolved `ultra` needs the config set
   to `max` first, then the flagless command. The README already states that remedy next to its example
   and does not change for G3.
7. **Release:** v0.6.0 - a minor bump, per the v0.2.0 precedent (the review-gate routing changes).
8. **Untouched:** the four agents (including diff-reviewer's description - it remains the final
   fallback), AGENTS.md, the CLAUDE.md delegation section, the Codex routes and their efforts.

## Artifacts and expected diff shape

| File | Change |
|---|---|
| `.claude/skills/delegation-protocol/SKILL.md` | §2: pre-merge row "(unavailable -> diff-reviewer, §5)" -> the fallback row; the fallback row split into (a) Codex route unavailable on a Claude-authored standard review -> Grok Build (§7) if installed, ready and allowed to leave the machine, else diff-reviewer, and (b) code that must stay local, or a Codex-authored diff -> diff-reviewer. §5: "a standard review goes to diff-reviewer" -> takes the §2 fallback row; `ultra` sentence gains the direct-path branch remedy (G3). §6 step 3 (two places) and step 5 (one): "diff-reviewer per the §2 fallback row" -> "the §2 fallback row". New §7 with decision 3 (~15-20 lines) |
| `CLAUDE.md` | review-gate bullet: "otherwise the diff-reviewer agent" -> Grok Build if installed and ready (optional, skill §7), else diff-reviewer; line 27 `ultra` remedy (G3); Data boundary: Grok on xAI (~3 lines) |
| `README.md` / `README.es.md` | new section "Optional: Grok Build plugin (cross-family fallback reviewer)" after the Codex section (install, role, mechanics summary, known issues with stamps, Windows caveat); "Without the plugin..." and the model-unavailable sentence name the fallback chain; Security note adds xAI (~25 lines each, EN/ES parity) |
| `llms.txt` | one bullet for the Grok integration (~2 lines) |
| `CHANGELOG.md` | `[Unreleased]` entries (rolled to `[0.6.0]` only on the human's release order) |
| `SECURITY.md` | Data boundary names xAI alongside OpenAI; Grok opt-in per task (~2 lines) - added by amendment, below |
| `docs/example-workflow.md` | review route sentence names the Grok step (~1 line) - added by amendment, below |
| `docs/specs/grok-build-fallback-reviewer.md` | this spec |

Amendment (2026-10-13, human-approved before the review): the implementation pass found two files the
original table missed - `SECURITY.md`, whose data-boundary section named only OpenAI (misleading once
Grok ships code to xAI), and the worked example's review route sentence.

Implementation happens on the main thread: the README prose is the bulk of the work and a delegation
spec precise enough to write it would be as long as the text itself (skill §3, "do not delegate tasks
smaller than their spec").

## Validation

- `git diff --check` clean; `git status --short` shows exactly the files above.
- Every "falls back to diff-reviewer" left in SKILL.md is either the final step of the chain, the
  risk-path pass, a Codex-authored route, or a complementary inline-task route - grep and justify each.
- No live instruction routes the risk-path pass, a Codex-authored diff, or writing to Grok.
- EN/ES README parity on every changed paragraph.
- Leak check before any commit, over the staged diff and the commit and tag messages: no personal paths,
  usernames, drive letters, scratchpad or plugin-cache paths, private repository names or plan details.
  Generic forms only (`~/.grok/...`, `<bridge>`).
- No money figures (the Grok record has a cost field; it is never copied).

## Review of this change

Doc-only, so self-merge is legal under §3 - but this is review-gate policy, the high-stakes case §3
says to review voluntarily. Proposed: Claude-authored -> GPT-6.1 Sol via inline-task (the standard
route), rollout-verified. Optional second pass: Grok through the new §7 mechanics on the same delta, as
their first live run.

## Open items

- Whether the per-call `read_only` flag is set to false for a shell or editing call (no such tool was
  offered on the probe) - the allowlist check stops a run that is offered one, but only once the check
  runs; it is a detection-and-stop tripwire, not a guarantee that no such call happens first.
- Whether an omitted `--effort` actually inherits `xhigh` from `models.default_reasoning_effort` through
  the bridge - verify from the session record on the first run configured that way.
- The bridge's `review` handler forwards positional text as "User focus" (source-read, plugin 0.2.1),
  although the command documents no focus text. If verified live, it could carry the risk-path focus
  text and an untrusted-data / no-shell contract - a later spec, not this one.
- Upstream #24 (read-only on Windows), #31 (`--model` headless), #35 (`critique`), #18/#49 (hook): re-check
  on every plugin or CLI update; a fix to #31 would allow pinning the model.
- Timing: one sample (~9 min for ~95 KB with 22 tool calls). The 25-minute deadline is borrowed from §6,
  not measured for Grok at the upper end.
