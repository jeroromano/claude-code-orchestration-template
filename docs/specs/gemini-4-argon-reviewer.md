# Spec: Gemini 4 Argon as a reviewer - evaluated, deferred

Status: deferred 2026-10-14 - NOT adopted. Authored on the main thread (Opus 5.5) after the human asked
for a critical evaluation of a proposal - itself drafted by Gemini 4 Argon - to add Argon as a third
external reviewer next to Codex and Grok Build. The human decided to keep Argon out for now and to record
the evaluation for anyone weighing the same integration. Nothing in the routing table, the skill or
CLAUDE.md routes work to Argon; this file records why, and what would have to be true first. The parts of
the proposal that survived the evaluation shipped separately (see "Adopted from this evaluation").

## The proposal, summarized

- Argon through `@google/gemini-cli` with a personal Google OAuth login:
  `gemini -m gemini-4-argon -p "..." --output-format json`.
- Volume-based routing for Claude-authored diffs: over ~25k tokens, over 6 files, or schema / contract /
  security changes -> Argon primary, sent the diff plus the full touched files plus `AGENTS.md`, failing
  over to Codex, then Grok; smaller diffs -> Codex primary, failing over to Argon, then Grok.
- A bash wrapper (`.claude/scripts/adversarial-review.sh`) doing dispatch and failover, with a
  deterministic pre-review gate inside it.
- A fixed BLOCKER / MAJOR / NIT taxonomy, nits applied without a new round, a hard two-round cap.
- diff-reviewer banned for Claude-authored diffs.
- `.claude/settings.json` deny rules for secrets, OAuth session directories and destructive Bash
  commands, plus a `PreToolUse` hook.
- `AGENTS.md` as the single source of truth for conventions and the review contract.

## Facts verified on 2026-10-14

Point-in-time observations - re-verify every one before acting on this file.

- **The model exists.** `gemini-4-argon` rolled out on 2026-10-13 in the Google Antigravity desktop app
  and the Antigravity CLI (`agy`) for Google AI Ultra subscribers; businesses reach it through the Gemini
  API and Gemini Enterprise. Google's model page lists a 1M-token input and output limit.
- **Gemini CLI does not serve it.** The support request (google-gemini/gemini-cli#29689, opened
  2026-10-09) and its PR (#29737) are open with no maintainer reply; the requester notes the CLI is a
  frontend and Google must enable the model server-side.
- **Personal login is gone from Gemini CLI.** Google's Gemini Code Assist page (updated 2026-10-09): the
  consumer tiers are deprecated and "Login with Google" can no longer be used for the IDE extensions or
  Gemini CLI. Gemini CLI's terms page carries a banner: replaced by Antigravity CLI on 2026-06-18 for
  unpaid-tier and Google One users - and Google AI Pro / Ultra fall under the Google One terms there. A
  personal account therefore reaches Argon only through Antigravity.
- **Antigravity's data terms** (Additional Terms of Service; no date shown): Google records
  "Interactions" - user data, usage data, related metadata and feedback - and uses them to evaluate,
  develop and improve Google and Alphabet research, products and services; Google employees and
  contractors may access, view and review them; no retention period is stated - they stay until you
  request deletion (by email); a settings "preference on how such data is used" exists, but the terms do
  not define what it covers (in practice a telemetry toggle; developer-forum threads dispute whether it
  stops training). Those terms do not apply when Antigravity is accessed through Gemini Enterprise
  (Google Cloud) or a Google Workspace subscription.
- **Routes documented as no-training:** the Gemini API paid tier (pricing page: content "not used to
  improve our products", against "used" on the free tier); Gemini Code Assist Standard / Enterprise
  (Google Cloud terms); Vertex AI under Google Cloud terms (a zero-data-retention claim made during the
  evaluation was NOT verified here).
- **Gemini CLI headless,** for when a route exists: `-p` / `--prompt`, `--output-format json`, stdin read
  as context (official automation tutorial). Read-only enforcement in headless mode is not documented
  there; `--yolo` auto-approves every tool call. Antigravity CLI's headless flags are not documented in
  its migration guide.
- **Claude Code facts** behind the rejections below (official docs, CLI 2.1.299): `AGENTS.md` is skipped
  when a `CLAUDE.md` exists, unless the Project instructions setting reads both or `CLAUDE.md` imports it;
  Bash permission rules match the command text and "aren't a security boundary around the program"
  (`/bin/rm`, `bash -c`, `git -C . push` slip past them); `Read` deny rules cover Claude's file tools and
  are not documented to stop a Bash `cat`; `PreToolUse` hooks fire inside subagents too; on Windows hook
  commands run in Git Bash when installed, else PowerShell; the Bash tool's foreground limit is 10
  minutes.

## Assessment

1. **The personal OAuth route is closed** - Gemini CLI no longer accepts consumer logins, and
   Antigravity's consumer terms let Google use interactions to improve its products with human access, no
   retention limit and no defined training opt-out. It fails the Data boundary for any code that must not
   be retained or used for training.
2. **Argon as primary for schema / contract / security changes inverts the order.** Those are risk paths:
   the adversarial pass runs on GPT-6 Astra at the maximum allowed effort. Grok entered the template only
   after a calibration probe against GPT-6.1 Sol on the same diff; an unmeasured model does not start on
   the costliest class.
3. **"Full files to discard false positives" beats only the inline-task transport,** where Codex sees the
   diff alone. On the direct path Codex explores the repository itself, Grok reads the whole committed
   tree, and the orchestrator validates every finding against the repository anyway (skill §6 step 6).
4. **"Discard anything covered outside the hunk" lowers recall silently** - when the model is wrong about
   the coverage, a real bug vanishes. Kept instead, for a future prompt: a candidate dismissed as covered
   elsewhere must cite the file:line of that coverage.
5. **The bash wrapper is not viable.** Reviews run up to 25 minutes plus one extension (a `max` pass took
   16m 27s), against the Bash tool's 10-minute foreground limit, and a chained failover can queue three
   reviews; a non-interactive script cannot ask for the per-run human OK Grok needs on native Windows; it
   would re-implement roughly 20 KB of the skill (preflight, job-ID dispatch, deadline / extension /
   cancel, rollout verification, chunking, Grok's worktree and tool allowlist) as untested shell with
   runtime surface of its own; and it breaks the template's declarative principle (`AGENTS.md`). The
   pre-review gate inside it would also miss Codex-authored diffs, which go to diff-reviewer.
6. **Banning diff-reviewer for Claude-authored diffs deadlocks the gate** whenever no external route is
   available or the code must stay local. Author / reviewer independence is the non-negotiable axis;
   cross-family is a bonus on top (skill §3).
7. **Taxonomy and rounds already exist** (skill §3, §6 steps 2 and 6). Applying nits without review
   breaks "every fix is new authorship"; a hard two-round cap would have escalated v0.6.0, which converged
   after the first pass, two fix rounds and a nit round.
8. **The proposed deny rules break the protocol.** `Read(~/.codex/**)` and `Read(~/.grok/**)` block the
   config preflight and the session-record verification every verdict needs (skill §5, §6 step 5, §7 step
   5) - every verdict would be INVALID - while not protecting anything a Bash `cat` can still read. Deny,
   if anything, the credential files only. Destructive-command denylists are bypassable.
9. **`AGENTS.md` as single source saves nothing.** Claude Code skips it next to a `CLAUDE.md`; moving the
   routing table into `CLAUDE.md` raises the per-session cost (it lives in the on-demand skill today); and
   a reviewer's contract belongs in its prompt, which is deterministic, not in a file each tool may or may
   not load.

## Adopted from this evaluation

Shipped in the same change-set as this file, independent of Argon:

- diff-reviewer's frontmatter description now follows the v0.6.0 fallback order - it still told the
  harness to pick diff-reviewer whenever Codex was unavailable, skipping Grok.
- No review on a red tree: the unit's VALIDATION commands pass before any review is dispatched; a failure
  goes back to the diff's author (skill §3).
- A pass whose only findings are nits counts as clean (skill §3).
- The README tells repositories that already keep an `AGENTS.md` to import it from `CLAUDE.md`.

## Conditions to revisit

1. **A no-training route** for the reviewed code: the Gemini API paid tier, Vertex AI, or Antigravity
   through Gemini Enterprise / Workspace - never a consumer account. Read the terms at that time; they
   change.
2. **A scriptable CLI that serves `gemini-4-argon` on that route,** with its exact command and version
   stamped.
3. **Harness-enforced read-only** in headless mode - not prompt-enforced - plus a session record that
   proves the model, the effort and every tool call (the skill §7 tool-control pattern).
4. **A calibration probe** on the diff GPT-6.1 Sol and Grok already reviewed (`v0.4.0...v0.5.0`),
   comparing recall and precision before any routing role.
5. **The prompt travels by stdin or file,** never argv (the ~32 KB Windows command-line cap, skill §6 step
   2); never `--yolo`.

## Target design, if the conditions are met

- **Role:** the cross-chunk pass of skill §6 step 4 - today diff-reviewer, same family as the author.
  With a 1M-token window Argon could see every chunk plus the full touched files at once, cross-family.
  Not a primary reviewer, not the risk-path pass, never a writer - unless its calibration says otherwise.
- **Auditable discard** in its prompt (assessment point 4).
- **Documentation:** a skill section mirroring §7 (readiness, consent, data boundary, read-only
  verification), Google added to CLAUDE.md's Data boundary, both READMEs, `SECURITY.md`.

## Sources

- Gemini CLI support request: https://github.com/google-gemini/gemini-cli/issues/29689
- Gemini 4 Argon in Google Antigravity: https://antigravity.google/blog/gemini-4-argon-in-google-antigravity/
- Gemini 4 Argon developer guide (Google Cloud): https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/guides/gemini-4-argon
- Gemini Code Assist consumer-tier deprecation: https://developers.google.com/gemini-code-assist/resources/privacy-notice-gemini-code-assist-individuals
- Gemini CLI terms and the Antigravity CLI replacement banner: https://geminicli.com/docs/resources/tos-privacy
- Gemini CLI headless mode: https://geminicli.com/docs/cli/tutorials/automation.md
- Antigravity CLI migration guide: https://antigravity.google/docs/cli/gcli-migration
- Antigravity Additional Terms of Service: https://antigravity.google/terms
- Gemini API pricing (free vs paid data use): https://ai.google.dev/gemini-api/docs/pricing
- Claude Code docs: https://code.claude.com/docs/en/memory, /permissions, /hooks, /sub-agents, /skills
