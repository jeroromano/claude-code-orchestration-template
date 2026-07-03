# Contributing

Thanks for your interest. This is a small, opinionated template, so contributions work best when they respect its constraints.

**Before you start,** read [`README.md`](README.md) for what the project is, and [`AGENTS.md`](AGENTS.md#working-rules) for the hard constraints - they apply to human contributors and AI agents alike (declarative-only, one source of truth in `CLAUDE.md`, keep the READMEs in sync, and never weaken the spend or review gates).

## Translations

English is canonical; `README.md` wins on any discrepancy. A change to `README.md` that alters meaning must be mirrored in [`README.es.md`](README.es.md). New-language translations are welcome as `README.<lang>.md`. The instruction files (`CLAUDE.md`, agents, skill) stay in English by design and are not translated.

## Proposing a change

1. Open an issue describing the problem before large changes.
2. Keep the diff scoped and self-explanatory; match the existing voice.
3. Add a note under `CHANGELOG.md`'s **Unreleased** heading if your change is user-visible.
4. Sanity-check: `git status --short`, `git diff --check`, and confirm any new Markdown links resolve to real files.

By contributing, you agree that your work is licensed under the repository's [MIT License](LICENSE).
