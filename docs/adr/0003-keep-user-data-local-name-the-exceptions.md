# Keep user data local; name the exceptions

* Status: accepted
* Deciders: rubentanahara
* Date: 2026-11-09

## Context and Problem Statement

Daybook reads a developer's commits, diffs, tickets and comments and turns them into a standup and a weekly summary. That material can contain employer code, secrets and other people's words. Where does user data live, and which data leaves the machine? When this was written, no code existed, so this ADR records intent, not existing behavior.

## Decision Drivers

* Source code and tracker content are sensitive. Most of it belongs to the user's employer.
* There is no Daybook-operated server in V1 (ADR-0002).
* The app is free, open source and BYO-AI. No revenue depends on collecting data.
* Users must be able to see what leaves the machine, so "local" cannot be a vague promise.

## Considered Options

* Local-only by default, with named exceptions
* Cloud sync of history across devices
* A Daybook-hosted backend that stores history or generated summaries
* Default-on telemetry/analytics
* Strictly offline, with no network exceptions

## Decision Outcome

Chosen option: "Local-only by default, with named exceptions". It is the only option that works without a backend (ADR-0002) and still lets the product read trackers and use the user's AI.

* History is stored in local SQLite (`better-sqlite3`). No sync. It stores generated text, edited text, the text at each copy and citation IDs, never raw diffs, prompts or tokens. The file is created with mode `0600`. Daybook does not encrypt it in V1: it relies on disk encryption, the docs say so, and history is included in backups. Malware running as the same user can read it.
* Secrets live only in `safeStorage`, through the async methods (the sync methods are removed in Electron 46). Tokens MUST NOT be stored when encryption is unavailable (on Linux, the `basic_text` backend counts as unavailable). A decrypt failure counts as a missing token and asks the user to reconnect; it never falls back to plaintext. Disconnecting an account deletes its token.
* No telemetry in V1.

### Exceptions: data that leaves the machine

| # | What leaves | To whom | Trigger | Consent mechanism |
|---|---|---|---|---|
| E1 | Prompts containing titles, descriptions, comments, diffs and saved daily text (after redaction) | The AI provider the user chose: an API key for Anthropic, OpenAI, Gemini or any OpenAI-compatible URL, or the provider behind the user's own Claude Code | The user clicks Generate. CLI routes also run on Test connection, which sends a fixed prompt with no repository data. | First-run notice, plus a preview before the first send for each repo. The Claude route is off by default and enabling it shows the notice in ADR-0001. |
| E2 | API requests carrying the user's credentials and queries for their own activity | GitHub (including Projects v2) and Azure DevOps | Generate, Connect or Refresh. No background polling. | The user connects each account |
| E3 | Crash reports: stack trace, app version, OS and Electron version, process type, error type | Sentry | A native crash | Opt-in |
| E4 | Update check request (IP address and request headers; no user content) | GitHub Releases, through `electron-updater` | After onboarding, every 24 hours while running, and on a manual "Check for updates" | On by default. A Settings toggle turns off automatic checks. The first-run notice names it. |
| E5 | OAuth sign-in requests | GitHub and the Microsoft Entra sign-in host (Azure DevOps) | The user connects an account | User-initiated |

### Rules

1. MUST redact secrets before any send, in titles, descriptions, comments, diffs and saved daily text. Redaction is pattern-based and will miss some secrets. User-facing docs MUST say: "Daybook removes text that looks like known secret formats, such as prefixed API tokens and private-key blocks. It cannot recognise passwords, connection strings in custom formats, internal hostnames, customer or personal data, or whether the code itself is confidential. Anything it does not recognise goes to your AI provider unchanged. If that is not allowed, use a local model or deny-list the repo."
2. MUST skip `.env` files, lockfiles, binaries and generated files.
3. MUST apply a size budget to each diff.
4. MUST show a preview of exactly what will be sent, after redaction, before the first send for each repo. The weekly roll-up reuses the approvals.
5. MUST support per-repo and per-org deny lists.
6. MUST offer a first-class local-model path through an OpenAI-compatible URL.
7. MUST show a first-run notice that the user's own provider receives this data and that automatic update checks are on.
8. MUST treat AI output as untrusted. Commit messages and comments written by other people reach the model. The model cites source IDs only; the app builds every link from fetched data, and an unknown ID counts as an invalid citation.
9. The renderer MUST NOT load remote images or links automatically. Markdown renders with raw HTML off, and images are stripped from display, clipboard and export. External links open only from the main process, only for `https:` URLs on `github.com` or the connected Azure DevOps host.
10. Local `git` MUST NOT run `fetch` in V1. Daybook reads local refs and shows when the last fetch happened.
11. Source API requests (E2) MUST run only on Generate, Connect or Refresh, with no background polling. Update checks (E4) follow their own row.
12. Sentry MUST be initialized only when persisted consent is true, read before the app is ready, with `dataCollection: { userInfo: false }`, only the `SentryMinidump` integration (so `MainProcessSession` and every default integration are removed), `beforeBreadcrumb: () => null`, and `beforeSend` dropping every event whose `platform` is not `native`. Tracing, renderer init, replay and feedback MUST NOT be enabled. A consent change relaunches the app, and crash coverage starts after the relaunch. Opting out deletes pending minidumps and the Sentry cache under `userData`. The DSN is present only in tag builds; the Sentry test injects a dummy DSN and a capturing transport. A forced native crash against a capturing transport MUST send exactly one event, and a normal exit MUST send none.
13. Logs MUST NOT contain prompts, model output, HTTP headers or bodies, full CLI or git argument lists, or remote URLs. Home paths are shortened to `~`.
14. An OpenAI-compatible URL MUST use `https:`, except loopback addresses, where `http:` is allowed.

### Positive Consequences

* Daybook operates no store of user data, so it has none to breach.
* Outbound flows are enumerated above, so the plain-language data page can be written from them.
* The local-model path lets a user keep E1 on their own machine as well.

### Negative Consequences

* History is tied to one machine. Daybook offers no backup, and a second device starts empty.
* Without telemetry, Activation and Quality are measured only in moderated usability sessions.
* Redaction (rule 1) will miss some secrets. With a cloud provider, residual risk sits with the user.
* The Sentry rules rest on the `@sentry/electron` 8.3.0 package source, not its documentation. The Sentry test in rule 12 is the gate; if it fails, Sentry is cut from V1.
* Keychain behavior across updates depends on identical Team ID, `appId` and `productName` in every release. The `safeStorage` across an update gate in the decision ledger tests it.

## Pros and Cons of the Options

### Local-only by default, with named exceptions

* Good, because it needs no backend and matches ADR-0002.
* Good, because every exception is user-driven, opt-in, or on by default with a Settings toggle and a first-run notice (E4), and stated in writing.
* Bad, because it offers no multi-device history and no backup.

### Cloud sync of history across devices

* Good, because history would follow the user to other devices.
* Bad, because summaries derived from employer code would be stored off-machine.
* Bad, because it needs a sync service or a third-party store. V1 has no sync.

### A Daybook-hosted backend that stores history or generated summaries

* Good, because it would allow server-side features later.
* Bad, because it needs a backend, and ADR-0002 rules that out for V1.
* Bad, because Daybook would become a custodian of user code-derived data and a breach target.

### Default-on telemetry/analytics

* Good, because the Activation and Quality metrics could be measured in the field.
* Bad, because V1 has no telemetry. Crash reporting is opt-in only, and the Sentry session integration is removed.
* Bad, because it would add an unnamed outbound flow and weaken the "local" promise.

### Strictly offline, with no network exceptions

* Good, because no data would leave the machine at all.
* Bad, because GitHub and Azure DevOps (E2) could not be read and cloud AI (E1) could not be used, which removes core V1 sources and routes.
* Bad, because there would be no auto-update. Users who want offline AI already have the local-model path.

## Links

* Depends on ADR-0002
* Related to ADR-0001
