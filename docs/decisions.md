# Daybook: decision ledger

Settled on 2026-11-09. The public name is Daybook. No trademark search was performed; three iOS and iPad journaling apps on the App Store use the name, none is a developer tool, and the risk is accepted. The name needs no npm package, GitHub user or domain in V1: distribution is GitHub Releases and the landing page is on GitHub Pages.

## Product

Daybook is a desktop app that writes a developer's daily standup and weekly summary from their git and tracker activity. Every generated line cites the commit or ticket it came from.

| Area | Decision |
|---|---|
| Model | Free and open source (MIT). Users bring their own AI. |
| Platforms | macOS arm64 ships first. Intel (x64) is V1.x: it needs one electron-builder run for both architectures and an Intel runner for the smoke test. CI builds macOS, Windows and Linux from day one. Windows and Linux builds are unsigned CI artifacts, never published, until those platforms ship. |
| Sources in V1 | Local git, GitHub, GitHub Projects v2, Azure DevOps. |
| Sources later | Jira in V1.1. Nothing in V1 depends on it. |
| Outputs | Daily standup and weekly summary, editable, copy to clipboard and Markdown export. The weekly summary rolls up saved dailies. |
| History | Local SQLite. No sync. |

## V1 features

In:

- "Since last working day" logic (Monday covers Friday), with configurable work days and timezone. Public holidays are not modelled. A window with no activity shows an empty state and makes no AI call.
- Commits come from local git across all branches for cloned repos. For uncloned repos, default-branch commits come from the commits API and feature-branch work comes only through pull requests. Branch work with no pull request in an uncloned repo is not seen, and the UI says so.
- Multiple accounts: more than one GitHub account and more than one Azure DevOps organization.
- Tone and template presets (concise, detailed) and one editable prompt template.
- An output-language setting. It changes the prompt only, not the UI.
- Browsable local history.
- Keyboard shortcuts.
- Auto-update.
- Onboarding in three steps: (1) pick the AI route, with the data notice inline; (2) connect one source, local git first, GitHub and Azure DevOps skippable; (3) first Generate, with the preview as its gate. Notices are inline text, not stacked modals.
- Opt-in crash reporting.

Out of V1: GitHub Enterprise Server, scheduled "standup ready" notifications, UI translation, team features, plugins, Jira, Intel Macs, Codex and Copilot subscription routes.

## AI access

| Route | Decision |
|---|---|
| API keys | Anthropic, OpenAI, Gemini, and any OpenAI-compatible URL (covers Ollama and similar), through the Vercel AI SDK providers `@ai-sdk/anthropic`, `@ai-sdk/openai`, `@ai-sdk/google` and `@ai-sdk/openai-compatible`. Gemini is its own adapter. Keys are stored in `safeStorage` only. An OpenAI-compatible URL must be `https:`, except loopback addresses. |
| Subscription: Claude | V1's only subscription route. Spawns the user's own installed `claude -p`. Off by default; enabling it shows the notice in ADR-0001. |
| Subscription: Codex, Copilot | V1.x. Both fail the tools-off requirement (ADR-0001). |
| Subscription: Gemini | Not offered. Gemini is reachable by API key only. |
| Design | One `CliProvider` adapter with a small per-CLI argument builder. The app never handles subscription credentials. |
| Gate | Each provider is gated by its tests. A provider that fails its gate moves to V1.x. |

The exact `claude` argument list, the environment allowlist, the readiness check and the notice text are rules 1 to 13 of ADR-0001.

## Data sent to the user's AI provider

Titles, descriptions, comments, diffs, and saved daily text for the weekly roll-up, with these guardrails:

- Secret redaction before any send, applied to titles, descriptions, comments, diffs and saved daily text. Skip `.env` files, lockfiles, binaries and generated files.
- A size budget per diff.
- A preview of exactly what will be sent, after redaction, shown before the first send for each repo. One sheet lists the repos as rows. A repo not yet approved blocks Generate inline. The weekly roll-up reuses the approvals.
- Per-repo and per-org deny lists.
- A first-class local-model path through an OpenAI-compatible URL.
- A first-run notice that the user's own provider receives this data.

Redaction is pattern-based and will miss some secrets. The docs say so (ADR-0003 rule 1).

## Sources and sign-in

Every sign-in is client-side. The app ships no `client_secret`. Source facts below were read from official documentation on 2026-11-09.

### GitHub

- Repo data: a GitHub App with read-only Contents, Pull requests, Issues and Metadata. Device flow is on. Expiring user tokens are on (8 hours for the access token, 6 months for the refresh token) and the sign-in requests `offline_access`. Refreshing needs only `client_id`, `grant_type=refresh_token` and `refresh_token`, because the token came from device flow. The new refresh token is persisted before the new access token is used. A token response with no `refresh_token` is refused.
- The App can only access accounts where it is installed. A repo owner without the installation shows a banner "Daybook is not installed on `<owner>`" with the install link, and its repos are excluded. A partial result is never shown as complete.
- When the App cannot be installed on an owner, the user may paste a fine-grained personal access token with read access to Contents, Pull requests, Issues and Metadata, stored in `safeStorage`.
- Projects v2: a separate connection through a GitHub OAuth App with the `read:project` scope only, device flow, no `repo` scope and no `offline_access`. `read:project` covers user-owned and organization-owned boards. Fine-grained tokens are never used for Projects, because GitHub documents that they cannot access user-owned projects. An organization policy that blocks the OAuth App shows a banner.
- Activity discovery, in this order: `GET /search/issues?q=is:pr author:LOGIN updated:>=DATE` (30 requests per minute); if it returns 403, `GET /repos/{owner}/{repo}/pulls?state=all` for the repos the user picked. Then `GET /repos/{owner}/{repo}/pulls/{n}/commits` per pull request, at most 250 commits each, filtered by author and window. Default-branch commits of uncloned repos come from `GET /repos/{owner}/{repo}/commits?author=&since=&until=`. Commit search, the events API and GraphQL contribution counts are not used, because they cover only the default branch or too short a history.
- Requests run serially and honor `retry-after` and `x-ratelimit-reset`. User tokens have a 5,000 requests-per-hour quota.

### Azure DevOps

- A Microsoft Entra public client: authorization code with PKCE, resource `499b84ac-1321-427f-aa17-267ca6975798`, scope `499b84ac-1321-427f-aa17-267ca6975798/.default` plus `offline_access`, no secret. The app is registered multi-tenant (`AzureADMultipleOrgs`) and signs in against the `/organizations` authority. It registers delegated `vso.code`, `vso.work` and `vso.profile`. The Entra sign-in gate checks the scopes in the issued token.
- The redirect is `http://127.0.0.1` registered through the manifest `replyUrlsWithType`. Entra ignores the port when matching. `localhost` and `[::1]` are not used.
- Organizations backed by personal Microsoft accounts are not supported by Entra apps for Azure DevOps. They connect with a personal access token only.
- A consent error shows a banner telling the user to ask an administrator to approve Daybook, with the token fallback.
- The legacy Azure DevOps OAuth is not used: Microsoft accepts no new registrations since April 2025 and schedules full deprecation in 2026.
- Personal access tokens are organization-scoped with `vso.code` and `vso.work`. Their maximum lifetime is set by the organization, so any 401 or non-JSON response is shown as "expired or revoked". The token path asks the user to confirm their email, and commits are re-filtered locally by author email.
- Endpoints, API version 7.1: commits `GET https://dev.azure.com/{org}/{project}/_apis/git/repositories/{repo}/commits` with `searchCriteria.author`, `fromDate`, `toDate`; pull requests `GET https://dev.azure.com/{org}/{project}/_apis/git/pullrequests` with `creatorId`, `status=all`, `minTime`, `maxTime`; work items `POST https://dev.azure.com/{org}/_apis/wit/wiql?timePrecision=true` with `[System.ChangedBy] = @Me AND [System.ChangedDate] >= '<ISO time>'`, then a batch fetch by ID; identity `GET https://app.vssps.visualstudio.com/_apis/profile/profiles/me` (Entra tokens only). Work items are labelled "items you last changed".

### Native app sign-in rules (RFC 8252)

- The redirect is `http://127.0.0.1:{random port}/{path}`, listening on loopback only, single use, with a 2-minute timeout.
- PKCE is required. A `state` check is required, which is stricter than the RFC.
- No custom URL scheme: another app can register the same scheme and receive the code.

## Stack

- TypeScript (strict), pnpm workspaces: `apps/desktop`, `packages/core`. `core` holds the git and tracker clients, prompt building and redaction, so it is testable without Electron.
- Fixed versions: `electron` 45.2.3 (supported majors are 43 to 45), `electron-builder` 26.18.1 and `electron-updater` 6.8.12 (both the registry `v26` tag), `better-sqlite3` 13.0.3, `@sentry/electron` 8.3.0, pnpm 12.16.1, Node 26.13.0 for tooling. Electron 45 bundles its own Node, so main and preload use no Node-26-only APIs. `electron` is a `devDependency` and `electron-updater` a `dependency`.
- Every other dependency (`electron-vite`, React, Tailwind, TanStack Query, `zod`, `ai`, Vitest, `@playwright/test`, Storybook) is installed at the registry latest, saved exact, after a context7 check. Registry latest on 2026-11-09: `electron-vite` 6.0.0, `zod` 4.6.7, `@playwright/test` 1.65.0, `ai` 7.0.166.
- Electron with `electron-vite`, React, Tailwind, TanStack Query. A typed IPC contract validated with `zod`.
- `pnpm-workspace.yaml` sets `nodeLinker: hoisted` and `allowBuilds: { electron: true }`. `dangerouslyAllowAllBuilds` is never set.
- SQLite through `better-sqlite3`, which ships N-API prebuilds for `darwin-arm64`. `asarUnpack` is `["node_modules/better-sqlite3/**"]`. `npmRebuild` is `false`, because the N-API prebuilds ship in the package; there is no `@electron/rebuild` and no `install-app-deps` step. History is unencrypted. It stores generated text, edited text, the text at each copy and citation IDs, never raw diffs, prompts or tokens. The file is created with mode `0600`.
- Shell out to the user's own `git`, through `execFile` with no shell. Read subcommands only: `log`, `show`, `diff`, `rev-parse`, `for-each-ref`. Never `status` or `fetch`. The argument list is `git --no-pager -c core.fsmonitor=false <subcommand> <options> --end-of-options <revisions>`, with `--no-ext-diff --no-textconv` on `log`, `show` and `diff`. The environment sets `GIT_TERMINAL_PROMPT=0` and `GIT_OPTIONAL_LOCKS=0` and removes every inherited `GIT_*` variable. `octokit` for GitHub. Plain `fetch` for Azure DevOps.
- Sentry for crash reporting, opt-in, native crashes only (ADR-0003 rule 12).
- Vitest, Playwright for Electron, Storybook with the accessibility addon.
- `electron-builder` with `electron-updater` (rules under CI/CD).
- `electron-log` with the log rules of ADR-0003 rule 13. Window bounds are persisted in-house; `electron-window-state` is not adopted.

## Architecture rules

1. Citation model: `core` emits `Line = { id, text, sources: SourceRef[] }`. A line is one bullet and carries one or more sources. A `SourceRef` is a commit SHA, pull request number, issue number, work item ID or project item ID, with its repo or organization. The model returns line text plus source IDs. `core` drops any ID not in the fetched set. The editor stores `sources` beside `text`, never inline: editing text leaves `sources` untouched, split copies them, merge unions them, and the user can remove a chip. `core` builds every source URL from the stored record, never from model text.
2. IPC: one contract in `apps/desktop/src/shared/ipc-contract.ts`, a map of channel to `{ input, output }` zod schemas. Main parses both. Preload exposes one typed `invoke` and never raw `ipcRenderer`. A sandboxed preload in Electron 45 can `require` only `electron`, so `zod` is bundled into it. IPC exposes task-level operations only (for example `generateStandup(repoId)`); no call accepts a path, command, URL or git arguments chosen by the renderer. Domain schemas live in `core`, so the dependency runs from desktop to core only.
3. Renderer: served from a custom `app://` protocol, never `file://`. The scheme is registered once, before `ready`, with `standard` and `secure`, and never `bypassCSP`. The CSP is sent in the `Response` headers and path traversal is blocked. Prod CSP: `default-src 'none'; script-src 'self'; style-src 'self'; img-src 'self'; font-src 'self'; connect-src 'none'; base-uri 'none'; form-action 'none'`. The renderer makes no network calls. Dev adds only the Vite origin and `'unsafe-inline'` for HMR and never ships.
4. Data fetching: TanStack Query with `networkMode: 'always'`, `refetchOnWindowFocus: false`, `staleTime: Infinity` for database reads and `retry: false` for generate. Keys are `[channel, input]`. Generate returns a `jobId`; main emits `job:progress` and `job:done`; `job.cancel(jobId)` aborts the child process or fetch. The timeout lives in main.
5. `core` has no Electron imports, enforced by a lint rule. `core` defines a `HistoryStore` interface with an in-memory fake for its tests. The SQLite implementation lives in `apps/desktop/src/main/db`. The workspace package is excluded from dependency externalization, or exported as TypeScript source.
6. AI output renders as text nodes, never `dangerouslySetInnerHTML`. Markdown export is built in `core` from structured lines, with escaping. `setWindowOpenHandler` denies, `will-navigate` is blocked, and ANSI codes are stripped from CLI output.
7. Components receive an `IpcClient` through React context. The app provides the real one and Storybook a fake.
8. The app is a Dock app with a menu-bar item. Closing the window hides it; Cmd+Q quits. The menu-bar item and the global shortcut only show and focus the window; they never generate. `globalShortcut.register` returns a boolean and fails silently on a conflict, so a `false` is shown in Settings and the accelerator is configurable. The `Tray` object is held at module scope. Tray icons are named `*Template.png` with `@2x` siblings.
9. `data-platform` is set on `<html>`. macOS gets vibrancy and `titleBarStyle: 'hiddenInset'`; Windows and Linux get an opaque background. Vibrancy has a solid fallback that meets 4.5:1 contrast.
10. Auto-update: `checkForUpdates` is called from one place, after onboarding is dismissed and then by an in-app timer every 24 hours, because `electron-updater` has no scheduler. A manual "Check for updates" menu item always works. The Settings toggle "Check for updates automatically" turns off the timer and sets `autoInstallOnAppQuit = false`. Every updater call is guarded by `process.platform === 'darwin'` and an `error` listener is attached.

## Security baseline (V1 ship gates)

- `contextIsolation` on, `sandbox` on, `nodeIntegration` off, strict CSP, no remote content, navigation blocked.
- Electron fuses, set with the native electron-builder `electronFuses` option (flipped before signing, so there is no `afterPack` hook): `runAsNode` false, `enableNodeCliInspectArguments` false, `onlyLoadAppFromAsar` true, `enableEmbeddedAsarIntegrityValidation` true, `grantFileProtocolExtraPrivileges` false, `enableNodeOptionsEnvironmentVariable` false. Playwright launches Electron with `--inspect=0` and its docs require the `nodeCliInspect` fuse to stay on, so the Playwright smoke test runs on a test-flavor build that differs only in `enableNodeCliInspectArguments: true`. electron-builder 27 alpha is not used.
- Every IPC message validated with `zod`.
- Secrets live only in `safeStorage`, through `isAsyncEncryptionAvailable`, `encryptStringAsync` and `decryptStringAsync` (the sync methods are removed in Electron 46). Tokens are not stored when encryption is unavailable; on Linux the `basic_text` backend counts as unavailable. Any decrypt failure counts as "token missing" and asks the user to reconnect; it never falls back to plaintext. Disconnecting an account deletes its token. Team ID, `appId` and `productName` stay identical in every release, because Electron documents that signing changes can make Keychain re-prompt on every update.
- Prompt-injection posture: commit messages and comments written by others reach the AI, so AI output is untrusted and the renderer loads no remote images or links automatically. The model cites source IDs only, and the app builds every link (ADR-0003 rules 8 and 9).
- Supply chain: Dependabot with the `github-actions` and `npm` ecosystems (`npm` covers pnpm); GitHub Actions pinned by SHA with a `# vX.Y.Z` comment; a weekly `pnpm audit --audit-level high --prod` job whose failure turns the run red (pnpm 12's two-document lockfile is invisible to the dependency graph, so this audit is the vulnerability signal); gitleaks as the MIT CLI from a pinned release with a verified checksum, never `gitleaks-action` (its licence is not MIT since v2.0.0), and run locally before the first push.
- Workflows: top-level `permissions: contents: read`, with `contents: write` only on the release job; `persist-credentials: false`; `pnpm install --frozen-lockfile`; `pull_request`, never `pull_request_target`. Repository setting "Require approval for all external contributors". Fork pull requests receive no secrets.
- Rulesets: `main` requires a pull request with 0 approvals, no force pushes and linear history; the required status checks are added to the ruleset in the pull request that adds `ci.yml`, because a required check that never reports blocks every merge. The maintainer's account is a `User` bypass actor for pull requests only; the maintainer uses it only to merge the release pull request. Tags `v*` block `update`, `deletion` and `non_fast_forward`; `creation` is not restricted, because Release Please must create the tag. Squash merge only, with the pull request title as the commit message.
- `SECURITY.md`, and a `security-engineer` audit before the first release.

## QA

- `packages/core` at 85% line coverage or better, plus branch-coverage floors on redaction, the work window, citation validation and the CLI argument builders, plus a nightly mutation run on those four modules. Each floor is the first measured baseline and is never lowered. `core` tests run on all three OSes.
- Tracker clients covered by contract tests against recorded, sanitized fixtures. Fixture repos are generated by a script with fixed dates and `example.com` identities.
- Trust is measured on a golden set of fixture repos with expected citations, against recorded model responses keyed by prompt template version, input, model and parameters. A cache miss fails with "re-record needed". The nightly live-model run asserts structure only: citations resolve with the same validator, no cited ID is outside the input, a planted canary secret is absent from the built payload, and latency. A nightly failure opens an issue and never blocks a pull request.
- The redaction corpus: positive cases for common secret formats, including secrets on removed diff lines and in commit messages and comments; a negative corpus (SHAs, UUIDs, lockfile hashes must survive); a `known-misses` fixture that asserts what is not caught and generates the docs list; a test that the preview bytes equal the bytes sent to a capturing fake provider. Secret-shaped fixtures are built from fragments at runtime and allowlisted in gitleaks.
- The work window is a pure function of `now`, timezone and work days, tested table-driven: Monday covers Friday, a Sunday-to-Thursday week, 23-hour and 25-hour DST days, a 00:30 commit, author vs committer date, and a user timezone that differs from the machine's.
- Injection fixtures with hostile commit messages, comments and branch names, including a forged citation, asserted on rendering. Playwright asserts zero remote requests. Multi-account fixtures: two GitHub accounts seeing one repo, two Azure DevOps organizations, one account failing mid-run, and a squash merge duplicating local commits.
- A git hardening test: a fixture repo whose `.git/config` sets `core.fsmonitor`, `diff.external` and a `textconv` driver to scripts that write a marker file; running the app's git wrapper over every allowed subcommand must leave no marker.
- Playwright smoke tests run only on the test-flavor build, on a macOS runner. The release build gets: `codesign --verify --deep --strict`, `spctl --assess`, `stapler validate`, `npx @electron/fuses read` against the expected fuses, `ELECTRON_RUN_AS_NODE=1` doing nothing, a tampered copy of `app.asar` refusing to start, and the launch check: the app must create the history database file within 20 seconds of launch, then it is terminated. Playwright launches the packaged binary through `executablePath` set to `<App>.app/Contents/MacOS/<binary>` with no main-script argument, and asserts the CSP header on the document response.
- Live tests against a dedicated test GitHub org and Azure DevOps org, run manually or nightly, never in pull-request CI.
- CLI providers are tested in CI with fake executable scripts: missing binary, signed out, timeout with the process group killed, empty, non-UTF8, oversized and ANSI output, non-zero exit with output, a stdin write larger than the pipe buffer, the exact argument list present in argv, an empty working directory removed afterward, and zero spawns at launch and on settings screens. Fakes are seeded from sanitized real output captured when the route was built.
- The Sentry test: a forced native crash against a capturing transport sends exactly one event, and a normal exit sends nothing.
- Storybook stories for the React components.

## CI/CD

- GitHub Actions. Pull requests run lint, typecheck, unit tests and an unsigned build, and never receive signing secrets. Steps use `pnpm/action-setup` first, then `actions/setup-node` with `cache: pnpm`.
- `ci.yml` runs on Ubuntu: lint, typecheck, Vitest with the coverage gates, and gitleaks. A `macos-26` job builds unsigned and runs the Playwright smoke test. Windows and Linux jobs build unsigned artifacts with `--publish never`; `packages/core` Vitest also runs in the macOS, Windows and Linux jobs. Required checks use no path filters. The macOS runner is the pinned arm64 `macos-26`, never a `-large` or `-xlarge` runner; it is free on public repositories. `nightly.yml` uses `schedule` and `workflow_dispatch`, with live-test secrets in a `live-tests` environment. Node is pinned in `.node-version` and pnpm by the `packageManager` field.
- Conventional commits, versioned by Release Please in manifest mode with one root component, `"draft": true` and `"force-tag-creation": true`. Pull request titles are linted by an in-repo regex step under `pull_request`, with no third-party action. The release pull request is created with `GITHUB_TOKEN`, and GitHub documents that events triggered by `GITHUB_TOKEN` create no new workflow runs, so the release pull request gets no required checks. The `main` ruleset lets the maintainer's account bypass it for pull requests only, and the maintainer merges the release pull request through that bypass. No personal access token or GitHub App token is used for this.
- The release build is a job in the Release Please workflow, gated on `release_created`, running in the `release` environment. That environment uses "Selected branches and tags" with `main`, requires the maintainer as reviewer, and leaves "Prevent self-review" off. It holds the signing secrets (see Maintainer prerequisites). The job runs `electron-builder --mac --arm64 --publish always` with the tag equal to `v<version>`, then publishes the draft release only after `gh release view` lists the zip, the dmg and `latest-mac.yml`. Clients never see a release without the feed file.
- Signing: Developer ID Application certificate, hardened runtime, `notarize: true`, custom entitlement plists with `allow-jit` and `allow-unsigned-executable-memory`. `disable-library-validation` is added only if the launch check on the release build fails. The `.p8` key is written to a 0600 temporary file, `APPLE_API_KEY` points at that path, the file is deleted afterward, and no Apple ID variables are set. electron-builder 26.18.1 only warns on missing credentials, so CI always runs `stapler validate` and `spctl --assess`.
- Sentry DSN: injected only in tag builds, as a non-secret Actions variable. Pull request and test builds have an empty DSN, so the SDK sends nothing; the Sentry test injects a dummy DSN and a capturing transport. Sentry auth tokens are never committed.
- The release spike, run as soon as the maintainer prerequisites exist, opens SQLite, does a `safeStorage` round trip, publishes v0.0.1, updates to v0.0.2 and checks that the stored value survives the update.
- Release checklist, run by the maintainer for every release: run the Claude canary script (ADR-0001 rule 12); re-read the vendor pages cited in ADR-0001 and bump the notice version if the text changed; merge the release pull request through the ruleset bypass; confirm the draft release lists all three assets.

## Build order

1. Gate spikes that need no prerequisites: Claude route canary, git hardening test, packaged `better-sqlite3` on an unsigned build.
2. Tracer bullet: local git plus one API-key provider or the Claude route, producing a cited standup the user copies.
3. Everything else, built against fakes and recorded fixtures where a maintainer prerequisite is missing.
4. At productionization, once the maintainer prerequisites exist: signed hello-world with auto-update, Sentry crash test, GitHub and Azure DevOps sign-in, live recordings.

If time runs short, cut in this order: the Claude CLI route, Windows and Linux CI, Sentry, multi-account, Storybook. Auto-update stays: shipped clients without an updater can never be updated.

## Metrics of success

| Metric | Target |
|---|---|
| Trust | At least 98% of generated bullet lines carry at least one source ID that exists in that run's fetched set and falls inside the report window. A line with no resolvable citation is invalid. The count is taken on the model's raw output, before the app drops unresolved IDs. CI measures it on recorded responses (a regression check on the parser and validator); the nightly live run reports it per API-key provider and opens an issue when it falls below 90%. |
| Coverage | On the golden set, at least 95% of expected activity items, including feature-branch commits, appear in the standup. |
| Activation | Median time from first launch to first standup rendered under 3 minutes, in 5 moderated usability sessions. Participants are pre-screened for an API key or a signed-in Claude Code; sessions with and without those prerequisites are reported separately. Directional at n=5. |
| Quality | In the 5 moderated sessions, the median share of characters changed between the generated text and the text at copy time is under 30%, computed locally from saved history. Nothing is transmitted. Directional at n=5. |
| Reliability | Zero crashes or hangs across the Playwright smoke suite. A day's standup renders in under 15 seconds at p95 in the nightly Playwright run on the test-flavor build against an API-key provider, measured from the Generate click to the rendered standup, including fetch, redaction and the AI call, and excluding the preview wait. |
| Reach | 100 GitHub stars and 500 release downloads within 3 months, counted from GitHub. |

The targets other than Trust are guesses. There is no telemetry in V1: Activation and Quality come from moderated sessions only, and crash reporting sends only on a native crash.

## Repo and process

- `rubentanahara/daybook`, public from the first commit.
- Trunk-based with pull requests even when solo, conventional commits, ADRs in `docs/adr`.
- The backlog lives on a GitHub Projects board.
- Docs: `README` with a GIF, `CONTRIBUTING`, `SECURITY`, a plain-language page on what data leaves the machine (`docs/data-and-privacy`, written before the first release), and a landing page on GitHub Pages deployed from `site/` through Actions so `docs/` is not published by accident. Live tests need maintainer-owned test orgs and never run on pull requests.

## Design

- A Dock app with a menu-bar item and a global shortcut. The shortcut and the menu-bar item open the window on the latest saved standup with Generate and Copy as visible buttons; they never generate.
- macOS-native feel: system font, vibrancy, follows the OS light and dark setting.
- Mockups: a Vite design canvas in `design/` from `design-workflow`, finished before user stories.
- Screens: onboarding and connect sources, repo picker, standup editor with source hover, weekly summary, settings and data preview.

### UX policies

- No activity in the range: no AI call, and an empty state offers to widen the range.
- A failing source gives a partial standup with a banner naming the source and a fix action, for example "GitHub (acme): token expired, excluded". A partial result is never shown as complete.
- Each line has a focusable source chip. The popover opens on hover or focus, and a click opens a side panel. Popover text renders as plain text. An edited line keeps its chips, shows an "edited" mark, and keeps the generated original so the line can be reverted.
- The weekly view lists the days (saved, missing, skipped) with "Generate missing day". Weekly lines cite the dailies' citations, and the roll-up uses the edited text.
- The repo picker groups repos by account or organization, with a badge on each row. Citations read `owner/repo#123`. Onboarding connects one account; "Add another" lives in Settings. Deny lists sit on the row menu and in Settings.
- The Claude route shows as "Off" in onboarding. An inline toggle expands the notice with an Accept button. Acceptance is stored locally with the notice version and is revocable.
- GitHub and Azure DevOps requests run on Generate, Connect and Refresh only, with no background polling. Connecting a source runs one proof fetch (for example "14 commits found").
- Updates follow Architecture rule 10.

## Maintainer prerequisites

Only the maintainer can create these, at productionization. Until then, builds are unsigned and unpublished, and the blocked features are developed against fakes and recorded fixtures. Each prerequisite blocks the feature named; if it is never created, that feature moves to V1.x.

| Prerequisite | Blocks |
|---|---|
| Apple: a Developer ID Application certificate (`.p12`) and an App Store Connect API key (`.p8`, key ID, issuer ID), stored as `CSC_LINK`, `CSC_KEY_PASSWORD`, `APPLE_API_KEY`, `APPLE_API_KEY_ID` and `APPLE_API_ISSUER` in the `release` environment. | Signed, notarized and auto-updating builds (a release blocker) |
| GitHub: a GitHub App (device flow on, expiring tokens on, read-only Contents, Pull requests, Issues, Metadata) and a GitHub OAuth App (device flow, `read:project`). Their client IDs are public constants. | GitHub source, Projects v2 |
| Microsoft: an Entra app registration (multi-tenant, public client, manifest redirect `http://127.0.0.1`, delegated Azure DevOps permissions), and an Azure DevOps test organization. | Azure DevOps source |
| Sentry: a project and its DSN, stored as the Actions variable `SENTRY_DSN` in `release`. | Crash reporting |
| A dedicated GitHub test organization. | Live tests |
| Secrets in the `live-tests` environment: an Anthropic API key, a fine-grained read-only personal access token for the GitHub test organization, and a personal access token for the Azure DevOps test organization. The nightly run covers the API-key providers and the token paths. The Claude route and Projects v2 are covered by the release checklist and the gates. | Nightly live tests |

## Verification gates

Facts that exist only after code runs are encoded as gates. Each has a pass condition and a fallback decided now.

| Gate | Pass condition | Fallback |
|---|---|---|
| Claude canary (maintainer, before each release) | The exact argument list and environment allowlist make `claude` answer a prompt that asks it to run a command and read a file outside the working directory with `NO-TOOLS`, and the marker file is not created. Passed on Claude Code 2.1.322 on 2026-11-09. | The Claude route is disabled in that release. |
| Git hardening test | No marker file after any allowed subcommand. Passed on git 2.56.0 on 2026-11-09: unhardened `log -p`, `show`, `diff` and `status` ran the textconv, external diff and fsmonitor helpers; the hardened argument list ran none. | A failing subcommand is removed from the allowed list. |
| Packaged `better-sqlite3` | The packaged arm64 app opens a database and runs `select sqlite_version()`. | Set `npmRebuild: true` and `better-sqlite3: true` in `allowBuilds`, then rerun the check; V1 does not ship until it passes. |
| `safeStorage` across an update | A value stored by v0.0.1 decrypts in v0.0.2. | Decrypt failure asks the user to reconnect; a failure that repeats on a rerun blocks the release. |
| Fuses and ASAR integrity | `@electron/fuses read` matches, `ELECTRON_RUN_AS_NODE=1` does nothing, a tampered `app.asar` refuses to start. | The release is blocked. |
| `app://` CSP header | Playwright sees the CSP on the document response. | Set it in `webRequest.onHeadersReceived`, then in a `<meta>` tag. |
| Playwright on the packaged binary | `executablePath` set to the binary launches the test-flavor app. | The release is blocked. |
| Sentry | A forced native crash sends exactly one event; a normal exit sends none. | Sentry is cut from V1. |
| Auto-update | v0.0.1 updates to v0.0.2 on a signed build, with one feed listing the arm64 zip. | The release is blocked. |
| Entra sign-in | A real Entra tenant completes sign-in, the issued token's `scp` contains `vso.code`, `vso.work` and `vso.profile`, and one commits query succeeds. | If the scopes are missing, register `user_impersonation` and document it as full access. If sign-in itself fails, Azure DevOps is token-only in V1. |
| Azure DevOps live recording | One recording against the test organization passes the contract tests. | Azure DevOps moves to V1.x. |
| `@Me` under a personal access token | The WIQL query returns the user's items. | Work items on the token path are filtered locally by identity email. |
| GitHub OAuth App for Projects | One live read of a user-owned and an organization-owned board with `read:project`. | Projects v2 moves to V1.x. |
| Entitlements | The launch check on the release build passes with only `allow-jit` and `allow-unsigned-executable-memory`. | Add `disable-library-validation`. |
| Release flow | The draft release holds the zip, the dmg and `latest-mac.yml` before publishing. | The release stays a draft. |
| Workspace packaging | The packaged `app.asar` contains every production dependency. | The release is blocked. |
| Coverage metric | At least 95% on the golden set. | The release is blocked. |
| Trust metric | At least 98% on the recorded golden set in CI. | The release is blocked. |

## Evidence and sources

Read on 2026-11-09.

- Anthropic: https://code.claude.com/docs/en/legal-and-compliance (no third-party Claude.ai login or plan-credential routing; Agent SDK products should use API key authentication; an end user may sign in to the unmodified Claude Code binary); https://code.claude.com/docs/en/headless (`claude -p` is the Agent SDK via the CLI; bare mode never reads OAuth credentials or the keychain); https://code.claude.com/docs/en/agent-sdk/overview.
- OpenAI: https://developers.openai.com/siwc/token-sharing-open-source (Sign in with ChatGPT lets an open-source app request the user's plan; the app holds and refreshes OAuth tokens); https://learn.chatgpt.com/docs/non-interactive-mode (`codex exec`).
- GitHub: https://docs.github.com/en/copilot/how-tos/copilot-sdk/auth/authenticate (Copilot SDK OAuth device flow); https://docs.github.com/en/copilot/how-tos/copilot-cli/automate-copilot-cli/run-cli-programmatically.
- Google: https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/ (Gemini CLI remains accessible via paid API keys); Google's Gemini CLI and Antigravity FAQs call piggybacking on CLI sign-in a violation.
- GitHub auth: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/refreshing-user-access-tokens; https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-with-a-github-app-on-behalf-of-a-user; https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens.
- Microsoft: https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/entra-oauth; https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/oauth.
- Native sign-in: https://www.rfc-editor.org/rfc/rfc8252.
- Playwright: https://playwright.dev/docs/api/class-electron (the `nodeCliInspect` fuse must not be false).
- Release pull request: https://docs.github.com/en/actions/concepts/security/github_token and the release-please-action README state that events triggered by `GITHUB_TOKEN`, including the release tag and release pull request, create no new workflow runs. GitHub's ruleset documentation offers the bypass mode "For pull requests only".
- Electron: https://www.electronjs.org/docs/latest/tutorial/fuses; https://www.electronjs.org/docs/latest/api/safe-storage.
- Facts checked locally on 2026-11-09: git 2.56.0 behavior against a hostile repo configuration; Claude Code 2.1.322 flags and a canary run; the npm registry for every fixed version; the `@sentry/electron` 8.3.0 package source for the `MainProcessSession` and `SentryMinidump` integrations, `dataCollection.userInfo` and the `native` event platform.
- Codex, read through the GitHub API on 2026-11-09: openai/codex issue 58662 (closed as not planned on 2026-11-07, `codex exec` 0.154.0) and issue 60069 (open); the installed Codex 0.165.0 `exec --help` lists no tool-off flag. Copilot: github/copilot-cli issue 5306 (created 2026-10-30, an empty `--available-tools` sends all 21 tools) and the programmatic-use page sentence on piped input; the installed Copilot CLI 1.0.108 `--help` lists no hooks-off flag and no status command. No Codex or Copilot tool canary was run, because neither route ships in V1.
