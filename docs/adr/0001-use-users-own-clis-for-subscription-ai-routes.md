# Use the user's own installed CLIs for subscription AI routes; the app never handles subscription credentials

* Status: accepted
* Deciders: rubentanahara
* Date: 2026-11-09

## Context and Problem Statement

Daybook is a macOS-first Electron app that writes a developer's daily standup from local git, GitHub, GitHub Projects v2 and Azure DevOps, using the user's own AI. API-key providers (Anthropic, OpenAI, Gemini, OpenAI-compatible URL) are one route. This ADR covers the other route: how Daybook reaches a model through a user's existing subscription without taking custody of credentials, and which CLIs can be driven safely.

## Decision Drivers

* Anthropic's legal page: "Anthropic does not permit third-party developers to offer Claude.ai login into their own applications", nor to route requests through plan credentials. It "does not prevent an end user from signing in to the unmodified Claude Code binary with their own Claude subscription". It also says products using the Agent SDK should use API key authentication. The headless page says `claude -p` is the Agent SDK via the CLI. No page addresses an app that spawns the user's own `claude -p`.
* OpenAI documents Sign in with ChatGPT for open-source apps, but the app must hold and refresh OAuth tokens. Its Codex documentation says nothing about third-party tools driving `codex exec`.
* GitHub documents an OAuth device flow for the Copilot SDK. Its Copilot CLI documentation says nothing about third-party apps spawning `copilot`.
* Google: the Gemini CLI remains accessible via paid API keys, and Google's FAQs call piggybacking on CLI sign-in a violation. Gemini is not offered as a subscription route.
* No backend in V1 (ADR-0002). User data stays local (ADR-0003).
* One uniform adapter across subscription providers.
* The app should never hold subscription credentials. Only the vendor's own CLI touches them (rule 1).
* A CLI is an agent. It can read files and run commands unless its tools are switched off, so a route ships only if the vendor documents how to switch them off.

## Considered Options

* Spawn the user's own installed CLI
* In-app OAuth or sign-in for subscriptions (Daybook holds the tokens)
* Route subscription requests through a server
* Bring-your-own API key only

## Decision Outcome

Chosen option: "Spawn the user's own installed CLI", because it is the only option that serves subscription users without Daybook offering a provider login or holding plan credentials. One adapter, no OAuth client to register, no token to refresh, and the Claude route needs the CLI anyway. "Daybook never holds tokens" is a weak argument alone, since it already stores GitHub, Azure DevOps and API-key secrets in `safeStorage`; the case rests on the policy facts above.

V1 ships one subscription route, Claude Code. Codex and Copilot fail the tools-off requirement on vendor documentation read on 2026-11-09:

* Codex: a reply from an OpenAI contributor on openai/codex issue 58662 (closed as not planned on 2026-11-07, for `codex exec` 0.154.0) says no supported configuration guarantees both an empty model-visible tool list and no tool execution. Issue 60069, which asks for a zero-tools option, is open. The installed 0.165.0 `exec --help` lists no tool-off flag.
* Copilot: `-p` ignores piped stdin ("Piped input is ignored if you also provide a prompt with the `-p` or `--prompt` option"). On github/copilot-cli issue 5306 (1.0.105), an empty `--available-tools` leaves all 21 built-in tools available. The installed 1.0.108 `--help` lists no hooks-off flag and no status command.

Neither route was run with a tool canary, because neither ships in V1.

Gemini is not offered.

### Rules

For the Claude route:

1. The app MUST NOT read, store, copy or transmit subscription credentials. It MUST NOT run `claude setup-token` or `claude auth login`, set `CLAUDE_CODE_OAUTH_TOKEN`, or read the macOS Keychain entry or `~/.claude/.credentials.json`.
2. A CLI MUST be spawned only by an explicit user click: Generate, or Test connection (rule 9). It MUST NOT be spawned at launch or on a settings screen.
3. The CLI MUST run in an empty temporary directory that is removed afterward.
4. The argument list MUST be exactly `claude -p --safe-mode --tools "" --disallowedTools "mcp__*" --strict-mcp-config --disable-slash-commands --settings '{"disableAllHooks": true}' --permission-mode dontAsk --no-session-persistence --output-format text`, plus `--model haiku` on Test connection only. The app MUST NOT use `--bare` (bare mode never reads OAuth credentials or the keychain), `--allowedTools`, `--add-dir`, `--mcp-config` or `--dangerously-skip-permissions`. If a flag is rejected, the run MUST fail and MUST NOT retry without it.
5. The prompt MUST be passed on stdin.
6. Every run MUST have a 120-second timeout, after which the process group is killed.
7. CLI output MUST be treated as untrusted text: ANSI codes stripped, and output over 64 KiB counts as a failure.
8. The app MUST refuse a `claude` older than 2.1.322, the version the argument list was verified on, and show "update Claude Code". The check runs `claude --version` as the first step of Test connection and of Generate.
9. A provider MUST show "configured" when the stored absolute path exists and is executable (a file check, no spawn), and "ready" only after Test connection succeeds. Test connection runs `claude auth status` (exit 0 means signed in, exit 1 means signed out), then sends the fixed prompt "Reply with the word ok." with no repository data, and succeeds on non-empty stdout. A failed Generate demotes "ready" to "configured".
10. The child process MUST receive only `PATH` (`/usr/bin:/bin` plus the binary's directory), `HOME`, `USER`, `LANG`, `TMPDIR` and `CLAUDE_CODE_DISABLE_FILE_MENTIONS=1`.
11. The app MUST NOT spawn a shell to find a CLI. It checks `~/.local/bin`, `/opt/homebrew/bin` and `/usr/local/bin`, and the user can choose another path in Settings. A macOS app launched from Finder does not inherit the shell `PATH`, and a shell function is invisible to `execFile`.
12. Before each release the maintainer MUST run the canary script: it sends a prompt asking the CLI to run a command and read a file outside the working directory, and passes only if the answer is `NO-TOOLS` and the marker file does not exist. A failing canary disables the route in that release.
13. The route MUST be off by default. Enabling it MUST show this notice, with the Accept button stored locally with the notice version and revocable: "Anthropic's documentation says products built on `claude -p` or the Agent SDK should use API key authentication, and that third-party developers may not route requests through subscription credentials. Daybook runs your own Claude Code and never touches your credentials, but Anthropic's terms do not explicitly cover this, and your account could be affected. API keys are the supported route." Before each release the maintainer MUST re-read the cited pages and bump the notice version if the text changed.

Conditions for reopening a route in V1.x:

* Codex: OpenAI documents a no-tools option, or Daybook adopts Sign in with ChatGPT after a new ADR. Codex MUST NOT read `~/.codex/auth.json`, MUST NOT run with `CODEX_API_KEY`, and MUST be removed if Daybook becomes paid or hosted, because OpenAI states app-server authentication was never permitted for commercial or hosted services.
* Copilot: GitHub documents a zero-tools option and a hooks-off flag. The prompt goes on stdin with no `-p`. The app MUST NOT pass `GH_TOKEN`, `GITHUB_TOKEN` or `COPILOT_GITHUB_TOKEN`, because an environment variable silently overrides the stored sign-in, and MUST NOT read `~/.copilot/config.json`.
* An undocumented value such as `__none__` is never relied on.

### Positive Consequences

* Credentials stay inside Claude Code. Daybook has no token storage to secure for this route.
* Adding a CLI means adding an argument builder, not a new integration.
* The route is testable in CI with fake executables, and the real CLI is covered by the release canary.

### Negative Consequences

* The route carries terms risk the user accepts. Anthropic's pages lean restrictive and may change.
* V1 subscription users without Claude Code get no subscription route. They use an API key.
* Daybook depends on flags and output formats it does not control. The canary and the version floor catch drift.
* The canary needs a real signed-in Claude Code, so it runs on the maintainer's machine, not in CI.

## Pros and Cons of the Options

### Spawn the user's own installed CLI

* Good, because the user signs in to the vendor's own unmodified CLI, so Daybook never handles credentials.
* Good, because one adapter covers every subscription provider that passes the tools-off requirement.
* Bad, because the user has to install the CLI and sign in to it first.
* Bad, because CLIs are agentic tools that must be locked down (rules 2 to 7 and 10).

### In-app OAuth or sign-in for subscriptions (Daybook holds the tokens)

* Good, because the user would not need a separate CLI.
* Bad, because Anthropic does not permit third-party developers to offer Claude.ai login or route requests through plan credentials.
* Bad, because Google's guidance calls piggybacking on CLI sign-in a violation.
* Bad, because Daybook would take custody of tokens, which rule 1 forbids. OpenAI's Sign in with ChatGPT for open-source apps requires exactly that.

### Route subscription requests through a server

* Bad, because there is no backend in V1 (ADR-0002).
* Bad, because a server would hold or relay plan credentials, which has the same policy and custody problem as in-app OAuth.

### Bring-your-own API key only

* Good, because it has no subscription-terms risk, and keys stay on the user's machine.
* Bad, because users whose only AI access is a subscription would have no route.

## Links

* Related to ADR-0002 (No backend in V1)
* Related to ADR-0003 (Keep user data local; name the exceptions)
* Issues read 2026-11-09: https://github.com/openai/codex/issues/58662, https://github.com/openai/codex/issues/60069, https://github.com/github/copilot-cli/issues/5306
* Sources read 2026-11-09: https://code.claude.com/docs/en/legal-and-compliance, https://code.claude.com/docs/en/headless, https://developers.openai.com/siwc/token-sharing-open-source, https://docs.github.com/en/copilot/how-tos/copilot-sdk/auth/authenticate, https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/
