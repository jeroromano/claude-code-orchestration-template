# Spec: Review effort ceilings (max via direct, xhigh via inline-task)

Status: approved 2026-08-09. Authored on the main thread (Fable) from the human's directive for this
session - reviews may now run at `max` on the direct (sandboxed) path and up to `xhigh` (pinned) on the
inline-task path - plus measurements taken 2026-08-08/09 (plugin 1.0.6, Codex CLI 0.144.x, Windows 11). Supersedes the effort ladder of `gpt-5-6-sol-routing.md` (decision 2)
and the v0.2.1 attribution of the `max` limitation to "the documented TOML values".

Revised 2026-08-09, same day, after the independent audit of the released delta: GPT-5.6 Sol in two
inline-task chunks, both at rollout-verified `max` (inherited from the resolved config - the review
demonstrated decision 1's inheritance rule with its own run), plus a diff-reviewer cross-chunk pass;
1 blocker + 6 majors + 1 minor + 2 concerns + 1 nit (the nit - a conservative size estimate in the
expected-diff-shape table below - needed no fix), all classified. Fixes folded in: the context paragraph
recording the session premise no longer states a billing regime, and the validation criterion is
scoped to lines the delta adds (the blocker; pre-existing pool-mechanism wording classified out of
scope); "the only path exposing `max`" corrected to "the only path that can PIN `max` per invocation"
in decisions 1/5 and every artifact (inline-task demonstrably reaches `max` by inheritance); the
raw-CLI `--commit` form is labeled single-commit and excluded as a substitute for branch-level review
across its full five-file footprint (the cross-chunk pass established the footprint was wider than
either chunk saw); preflight defines unset/indeterminate config as BELOW the target (decision 2);
rollout verification gains per-path session-id correlation - an uncorrelatable direct review is
unverifiable, hence invalid - and a non-read-only permission profile now also invalidates (decision
3); decision 6's authority is narrowed to APPROVED designs/specs, keeping audit findings inside the
finding-validation flow; transport wording in the READMEs is scoped to delivery selection; and the
CHANGELOG citation for the configured-tier fix no longer implies the READMEs use provenance-tier
vocabulary. The audit run itself also discharged this spec's first open item (the voluntary
independent pass doubling as the live companion-dispatch probe): dispatch returned capturable
`task-*` IDs, both chunks (47.6 KB and 30.2 KB prompts) completed in 8m 56s and 5m 50s - inside the
25-minute provisional deadline - and both rollouts were read and proved `max` on a read-only profile.

## Context

The template's effort ladder said: Sol reviews at `high`, `xhigh` on risk paths, `max` only as an
exceptional human-authorized escalation "when the invocation path exposes it (the current plugin path
does not)". Three of those clauses have now been measured false or obsolete:

- **`max` is the real top of the effort enum, and the TOML accepts it.** The API rejects an invalid
  `model_reasoning_effort` by listing the supported values: `none | minimal | low | medium | high |
  xhigh | max`. There is nothing above `max` to escalate to, so "escalate on risk paths" has no rung
  left once a review already runs at the ceiling.
- **The Codex CLI does not validate the value locally.** A typo (verified with `"ultramax"`) passes
  parsing, prints verbatim in the run header, and dies mid-run at the API with HTTP 400 - after the
  review has already burned its startup. Treat the TOML value as API-validated only.
- **The ceiling is per-path, set by the wrapper, not by Codex:**
  - The plugin's companion `task` runtime (the inline-task transport) accepts `--effort` up to
    `xhigh` and rejects `max` with "Unsupported reasoning effort" (measured 2026-08-09, plugin 1.0.6).
    An *omitted* `--effort`, however, inherits the resolved Codex config - which may be `max`, and the
    thread then really runs at `max` (rollout-verified, see below).
  - The direct path exposes `max` fully: the raw CLI accepts a per-invocation override -
    `codex review --commit <sha> -c model_reasoning_effort="max"` (verified 2026-08-08) - and the
    plugin's `/codex:review` (still flagless) inherits whatever the resolved config pins.
- **Pinning is exact selection, not a raise.** Passing `--effort` (or `-c`) overrides the resolved
  config and can therefore LOWER a machine configured at `max` down to the pinned value. Measured
  consequence: a three-round review gate pinned to `high` returned "may merge" on a branch that a
  later unpinned pass (inheriting `max`) rejected with a blocker plus majors - an observed
  association, not a controlled comparison (different diffs, different prompts, one sample), but the
  direction of the mistake is structural: the pin was believed to raise and it capped.
- **Effort is verifiable after the fact.** The CLI does not echo the runtime effort on stdout, but it
  records it: `~/.codex/sessions/<YYYY>/<MM>/<DD>/rollout-*-<session-id>.jsonl` carries a
  `thread_settings_applied` event with the thread's real `reasoning_effort`, `model` and
  `permission_profile` (the last also evidences the read-only sandbox: `file_system` restricted /
  `access: read`, `network: restricted`). The session id comes from the job result. Checked with a
  control: the extractor returns `high` for a dispatch pinned `--effort high` and `max` for one
  dispatched without the flag on a `max`-configured machine - it discriminates, it does not echo the
  config.
- **Cost is real.** Measured 2026-08-09: a `max` review took 16m 27s on a 16 KB diff, against
  3m 40s / 2m 25s / 37s for three `high` passes on a 12 KB one. Different inputs - an association,
  not a controlled comparison - but the old 15-minute chunk deadline would have cancelled the `max`
  pass at 15m 00s and discarded the blocker it found.
- **The native-Windows sandbox breakage has a verified per-machine fix** (2026-08-08): install
  PowerShell 7 outside the `WindowsApps` store path (the Store shim breaks process spawning inside
  the sandbox) and restart the plugin's app-server process, which caches the PATH it was launched
  with. After the fix, `direct` reviews complete on Windows 11 - and the reviewer inspects the real
  repository (runs `git diff`, greps consumers, executes the suite) instead of judging a pasted diff,
  which removes the context-starved false positives that inline-task's validation pass exists to
  classify.

Separately, the human directed that escalation to the top tier be treated as a stable, first-class
rung of the routing. Per this repo's rot rule, billing-regime statements (plan inclusion, rates,
quotas) stay banned from the artifacts - a regime is exactly what time makes false - so nothing gains
a regime claim; the consequence is structural instead: the template now states how a top-tier design
binds the cheaper models that implement it.

## Decisions

1. **Reviews run at the highest effort their invocation path exposes.** `max` on the direct path;
   `xhigh` as the pinnable ceiling on inline-task (an omitted `--effort` may inherit higher from the
   resolved config - that is legal and rollout-verifiable, not a violation). `high` remains the floor:
   pin it (or `xhigh`) only to raise a machine whose resolved config sits BELOW the target - never
   "pin the ceiling" on a machine already configured at it, because a pin is an exact selection and
   can lower. The risk-path adversarial pass runs at the same maximum - there is no higher rung, so
   the old "raise to xhigh, then set it back" lifecycle is deleted (a restore step that survives on
   memory is a bug factory). Writing effort is unchanged: `medium`, `high` only for hard bugs or
   multi-module work. `max` stops being "exceptional, human-authorized": the human authorization now
   lives in adopting this policy, not in each invocation.
2. **Preflight before dispatch.** Read the resolved Codex config (repo-local `.codex/config.toml`
   only when it exists and the repo is trusted; otherwise the global `~/.codex/config.toml`) before
   dispatching a review. At-or-above target: omit `--effort`. Below target: pin the path's ceiling as
   a floor. An unset or indeterminate value counts as below - pin; it is never "satisfied by unknown".
   Never report "CLI default (unknown)" while a global config exists - name the file that actually
   resolved and its values.
3. **Verify the rollout for any verdict you act on.** Read the session rollout
   (`thread_settings_applied`) and report model/effort as *verified*, naming the session id. Correlate
   the session id from the invocation's own result (companion jobs name it in the job result -
   verified 2026-08-09); a direct review whose session id cannot be correlated is unverifiable under
   the same rule - never resolved by picking the newest rollout, which is unsafe under concurrent
   sessions. A rollout showing an effort below what the policy ordered, OR a permission profile that
   is not read-only, OR a rollout you could not read, makes the verdict INVALID: say so, do not
   classify its findings, re-dispatch. "We could not check" and "it was fine" must not look the same.
   The *requested*/*configured* tiers survive only for reports nobody acts on (status notes, aborted
   runs).
4. **Deadline: 25 minutes per chunk, PROVISIONAL, one extension.** Raised from 15 because the only
   `max` timings on record (16m 27s, 4m 29s) straddle the old default. Provisional because both sit
   near 16 KB - a third of the ~50 KB chunk budget - and differ 3.7x between themselves, so size does
   not predict runtime; re-measure at the upper end before treating 25 as a bound. On expiry: exactly
   ONE extension of the same length, granted only to a job still reported `running`; then cancel and
   fall back per the routing table. One extension, never two - "still running" is also what a hung job
   looks like. Raise the deadline whenever you raise the effort: they are one setting.
5. **Transport knob semantics unchanged; `direct` gains two reasons to exist.** `auto` still resolves
   to `inline-task` on native Windows (the breakage is the default state of a fresh machine; the fix
   is per-machine). The re-enable path is unchanged (probe review completes within deadline -> set
   `direct`) but now documents the verified fix and what `direct` buys: the only path that can pin
   `max` per invocation, and a reviewer that verifies its own hypotheses against the repo before
   reporting them.
6. **Top-tier design authority.** New model-routing rule in `CLAUDE.md`: an APPROVED design or spec
   authored by the escalated top tier is AUTHORITATIVE for the cheaper model that implements it -
   implemented to the letter, each point a checklist item. Prohibited: substituting a specified check
   with "implicit equivalence" reasoning, silently simplifying to an easier variant, or documenting
   the designed variant while implementing another. If a point seems wrong or unnecessary, flag the
   deviation to the human BEFORE deviating. Only overrides: an independent review catches a concrete
   error (repro or file:line), or an explicit human decision. Audit and review FINDINGS are not
   specs: they remain inside the finding-validation flow (skill §6) and are never auto-applied -
   authority binds implementation of approved designs, never the acceptance of unvalidated critiques.
   This is the existing "spec > code" hierarchy applied across model tiers, and it is the
   template-shaped consequence of the top tier being a stable rung.
7. **No regime claims.** Nothing in the template asserts Fable's (or any model's) plan inclusion,
   rates or quotas. The premium-reasoner gate, runtime disclosure and `/usage` verification stand
   unchanged.

## Invariants (must survive the change)

- Plain declarative Markdown: no scripts, no hooks, no shipped `.codex/` directory.
- Author-reviewer independence untouched; automatic review gate stays prohibited; `PREMIUM-APPROVED`
  untouched; transport never changes who reviews.
- Clean degradation: no plugin -> diff-reviewer, exactly as today; machines without Sol keep the
  documented model degradation.
- Rot rule: regimes and prices stay out; provenance stamps (dates, versions) are updated on
  re-verification, never removed.
- One source of truth: policy in `CLAUDE.md`, mechanics in the skill, agents project-agnostic.
  English canonical; `README.es.md` mirrors `README.md`.

## Changes by file

1. **`CLAUDE.md`** - Model routing: add the top-tier design-authority rule (decision 6). Review gate:
   replace the Sol effort bullet (high/xhigh/config-lifecycle) with the per-path ceilings + preflight
   + rollout-verification summary (decisions 1-3), and extend the transport bullet: the sandbox fix
   exists (PS7 outside WindowsApps + app-server restart, 2026-08-08), `direct` is the only path
   exposing `max`.
2. **`.claude/skills/delegation-protocol/SKILL.md`** - §2: rewrite the two Codex review rows (per-path
   ceilings; preflight; the flagless `/codex:review` inherits the RESOLVED config - repo file only
   when present, else global). §4: fix the *configured* tier (name the file that resolved; "CLI
   default (unknown)" only when neither file sets a value) and add the requested->verified promotion
   via rollout. §5: replace the effort guard with decisions 1-3 (ceilings, pin-lowers warning,
   preflight, invalid-verdict rule); keep every other guard. §6 step 3: omit `--effort` by default
   (inherit), pin only as floor; note the companion's flag cap. §6 step 5: 25-minute provisional
   deadline, one-extension rule, rollout verification with the sessions path. §6 step 7: verified
   provenance mandatory for acted-on verdicts; invalid-verdict consequence.
3. **`README.md`** - Sol routing section: new ladder (per-path ceilings), the enum/no-local-validation
   caveat, the companion flag cap, config snippet updated (`max` pin + trusted-repo note + typo
   warning; delete the "raise to xhigh then set it back" lifecycle), the raw-CLI per-invocation form.
   Windows section: the verified sandbox fix, what `direct` buys, probe-then-flip unchanged.
   Interface-coupling stamp: re-verified August 2026 (companion effort-flag cap measured).
4. **`README.es.md`** - mirror; sync stamp to agosto 2026.
5. **`llms.txt`** - Codex-integration concept line: per-path effort ceilings, one sentence.
6. **`CHANGELOG.md`** - `[Unreleased]`: Changed (effort ladder -> per-path ceilings; deadline 15->25
   provisional + one-extension; configured-tier resolution fix) / Added (rollout verification,
   requested->verified promotion, invalid-verdict rule; design-authority rule; Windows sandbox fix
   documentation; this spec).

Out of scope: the four agents (premium-reasoner keeps its generic, regime-free wording - decision 7),
`AGENTS.md`, `SECURITY.md`, `docs/example-workflow.md` (its review step names no effort), any
`.codex/` file, version bump/tag, downstream migrations (projects carrying this template as a payload
adopt these changes separately).

## Validation

- `git status --short` shows exactly the seven files above (six modified + this spec).
- `git diff --check` clean; every added relative link resolves.
- `git grep -n "xhigh"` / `git grep -n '"max"'` consistent: no surviving claim that the plugin path
  stops at `xhigh` unqualified, no surviving "raise then set back" lifecycle, no per-invocation
  human-authorization requirement for `max`.
- No regime claim (plan inclusion, rates) in any line this delta adds; pre-existing pool-mechanism
  wording in unchanged lines is out of this delta's scope (classified 2026-08-09).
- EN/ES parity on every touched section; `.codex/` absent from the tree.

## Expected diff shape

| File | Shape |
|---|---|
| `docs/specs/review-effort-ceilings.md` | new, ~150 lines |
| `CLAUDE.md` | ~10 lines touched |
| `.claude/skills/delegation-protocol/SKILL.md` | ~40 lines touched |
| `README.md` | ~30 lines touched |
| `README.es.md` | ~30 lines touched |
| `llms.txt` | ~2 lines touched |
| `CHANGELOG.md` | ~15 lines added |

## Open items (human)

- Independent pass on this diff (instruction docs: below the mandatory line, high-stakes - the
  voluntary pass §3 recommends). Doubles as the pending live probe of the companion dispatch.
- Release/tag decision (entries sit under `[Unreleased]`).
- Re-measure the 25-minute deadline near the 50 KB chunk ceiling before relying on it.
