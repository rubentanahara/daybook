# No backend in V1

* Status: accepted
* Deciders: rubentanahara
* Date: 2026-11-09

## Context and Problem Statement

Daybook is a macOS-first Electron app that writes a developer's standup from local git, GitHub, GitHub Projects v2 and Azure DevOps, using the user's own AI. Jira follows in V1.1. When this was written, nothing was built, so there was no existing server or infrastructure to keep or reuse. Does V1 need a server that Daybook operates for auth, AI or updates?

## Decision Drivers

* Ship V1 quickly. Any server work adds to the V1 timeline.
* Secret custody: no Daybook-held secrets or user tokens in transit through Daybook infrastructure.
* Security surface: an Electron app plus a public server is two things to harden and audit.
* Operating burden: a free MIT project with no revenue goal.
* Documented facts, read 2026-11-09: GitHub App device flow needs no `client_secret`, including for refresh ("Required unless the user access token was generated using the device flow"). Microsoft Entra public clients redeem codes with PKCE and no secret. Jira Cloud 3LO needs a `client_secret` for the token exchange and refresh.

## Considered Options

* No backend in V1
* Ship a backend (Hono on Lambda) in V1
* Include Jira in V1 and build the backend for it now
* Route subscription AI requests through a server
* Ship Jira's `client_secret` inside the desktop app
* A GitHub OAuth App with the `repo` scope for all GitHub data
* The legacy Azure DevOps OAuth

## Decision Outcome

Chosen option: "No backend in V1", because every V1 source signs in client-side. Jira is the only known case that needs a server-held secret, and Jira is in V1.1. Hono on AWS Lambda arrives in V1.1 with Jira, under its own ADR. Auto-update comes from GitHub Releases.

### Rules

1. V1 MUST NOT depend on any server that Daybook operates.
2. V1 source sign-in MUST complete client-side as a public client. The app MUST NOT contain any OAuth `client_secret`.
3. Auto-update MUST be served from GitHub Releases, using updater metadata that CI publishes. There is no update server. Auto-update MUST run only on platforms where releases are signed; in V1 that is macOS.
4. Sign-in methods per source are fixed in the decision ledger ("Sources and sign-in"): a GitHub App with device flow for repo data, a GitHub OAuth App limited to `read:project` for Projects v2, and a Microsoft Entra public client for Azure DevOps. Each source except Projects v2 falls back to a user-supplied personal access token stored in `safeStorage` when its sign-in cannot be used: a GitHub App not installable on an owner, or an Azure DevOps organization backed by personal Microsoft accounts. Projects v2 has no token fallback, because fine-grained tokens cannot read user-owned projects. A source whose sign-in and token path both fail, and Projects v2 when its sign-in fails, moves to V1.x. A backend is not built for one source.
5. Native sign-in redirects MUST follow RFC 8252: `http://127.0.0.1:{random port}/{path}`, loopback only, PKCE, a checked `state`, single use, a 2-minute timeout, and no custom URL scheme.

### Positive Consequences

* V1 has no servers to run, patch, monitor or pay for.
* No user token ever passes through Daybook infrastructure.
* The security review covers only the desktop app.

### Negative Consequences

* Jira users wait for V1.1.
* GitHub needs two registrations: a GitHub App and an OAuth App, because fine-grained tokens cannot read user-owned Projects.
* Each GitHub account or organization must install the GitHub App. Owners who do not install it are excluded or use a token.
* Azure DevOps organizations backed by personal Microsoft accounts are token-only.
* The V1.1 backend and Jira OAuth exchange get their own ADR covering the IaC tool and how the secret is held.

### Gates

The following facts exist only after live runs, so each has a gate and a fallback in the decision ledger ("Verification gates"): the Entra sign-in against a real tenant, the Azure DevOps live recording, the GitHub OAuth App reading user-owned and organization-owned boards, and `@Me` under a personal access token.

## Pros and Cons of the Options

### No backend in V1

* Good, because it adds nothing to the V1 timeline and nothing to operate.
* Good, because it keeps secret custody and attack surface to the app alone.
* Bad, because Jira is deferred.

### Ship a backend (Hono on Lambda) in V1

* Good, because the infrastructure would be ready for Jira.
* Bad, because no V1 source needs it. It would be infrastructure built before anything uses it.
* Bad, because it adds a public surface to harden plus ongoing operating burden, all before launch.

### Include Jira in V1 and build the backend for it now

* Good, because Jira users are covered at launch.
* Bad, because it makes a server, secret custody and a fifth source all critical-path work for V1.

### Route subscription AI requests through a server

* Bad, because ADR-0001 rejected it: the app never handles subscription credentials.

### Ship Jira's `client_secret` inside the desktop app

Considered because Jira Cloud 3LO needs a `client_secret` for the token exchange and refresh.

* Good, because it would make Jira possible with no server.
* Bad, because the repo is public from the first commit and the binary can be inspected, so the secret would be extractable, and rotating it would mean shipping a new release.
* Bad, because it breaks rule 2.

### A GitHub OAuth App with the `repo` scope for all GitHub data

* Good, because one registration would cover repos and Projects.
* Bad, because `repo` grants write access to private repositories, far beyond what a read-only standup tool needs.

### The legacy Azure DevOps OAuth

* Bad, because Microsoft accepts no new registrations since April 2025, schedules full deprecation in 2026, and the flow needs an app secret.

## Links

* Related to ADR-0001
* Required by ADR-0003
* To be followed by the V1.1 backend and Jira OAuth exchange ADR (not yet numbered)
* Sources read 2026-11-09: https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/refreshing-user-access-tokens, https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/entra-oauth, https://learn.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/oauth, https://www.rfc-editor.org/rfc/rfc8252
