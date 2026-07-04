# Changelog

Notable changes to this template. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions are annotated git tags following [Semantic Versioning](https://semver.org/).

## [Unreleased]

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

[Unreleased]: https://github.com/jeroromano/claude-code-orchestration-template/compare/v0.1.3...HEAD
[0.1.3]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.3
[0.1.2]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.2
[0.1.1]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.1
[0.1.0]: https://github.com/jeroromano/claude-code-orchestration-template/releases/tag/v0.1.0
