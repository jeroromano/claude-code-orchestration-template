# Security Policy

This template ships no executable code - only Markdown instruction files. It has no runtime, no dependencies, and makes no network calls of its own, so its "attack surface" is what it instructs an AI agent to do. Two areas deserve care.

## Data boundary (optional Codex integration)

If you enable the optional [OpenAI Codex](https://github.com/openai/codex-plugin-cc) integration, every delegated task or review **ships the touched code to OpenAI's infrastructure**. Never delegate secrets, `.env*` files, credentials, customer data, or real-data fixtures. For confidential or client repositories, keep review local (the `diff-reviewer` agent) and make Codex opt-in per task. See the **Data boundary** section in `CLAUDE.md` and the **Security note** in `README.md`.

## Spend safety

The `premium-reasoner` agent is a billing boundary. It refuses to run without an explicit `PREMIUM-APPROVED` token and pins its cost profile in frontmatter. Do not remove these guards, and do not enable the Codex plugin's automatic review gate by default - it registers a Stop hook that can loop models against each other and drain quota pools. See **The spend gate** in `README.md`.

## Reporting

This is an experimental community template with no warranty. If you find an unsafe default or an error in the guidance, please open an issue - or, for anything sensitive, a private security advisory via the repository's **Security** tab. There is no formal disclosure timeline.
