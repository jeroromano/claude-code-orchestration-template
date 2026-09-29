# Spec: Codex GPT-6 refresh (GPT-6.1 Sol default, GPT-6 Astra on risk paths, `ultra` excluded)

Status: approved 2026-09-29. Authored on the main thread (Fable) from the human's routing decision for
this session, plus measurements taken the same day (Codex CLI 0.157.0 installed, 0.159.1 current;
plugin 1.0.6; Windows 11). Supersedes the model slug of `gpt-5-6-sol-routing.md` (decisions on the
rescue pin and the recommended TOML) and two facts of `review-effort-ceilings.md`: "`max` is the top of
the effort enum" and "the rollout carries a `thread_settings_applied` event". Every other decision of
those specs stands.

Revised 2026-09-29, same day, after the independent review of the released v0.5.0 (GPT-6.1 Sol in
three inline-task chunks - rollout-verified `xhigh`, read-only, zero tool and sub-agent calls, on
Codex CLI 0.159.1 - plus a diff-reviewer cross-chunk pass): 3 majors, 7 minors, no blockers, all
classified against the repository. Human decisions taken on the findings: when GPT-6.1 Sol is
unavailable, work falls back to the LOCAL agents - never to a rerun without `--model`, whose CLI
default is whatever the resolved config names and may be Astra (decision 4 rewritten); Astra reviews
risk paths and never implements (interpretation confirmed, now stated in CLAUDE.md and the skill);
writing effort stays `medium`/`high` (interpretation confirmed). Also folded in: the pin rule gains
its `ultra` exception everywhere it is stated; the `thread_settings_applied` claim is stated per
measured version instead of as a version cutoff; the price comparison below is removed; the Astra
effort claims trace to sources; the README scopes config inheritance to the direct commands. The
review itself doubled as this spec's first open item (the live probe on CLI 0.159.x), which passed.

## Context

The human's decision: GPT-6.1 Sol becomes the default Codex model; GPT-6 Astra is reserved for the
risk paths where extra compute and reasoning change the outcome (security, payments, tenant
isolation and similar). Reason: cost on the Codex limits - Astra at `xhigh`/`max` is the costlier
tier and is not for routine use. This spec records that call; it states no rates, quotas or plan
regimes (the rot rule).

Facts verified on 2026-09-29 before designing this (point-in-time observations; every one carries its
version stamp so a reader can judge staleness):

- **The Codex plugin did not change.** `openai/codex-plugin-cc` is still at v1.0.6 (last push
  2026-07-08, tag and release list checked against GitHub). The session premise of a new plugin
  version was checked and is false; every plugin-level statement in the template (companion flags,
  job payload, `--effort` allowlist) still describes 1.0.6.
- **GPT-6.1 Sol requires Codex CLI 0.159.0 or newer.** `gpt-6.1-sol` was released 2026-09-29;
  third-party integration reports agree that the ChatGPT backend serves it only to Codex clients
  0.159.0 or newer (0.156.1 and 0.158.0 fail with the same account). The locally installed 0.157.0's
  model catalog does not list it. Current CLI: 0.159.1.
- **The Codex model catalog now spans two generations.** Catalog fetched 2026-09-29 (client 0.157.0):
  `gpt-6-astra` ("Frontier intelligence for the most demanding work"), `gpt-6-sol` ("Previous
  generation workhorse model"), `gpt-6-luna`, and the GPT-5.6 family ("Older generation"). OpenAI's
  model docs list `gpt-6.1-sol` with efforts `low|medium|high|xhigh|max` (no `none`/`minimal`) and
  position it near Astra in capability; OpenAI's GPT-6 guide states that GPT-6 Astra and GPT-6.1 Sol
  do not support `none` (it is silent on Astra and `minimal`). The catalog lists `xhigh` and `max` for
  Astra, and Astra threads ran at both on this machine (rollout census below).
- **`ultra` is a new effort value above `max`, and it IS Codex Ultra.** The catalog advertises
  `low|medium|high|xhigh|max|ultra` for the GPT-6 and GPT-5.6 Sol models; `ultra` is described as
  "Maximum reasoning with automatic task delegation". Rollout census (one machine, CLI 0.144.5-0.157.0):
  every `ultra` thread carries the developer message "Proactive multi-agent delegation is active" and
  the 48 `ultra` rollouts made sub-agent calls (`spawn_agent`, `wait_agent`, `send_message`, ...); the
  330 all-`max` rollouts carry "Do not spawn sub-agents unless the user or applicable AGENTS.md/skill
  instructions explicitly ask" and made none. The template's no-Ultra rule therefore has a concrete
  key now: `model_reasoning_effort = "ultra"`. `max` stops being "the top of the enum" and becomes the
  highest ALLOWED effort.
- **The rollout event the verification step names is gone.** `thread_settings_applied` appears in
  669 of 885 rollouts written by CLI 0.144.5, 13 of 314 by 0.154.0, and none of those written by
  0.153.4 or 0.157.0. The `turn_context` event is present across all measured versions and carries
  `model`, `effort`, `sandbox_policy` and `permission_profile` (e.g. `sandbox_policy.type: read-only`,
  `file_system: restricted` with `access: read`, `network: restricted`). Plugin-originated rollouts
  never mixed model or effort within one file (1146 files). As written, skill §6 step 5 would find
  nothing on a current CLI and - failing closed - invalidate every verdict.
- **The rollout path is unchanged but its future is not.** Rollouts still land at
  `~/.codex/sessions/<YYYY>/<MM>/<DD>/rollout-*-<session-id>.jsonl` under 0.157.0, but the CLI now
  ships `codex migrate-rollouts` ("migrate legacy local sessions to paginated thread history"), so the
  storage may move in a later version.
- **`review_model` overrides `model` for reviews.** Codex config reference: "Optional model override
  used by /review (defaults to the current session model)". A config that pins `model` alone does not
  control the reviewer when `review_model` is set elsewhere.
- **The companion still cannot express `max` or `ultra`.** Plugin 1.0.6 source: `--effort` allowlist
  `none|minimal|low|medium|high|xhigh`. An omitted `--effort` inherits the resolved config - which may
  now be `ultra`. The plugin's review commands advertise no model flag (their companion handler parses
  an undocumented `--model`/`-m`; source-read only, not a public interface, not verified live).
- **`codex review` (0.159.1) lists `--base <BRANCH>`** alongside `--commit`, `--uncommitted` and `-c`.
  A branch-level raw-CLI review with per-invocation pins may therefore exist; not verified live, so
  this spec does not route through it (open item).
- **Claude side:** the agents pin family aliases (`sonnet`, `opus`, `fable`), which the harness
  resolves to the current releases (Sonnet 5.5, Opus 5.5, Fable 5.1). No frontmatter change is needed.

## Decisions

1. **Default Codex model: `gpt-6.1-sol`.** Replaces `gpt-5.6-sol` wherever the template pins a model:
   the rescue (writing) command, the inline-task dispatch for standard reviews, and the recommended
   TOML. Efforts are unchanged: reviews at the per-path ceiling (`max` via direct, `xhigh` pinnable via
   inline-task - the human's "Sol 6.1 at xhigh or max"); writing at `medium`, `high` only for hard bugs
   or multi-module work.
2. **GPT-6 Astra runs the risk-path adversarial pass** of Claude-authored diffs, at the same per-path
   ceiling. Standard reviews and all writing stay on Sol 6.1. Astra is never the default.
3. **The model joins the preflight -> dispatch -> verify trio** (skill §5), because Astra-on-risk-paths
   makes the model a per-pass selection, not a machine constant:
   - Preflight the resolved `model` - and `review_model` for the built-in reviewer - alongside effort.
   - Dispatch: the inline-task path pins per invocation (`--model gpt-6.1-sol` or `--model
     gpt-6-astra`). The plugin's slash review commands expose no model flag: they run on the resolved
     config, so when it does not name the ordered model, either ship the pass through the §6
     inline-task mechanics with the right `--model`, or ask the human to point the resolved config at
     it for that run.
   - Verify: the rollout must show the ordered model (or a declared degradation, decision 4).
4. **Degradations, disclosed:** Astra unavailable on the account -> run the risk-path pass on Sol 6.1
   at the same ceiling and state "Astra unavailable - degraded to GPT-6.1 Sol" in the report (the human
   may still block the merge on it). `gpt-6.1-sol` rejected (CLI older than 0.159.0, or no account
   access) -> the LOCAL fallback: a rescue goes to fast-worker, a standard review to diff-reviewer, and a
   risk-path pass already degraded from Astra goes to diff-reviewer with the risk-path focus text; the
   report says which, and names a CLI below 0.159.0 as the cause when it is. Never rerun without
   `--model`: the CLI default is whatever the resolved config names - Astra on a machine configured for
   it - so that rerun could make Astra implement, or produce a review verification must reject
   (revised after the v0.5.0 review; the original text kept the v0.2.1 "rerun without `--model`" rule).
   The "limited-access preview" wording for Sol is retired: the gate is now the CLI floor plus account
   access.
5. **`ultra` is excluded as Codex Ultra.** The no-Ultra rule names it explicitly. Consequences:
   - Every "top of the enum" / "no higher rung" claim becomes "`max` is the highest allowed effort".
   - Preflight: a resolved `ultra` is never inherited - it counts as out of policy, like a value below
     the floor. Inline-task pins `xhigh` (the companion cannot express `max`) unless the human lowers
     the config to `max`; the flagless slash commands cannot run under it until the config changes.
   - Never pin `ultra` anywhere (`--effort` - the companion rejects it anyway - or `-c`).
   - Rollout: an `ultra` effort, or any sub-agent call (`spawn_agent` and its siblings) in a review
     rollout, invalidates the verdict - same handling as a below-policy effort.
6. **Rollout verification reads `turn_context`.** Every `turn_context` event in the session rollout
   must show the ordered model (or the declared degradation), the ordered effort and a read-only
   profile; `thread_settings_applied` is named only as the event older CLIs wrote (0.144.x, and a few
   0.154.0 rollouts). An
   unreadable rollout still invalidates (fail closed). Rollout storage is re-checked on every CLI
   update (`migrate-rollouts` exists).
7. **`review_model` in the recommended TOML.** The README snippet pins `model`, `review_model` and
   `model_reasoning_effort = "max"`; the *configured* provenance tier names `review_model` when it is
   set.
8. **Version stamps move only where re-verified.** CLI floor 0.159.0 for Sol 6.1; plugin stays 1.0.6
   (re-checked upstream 2026-09-29). The July 2026 Windows-sandbox stamps (plugin 1.0.6 / CLI 0.144.0)
   stay as they are - provenance, not re-verified here. The measured `max` timings (GPT-5.6 Sol,
   2026-08-09) stay as dated history.
9. **Claude side: no instruction changes.** Aliases already resolve to Fable 5.1 / Opus 5.5 /
   Sonnet 5.5; only the worked example's sample provenance line is refreshed from a live probe.

## Interpretations flagged to the human (both confirmed 2026-09-29, after the v0.5.0 review)

- "Sol 6.1 at xhigh or max" is read as the review ceilings - identical to the existing per-path policy.
  Writing effort stays `medium`/`high`.
- Astra's scope is the risk-path adversarial *review*. A risk-path *implementation* delegated to Codex
  stays on Sol 6.1 (and is then reviewed by diff-reviewer or a human, never Codex).
- The CLAUDE.md risk-path list is untouched: the human's examples are project-specific and belong in
  each project's own list.

## Artifacts and expected diff shape

| File | Change |
|---|---|
| `CLAUDE.md` | delegation bullet slug/degradation; Astra on the risk-path pass; `ultra` named in the no-Ultra rule; "no higher rung" reworded (~6 lines) |
| `.claude/skills/delegation-protocol/SKILL.md` | §2 rescue/risk-path rows; §5 model trio, `ultra`, degradations; §6 steps 3/5/7 (`--model`, `turn_context`, invalidation) (~25 lines) |
| `README.md` / `README.es.md` | Sol section (title, roles, command, availability, TOML with `review_model`, `ultra`), interface-coupling stamp, spend-gate provenance line (~20 lines each) |
| `llms.txt` | Codex integration line (~2 lines) |
| `docs/example-workflow.md` | rescue slug, availability sentence, sample provenance line (~3 lines) |
| `CHANGELOG.md` | `[Unreleased]` entries |
| `docs/specs/codex-gpt-6-refresh.md` | this spec |

Historical specs are not edited: this spec records what it supersedes.

## Validation

- `git diff --check` clean; `git status --short` shows exactly the files above.
- No `gpt-5.6-sol` left outside `CHANGELOG.md` history and `docs/specs/` (historical specs keep theirs).
- No live instruction still calls `max` the top of the enum, or names `thread_settings_applied` as the
  event to read.
- EN/ES README parity on every changed paragraph.
- No private repository names, paths or plan details in the diff (grep before any commit).

## Open items

- ~~Live probe on CLI 0.159.x~~ - DONE 2026-09-29 as the independent review of v0.5.0: companion
  `task` dispatch with `--model gpt-6.1-sol --effort xhigh` returned capturable `task-*` IDs, each job
  result named its thread (session) id, and the three rollouts showed `turn_context` with
  `gpt-6.1-sol` / `xhigh` / read-only and no `thread_settings_applied` event; 3m 56s - 5m 40s on
  31-53 KB prompts. Node now prints a DEP0190 deprecation warning on the companion's child-process
  spawn - harmless, to watch on the next plugin release.
- Whether `codex review --base <base>` accepts a custom prompt and `-c` pins together - if so, it is a
  branch-level direct review with per-invocation model/effort pins, retiring the single-commit caveat
  and the direct-path model gap of decision 3.
- The undocumented `--model` on the companion's review handler (plugin 1.0.6): verify live before any
  instruction relies on it.
- The 25-minute inline-task deadline was measured on GPT-5.6 Sol; re-measure for Astra at `xhigh`.
