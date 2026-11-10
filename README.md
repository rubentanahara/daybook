# Daybook

A desktop app that writes your daily standup and weekly summary from your local git, GitHub, GitHub Projects and Azure DevOps activity, using your own AI. Each generated line links to the commit or ticket it came from.

> **Status: pre-release.** The design is settled and documented. No application code exists yet, and nothing is installable.

## How it will work

- Reads what you did since your last working day (Monday covers Friday): local git across all branches, GitHub repositories and pull requests, GitHub Projects v2, and Azure DevOps.
- You choose the AI: an API key (Anthropic, OpenAI, Gemini, or any OpenAI-compatible URL, including local models), or your own signed-in Claude Code. The Claude Code route is off by default and shows a notice before you turn it on.
- The standup is editable. Copy it or export it as Markdown. The weekly summary rolls up your saved dailies.
- macOS on Apple silicon first. Free and open source.

## What leaves your machine

Daybook has no server of its own and no telemetry. Your history is stored locally. Data leaves your machine only in these ways:

- Prompts to the AI provider you chose, after secret redaction and with a preview before the first send for each repository. Redaction is pattern-based and will miss some secrets.
- Requests to GitHub and Azure DevOps for your own activity, and the sign-in requests to connect your accounts.
- Update checks against GitHub Releases, on by default, with a setting to turn them off.
- Crash reports, only if you opt in.

The full list and the rules behind it are in [ADR-0003](docs/adr/0003-keep-user-data-local-name-the-exceptions.md).

## Documentation

- [Decision ledger](docs/decisions.md): every product and engineering decision.
- [Architecture decision records](docs/adr)
- [Glossary](docs/glossary.md)

## License

[MIT](LICENSE)
