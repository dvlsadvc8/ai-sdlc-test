# Architecture: AI-SDLC Demo Website

- Status: proposed (Milestone 0.2 planning deliverable; awaiting human approval at Checkpoint B / Gate 1)
- Date: 2026-10-14 (revised the same day after the human answered the open questions)
- Inputs: `docs/requirements.md` (FR-1..FR-11, NFR-1..NFR-10, Q1..Q3, A1..A14; approved hash 61afd0a7be55) and `docs/backlog.md` (E-1..E-6, S-1..S-12), the DevOps engineer's read-only consultation notes, and the human's answers to the architecture open questions (section 14).
- Scope of this document: design only. No application code, no Azure resource, no GitHub setting is created during Milestone 0.2 (requirements "Out of scope", A13). Implementation is Milestone 0.3; provisioning and deployment are Milestone 0.4 (Gate 3).

## 1. Context and constraints

The product is one publicly reachable static web page (FR-1..FR-11) used to exercise the AI-SDLC framework. It has no backend, no accounts, no data submission, and no third-party runtime dependencies (requirements "Out of scope", NFR-5, NFR-4).

Binding constraints:

| Constraint | Source |
| --- | --- |
| Plain HTML, CSS, and JavaScript unless a concrete reason for a framework is recorded here | NFR-6, human direction |
| Hosting on Azure Static Web Apps (SWA) Free plan, default `*.azurestaticapps.net` hostname, HTTPS | NFR-8, A12 |
| Running cost 0 USD/month; any paid tier needs human approval | NFR-9 |
| Repository `dvlsadvc8/ai-sdlc-test` on github.com, GitHub Free, PUBLIC; every file, log, and commit is world-readable | Q1 (revised), NFR-5 |
| Default branch `main` (human answer to architecture open question 1; local `master` is renamed before Milestone 0.3) | section 14 |
| Automated tests run in GitHub Actions on every PR and default-branch push as a REQUIRED status check | Q2, S-11 |
| Infrastructure is defined as code (Bicep) and provisioned by an approval-gated automated pipeline, never by hand-run Azure commands per release; nothing is provisioned in Milestone 0.2 | human answer to open question 2 |
| Performance budget: <= 100 KB compressed initial transfer, zero third-party scripts/fonts, no render-blocking external requests, theme applied before first paint | NFR-4, A7 |
| WCAG 2.2 AA for structure, keyboard, focus, contrast (4.5:1 text, 3:1 large text / UI components / focus indicators) | NFR-1..NFR-3, A6 |
| Supported browsers: current and previous major Chrome, Edge, Firefox, Safari; no IE | A8 |
| Application artifacts live only in this repository, never in the framework repository | NFR-10, S-1 |
| Date formatted with `Intl.DateTimeFormat`, visitor locale, `dateStyle: "long"`, en-US fallback; tests pinned to en-US | Q3, A3 |
| Theme preference in `localStorage` under one key; two-state toggle; no cookies | A5, A11 |
| Graceful degradation without JavaScript | A10 |

Explicit assumptions made by this design (beyond A1..A14):

- **AA-1** Resolved: the default branch is `main` (human, 2026-10-14). Workflows, rulesets, and the OIDC trust all reference `main`.
- **AA-2** Node.js is used only as a development/test toolchain (Playwright). Nothing from `node_modules` is ever deployed.
- **AA-3** The SWA resource is created by the pipeline from Bicep without a linked GitHub repository, so Azure never injects its own workflow or token into the repository.
- **AA-4** "Built page" in S-10 means the files served from `src/`; there is no build step.
- **AA-5** The human holds Owner (or equivalent) rights on the Azure subscription and can create an Entra app registration; these rights are used once, outside the conversation, for the Gate 3 bootstrap (section 5.3). The pipeline identity never has them.

## 2. Proposed architecture

### 2.1 Overview

A single HTML document with one external stylesheet and one external script, served as static files by Azure Static Web Apps over HTTPS. There is no build step, no bundler, no framework, no API, and no server-side logic. The only state is one `localStorage` key in the visitor's browser. Infrastructure is one Bicep-defined resource applied idempotently by an approval-gated GitHub Actions workflow.

```
Visitor browser                      Azure (one resource group)           GitHub (public repo, Free)
+----------------------------+       +----------------------------+       +------------------------------+
| index.html                 | GET   | Static Web App (Free)      |       | main (protected)             |
|  - inline pre-paint theme  |<----->|  /index.html  /styles.css  |<------|  src/  uploaded by           |
|    script (hashed in CSP)  | HTTPS |  /app.js      /favicon.svg | token |  deploy-swa.yml              |
|  - <link> styles.css       |       |  staticwebapp.config.json  |       |  (manual, env "production")  |
|  - <script src=app.js>     |       |   -> globalHeaders         |       |                              |
| localStorage["theme"]      |       |                            |<------|  infra/main.bicep applied by |
+----------------------------+       | (CanNotDelete lock, RBAC:  | OIDC  |  provision.yml               |
                                     |  Contributor on this RG    | +ARM  |  (manual, env "production")  |
                                     |  only, to the pipeline     |       |                              |
                                     |  identity)                 |       | pull requests                |
                                     +----------------------------+       |  test.yml (required check)   |
                                                                          +------------------------------+
```

### 2.2 Components

| Component | Location | Responsibility | Stories |
| --- | --- | --- | --- |
| Page document | `src/index.html` | Semantic structure: `<html lang="en">`, `<title>AI-SDLC Demo</title>`, `<header>` with the single `<h1>AI-SDLC Demo</h1>` and the theme toggle `<button>`, `<main>` with the welcome paragraph and a `<time>` element for the date; `<noscript>` notice for the date. Contains exactly one inline script (the pre-paint theme script). | S-2, S-3, S-4, S-7 |
| Stylesheet | `src/styles.css` | Design tokens as CSS custom properties for light and dark; `color-scheme`; `@media (prefers-color-scheme: dark)` for OS preference when no manual choice; `[data-theme]` overrides for manual choice; fluid single-column layout from 320 px; visible focus indicators; toggle hit target >= 24x24 CSS px. All CSS external (no inline styles) so CSP can use `style-src 'self'`. | S-4, S-5, S-7, S-8 |
| Pre-paint theme script | inline in `<head>` of `index.html`, before the `<link rel="stylesheet">` | Synchronously reads `localStorage.getItem("theme")`; if the value is `"light"` or `"dark"`, sets `document.documentElement.dataset.theme` accordingly. Does nothing otherwise (OS preference is handled by CSS). Wrapped in `try/catch` so blocked storage is harmless. Approximately 10 lines, hashed in the CSP. | S-5, S-6 |
| App script | `src/app.js` (classic script, `defer`) | (1) Computes the local date via `Intl.DateTimeFormat`, fills the `<time>` element's text and `datetime` attribute, and un-hides it. (2) Un-hides the toggle, derives the effective theme (`data-theme` if set, else `matchMedia("(prefers-color-scheme: dark)")`), sets `aria-pressed`, and on click flips the theme, sets `data-theme`, and persists to `localStorage` inside `try/catch`. No other behaviour. | S-3, S-4, S-5, S-6 |
| Favicon | `src/favicon.svg` | Tiny SVG file to avoid a 404 on `/favicon.ico` requests; referenced with `<link rel="icon">`. (Coordinator-accepted default, open question 6.) | S-10 |
| SWA configuration | `src/staticwebapp.config.json` | `globalHeaders` with the security headers listed in section 6, including HSTS (human answer to open question 4); no routes, auth, or fallbacks needed for a single page. Placed in `app_location` as the SWA documentation requires. | S-10, S-12 |
| Test suite | `tests/*.spec.js`, `tests/static-server.mjs`, `playwright.config.js` | Playwright Test with Chromium only, `@axe-core/playwright` for accessibility. A dependency-free Node `http` static server serves `src/` on localhost and applies the same `globalHeaders` from `staticwebapp.config.json`, so CSP violations surface in tests exactly as they would in production. | S-9, S-8, S-10 |
| CI workflow | `.github/workflows/test.yml` | Runs the test suite on `pull_request` and `push` to `main`; the job name is the required status check. Also runs `az bicep build` and `az bicep lint` on `infra/` (no Azure login needed) so infrastructure changes are validated on every PR. | S-11, S-12 |
| Provision workflow | `.github/workflows/provision.yml` | `workflow_dispatch` only, guarded to `main`, `environment: production` (required reviewer). Logs in to Azure with OIDC (no secret), runs `az deployment group what-if` and writes the result to the job summary, then `az deployment group create --mode Incremental` with a fixed deployment name. Idempotent: a re-run with no template change reports `NoChange`. Authored in Milestone 0.3 as part of S-12; first executed only after Gate 3 approval (Milestone 0.4). | S-12 |
| Deploy workflow | `.github/workflows/deploy-swa.yml` | `workflow_dispatch` only, guarded to `main`, `environment: production` (required reviewer). Logs in with OIDC, fetches the SWA deployment token at run time with `az staticwebapp secrets list` (masked, never stored), then uploads `src/` with `Azure/static-web-apps-deploy` and `skip_app_build: true`. Authored in Milestone 0.3; first executed after Gate 3. | S-12 |
| Infrastructure definition | `infra/main.bicep`, `infra/main.bicepparam` | One `Microsoft.Web/staticSites` resource: `sku Free/Free`, `stagingEnvironmentPolicy: 'Disabled'`, `allowConfigFileUpdates: true`, no `repositoryUrl`, deterministic name and tags from the parameters file; output `defaultHostname` only (never secrets). | S-12 |
| Dependabot | `.github/dependabot.yml` | Weekly version updates for `npm` and `github-actions` (coordinator-accepted default, open question 8). | S-10, S-11 |
| Documentation | `README.md`, `docs/` | Purpose, local serve/test commands, Gate 3 bootstrap runbook (section 5.3), architecture, ADRs. | S-1 |

### 2.3 Runtime behaviour

**Page load sequence**

1. Browser parses `<head>`. The inline pre-paint script runs synchronously: if `localStorage.theme` is `"light"` or `"dark"`, it sets `<html data-theme="...">`. Otherwise it leaves the attribute absent.
2. `styles.css` loads (same origin, render-blocking by design; it is the only blocking request and is a few KB). The cascade resolves the effective theme:
   - `html[data-theme="dark"]` -> dark tokens;
   - `html[data-theme="light"]` -> light tokens;
   - no attribute and `@media (prefers-color-scheme: dark)` -> dark tokens;
   - otherwise -> light tokens (the default declarations).
   Because the attribute is set before the stylesheet is applied and the OS case is handled purely in CSS, the first paint is already in the correct theme (NFR-4 "no flash", S-5, S-6).
3. Body renders: `<h1>`, welcome text (visible immediately, FR-3), the date `<time>` (initially `hidden`), the toggle `<button>` (initially `hidden`), and a `<noscript>` notice next to the date.
4. `app.js` (deferred) runs after parsing: fills and reveals the date; reveals and wires the toggle.

**Theme resolution order (FR-7, FR-8, A5, A11)**: stored manual choice > OS `prefers-color-scheme` > light. Once a manual choice is stored, OS changes are ignored until the visitor toggles again. Before any manual choice, a live OS change is followed automatically because the CSS media query is live (permitted by A11; coordinator-accepted default, open question 5).

**Toggle (FR-6, NFR-2)**: a native `<button type="button">` with the stable accessible name "Dark theme" and `aria-pressed` reflecting whether the dark theme is active (toggle-button pattern; coordinator-accepted default, open question 3). Native `<button>` gives Tab reachability and Enter/Space activation for free. Clicking flips `data-theme` between `light` and `dark` (always writing an explicit value, so the result is deterministic regardless of OS preference) and writes the same value to `localStorage["theme"]`. If storage throws (blocked, quota, private mode), the toggle still works for the current view and the next load falls back to OS preference (A5, S-6).

**Date (FR-4, FR-5, A3, A4)**: computed once per load from `new Date()` on the device clock. Text: `new Intl.DateTimeFormat(locale, { dateStyle: "long" }).format(now)` where `locale` is `navigator.language` falling back to `"en-US"` (also on `RangeError`). The `<time datetime>` attribute is built from local `getFullYear()/getMonth()/getDate()` as `YYYY-MM-DD` so it reflects the device time zone, not UTC. No live update after load (A4).

**JavaScript disabled (A10)**: heading and welcome render normally; the OS theme is still honoured by CSS; the date element stays hidden and the `<noscript>` notice explains that the date needs JavaScript; the toggle stays hidden so no dead control is shown.

### 2.4 Repository layout

```
ai-sdlc-test/
  .github/
    dependabot.yml
    workflows/
      test.yml                 CI: Playwright tests + Bicep build/lint, required status check (S-11)
      provision.yml            Manual, environment-gated Bicep what-if + deploy (S-12, Milestone 0.4)
      deploy-swa.yml           Manual, environment-gated content upload to SWA (S-12, Milestone 0.4)
  .ai-sdlc/                    AI-SDLC coordinator state (not part of the application)
  docs/
    requirements.md            approved
    backlog.md                 approved
    architecture.md            this document
    adr/ADR-000N-*.md          decisions
  infra/
    main.bicep                 SWA resource, SKU Free, staging environments disabled (ADR-0007)
    main.bicepparam            deterministic name, location, tags
  src/                         the deployable site = SWA app_location; nothing else is deployed
    index.html
    styles.css
    app.js
    favicon.svg
    staticwebapp.config.json
  tests/
    static-server.mjs          dependency-free local server applying SWA globalHeaders
    page.spec.js               S-2 structure, title, welcome
    date.spec.js               S-3
    theme.spec.js              S-4, S-5, S-6
    layout.spec.js             S-7
    a11y.spec.js               S-8 (axe, both themes, focus ring contrast)
    budget.spec.js             S-10 size, requests, storage keys, CSP hash
  playwright.config.js
  package.json / package-lock.json   dev dependencies only
  .nvmrc                       pins the current Node.js LTS used locally and in CI
  .gitignore                   node_modules, playwright-report, test-results
  README.md
```

## 3. Technology choices with rationale

| Area | Choice | Rationale | ADR |
| --- | --- | --- | --- |
| Page | HTML5 + CSS custom properties + one classic ES2017+ script, no framework, no bundler, no transpiler | The page has three static strings, one computed string, and one boolean of state. Any framework would add more bytes than the whole page, a build step, a dependency tree to update, and nothing the requirements need. All target browsers (A8) support the features used. Satisfies NFR-6 and keeps NFR-4 trivially. | ADR-0001 |
| Hosting | Azure Static Web Apps, Free plan, default hostname, no custom domain | Mandated by NFR-8/A12; Free plan gives HTTPS with managed certificates, global distribution, 100 GB/month bandwidth, 250 MB/app, custom headers via `staticwebapp.config.json`. No SLA, acceptable for a demo. Cost 0. | ADR-0002 |
| Tests | Playwright Test, Chromium only, `@axe-core/playwright`, current Node LTS pinned in `.nvmrc` | NFR-7 requires headless verification of date (clock and time-zone control), theme (`colorScheme` emulation, storage across reload), responsive layout (viewport and layout measurement), and S-8 requires an automated WCAG audit. Playwright provides all of these natively; axe adds the audit. Chromium only keeps CI at 3-5 minutes. | ADR-0003 |
| Security headers | `staticwebapp.config.json` `globalHeaders`; CSP with a SHA-256 hash for the single inline script; HSTS `max-age=31536000` | A strict CSP without `'unsafe-inline'` is achievable because the only inline code is a tiny, stable script. Hashes work on a static host where nonces cannot. An automated test recomputes the hash so drift fails CI. HSTS approved by the human. | ADR-0004 |
| Theme state | Single `localStorage` key `theme` with values `light`/`dark`; OS preference via CSS media query; two-state `<button aria-pressed>` named "Dark theme" | Matches A5 and A11 exactly; CSS handling of OS preference removes the need for JS in the no-choice case and makes first paint correct without any script in that case. | ADR-0005 |
| CI/CD and cloud identity | Three workflows (test, provision, deploy); required check; manual environment-gated provision and deploy; Entra app registration with a federated credential (OIDC, no client secret) bound to the `production` environment; Contributor scoped to one resource group; deployment token fetched at run time, never stored | Separates the untrusted PR path (read-only token, no cloud access) from the privileged paths (manual, reviewed, `main` only). No long-lived credential exists anywhere in GitHub. Avoids the portal-generated SWA workflow. | ADR-0006 |
| Infrastructure | Bicep (`infra/main.bicep`), ARM incremental mode, deterministic names, applied only by `provision.yml` after what-if review and Gate 3 approval | Human requirement: automated, approval-gated provisioning; idempotent re-apply; unauthorized changes prevented by environment reviewer, branch protection, RG-scoped RBAC, and a CanNotDelete lock. Zero cost. | ADR-0007 |

## 4. Alternatives considered

| Alternative | Why not |
| --- | --- |
| Framework (React/Vue/Svelte) or static-site generator (Astro/Eleventy) | Adds build step, dependencies, and bytes without a requirement that needs them; NFR-6 forbids it absent a concrete reason. |
| Tailwind or any CSS framework | Needs a build step or ships far more CSS than the page; a few dozen lines of hand-written CSS suffice. |
| TypeScript for `app.js` | Would need a build/transpile step for ~40 lines of code; JSDoc comments give the same documentation value. |
| GitHub Pages | Not Azure (NFR-8). |
| Azure Storage static website (+ optional CDN) | Small non-zero cost, weaker managed-TLS story, no header configuration without a CDN. |
| Azure SWA Standard | Adds cost for SLA and features (networking, custom auth) the page does not need. |
| Cypress / Puppeteer / jsdom+Vitest | Cypress is heavier and weaker on colour-scheme and clock emulation; Puppeteer lacks a test runner and axe integration is manual; jsdom has no layout engine so S-7 and S-8 cannot be verified. |
| Multi-browser Playwright | Triples CI time and downloads for a page with no engine-specific code; one config line enables it later. |
| CSP with `'unsafe-inline'` | Defeats CSP's main purpose; unnecessary because the inline script is stable and hashable. |
| External `theme-init.js` instead of inline script | Allows `script-src 'self'` with no hash, but adds a blocking request before first paint. |
| Theme stored in a cookie / `sessionStorage` / three-state toggle | Excluded by A5, FR-8, and A11 respectively. |
| Portal-generated SWA workflow (push + PR triggers, preview environments) | Deploys every PR and exposes the deployment token to PR jobs in a public repo; rejected. |
| Human applies Bicep manually with `az deployment group create` at each release | Rejected by the human (open question 2): the framework must automate provisioning after approval; manual commands per release are not acceptable. |
| Deployment token stored as a GitHub secret | Works, but requires the human to copy a long-lived secret by hand and rotate it manually. Superseded by fetching the token at run time with the OIDC identity (`az staticwebapp secrets list`), so no credential is ever stored in GitHub. |
| Client-secret service principal | Long-lived secret in GitHub; OIDC federated credentials remove it entirely. Rejected. |
| User-assigned managed identity in the application resource group | Contributor on that RG includes `Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials/write`, so a compromised approved run could add a federated credential trusting another repository (privilege escalation). An Entra app registration has no ARM footprint and cannot be altered by RG Contributor. A UAMI in a separate identity-only RG is an acceptable alternative. |
| "Website Contributor" built-in role | Verified against the built-in role definition: its Web actions are `Microsoft.Web/sites/*`, `serverFarms/read`, `serverFarms/join`, `certificates/*`; it does not include `Microsoft.Web/staticSites/*`, so it cannot create or read the Static Web App. |
| Subscription-scoped role assignment | Grants far more than needed; the identity is assigned only on the single resource group. Rejected. |
| Pipeline applies the CanNotDelete lock | Creating locks requires `Microsoft.Authorization/locks/*`, which Contributor lacks and which should not be granted to the pipeline. The lock is applied once by the human during bootstrap. |
| Terraform / Deployment Stacks for one resource | Adds state management or a newer construct for a single resource; Bicep incremental deployments are idempotent and sufficient. |
| Secret scanning via third-party action | GitHub secret scanning and push protection are free for public repositories and need no workflow. |

## 5. Infrastructure and deployment implications (planning only)

No resource is created in Milestone 0.2. Milestone 0.3 authors the files below; Milestone 0.4 (Gate 3) performs the one-time bootstrap and the first provisioning and deployment runs.

### 5.1 Azure target

- One resource group (name decided at Gate 3; convention `rg-ai-sdlc-demo`) in a Free-plan region (DevOps recommends East US 2; decided at Gate 3).
- One Static Web App, SKU Free, from `infra/main.bicep`: deterministic name (for example `stapp-ai-sdlc-demo`), `stagingEnvironmentPolicy: 'Disabled'` (no PR preview environments can be created even by mistake), `allowConfigFileUpdates: true`, no `repositoryUrl` or `branch` (no Azure-injected workflow or token), tags for owner and purpose. Output: `defaultHostname` only.
- Idempotency: ARM incremental mode, fixed resource names, fixed deployment name (for example `swa-infra`) so the deployment history does not grow (a CanNotDelete lock on the RG prevents automatic history cleanup; the 800-deployment limit is therefore avoided by reusing the name). A re-run with an unchanged template reports `NoChange` in what-if and performs no modification.
- A `CanNotDelete` management lock on the resource group, applied by the human at bootstrap (section 5.3). Incremental deployments never delete, so the lock never interferes with provisioning; `ReadOnly` is not used because it would block deployments.

### 5.2 Pipeline identity and authorization

- Identity: one Microsoft Entra app registration (service principal) with no client secret and one federated identity credential: issuer `https://token.actions.githubusercontent.com`, audience `api://AzureADTokenExchange`, subject bound to this repository's `production` environment. Because the repository was created after 15 July 2026, GitHub issues the immutable subject format that includes owner and repository IDs (`repo:dvlsadvc8@<owner-id>/ai-sdlc-test@<repo-id>:environment:production`); the DevOps agent confirms the exact value from GitHub's documentation and a read-only `gh api` lookup of the IDs before the human enters it, and the first `azure/login` run verifies it.
- Role: built-in `Contributor` assigned at the scope of the single resource group only. Verified: this is the narrowest built-in role that includes `Microsoft.Web/staticSites/*` (needed for create/update and for `listSecrets`) plus `Microsoft.Resources/deployments/*` (needed for what-if and deployment); it excludes `Microsoft.Authorization/*` so the pipeline cannot change RBAC or locks. Optional later hardening at zero cost: a custom role limited to `Microsoft.Web/staticSites/*`, `Microsoft.Resources/deployments/*`, `Microsoft.Resources/subscriptions/resourceGroups/read`, `Microsoft.Authorization/*/read`.
- GitHub Environment `production`: required reviewer (the human), deployment branches limited to `main`, environment secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` (identifiers, stored as secrets per Microsoft guidance for public repositories), environment variables `AZURE_RESOURCE_GROUP`, `AZURE_LOCATION`, `SWA_NAME`. No deployment token is stored anywhere in GitHub.

### 5.3 One-time bootstrap (human, outside the conversation, Milestone 0.4)

These steps need Owner/User Access Administrator rights on the subscription and Entra application-creation rights, which the pipeline must never have. They are performed once by the human and documented as a runbook in the README; the AI agents only read the results.

1. Create the resource group in the chosen region.
2. Create the Entra app registration and service principal; add the federated identity credential bound to `repo:... :environment:production`; do not create a client secret.
3. Assign `Contributor` to the service principal on the resource group only.
4. Apply the `CanNotDelete` lock on the resource group.
5. In GitHub: create the `production` environment with the required reviewer and branch restriction; add the three identifier secrets and three variables; confirm the `main` ruleset requires the `test` check and a pull request.

### 5.4 Workflows

- **test.yml** (S-11): `pull_request` and `push` to `main`; `permissions: contents: read`; `ubuntu-latest`; checkout, setup-node from `.nvmrc`, `npm ci`, `npx playwright install --with-deps chromium`, `npx playwright test`, `az bicep build --file infra/main.bicep` and `az bicep lint` (no login), upload `playwright-report` only on failure. Actions pinned to full SHAs. 3-5 minutes; unmetered on a public repo.
- **provision.yml** (S-12, Milestone 0.4): `workflow_dispatch`; `if: github.ref == 'refs/heads/main'`; `environment: production`; `permissions: contents: read, id-token: write`; steps: checkout, `azure/login` (OIDC, client-id/tenant-id/subscription-id from environment secrets), `az deployment group what-if --mode Incremental --template-file infra/main.bicep --parameters infra/main.bicepparam` written to `$GITHUB_STEP_SUMMARY`, `az deployment group create --name swa-infra --mode Incremental ...`, then a second what-if asserting `NoChange` as an idempotency check. No secrets are printed; the template has no secure outputs.
- **deploy-swa.yml** (S-12, Milestone 0.4): `workflow_dispatch`; `main` only; `environment: production`; `permissions: contents: read, id-token: write`; steps: checkout, run the test suite, `azure/login` (OIDC), `token=$(az staticwebapp secrets list -n $SWA_NAME -g $AZURE_RESOURCE_GROUP --query properties.apiKey -o tsv)` immediately followed by `echo "::add-mask::$token"`, then `Azure/static-web-apps-deploy` with `action: upload`, `app_location: src`, `output_location: ""`, `skip_app_build: true`, `skip_api_build: true`, `azure_static_web_apps_api_token: $token`. No `pull_request` trigger and no `close_pull_request_job`. If OIDC-only content deployment is confirmed at Gate 3, the token step is removed.

### 5.5 Gate 3 approval package (prepared by the DevOps agent, Milestone 0.4)

Before the human approves provisioning, the DevOps agent presents: the Bicep source and lint result; the what-if output run locally against the bootstrapped (still empty) resource group using the human's existing read-only Azure CLI session (what-if changes nothing but needs the same permissions as a deployment, so it cannot be run by an anonymous or Reader identity); the expected result `1 to create` for `Microsoft.Web/staticSites/<name>` and nothing else; the cost statement (SWA Free: 0 USD; resource group, lock, app registration, role assignment: 0 USD); and the exact subject claim and role assignment to be used. On the human's explicit approval, the human dispatches `provision.yml` and approves the environment deployment; the job re-runs what-if, applies, and verifies `NoChange`. Then `deploy-swa.yml` is dispatched and approved the same way.

### 5.6 Local development and rollback

- `npm test` runs Playwright, which starts `tests/static-server.mjs` via `webServer`; `npm run serve` starts the same server for manual inspection. No global tools beyond Node.js (and Azure CLI only for Bicep lint).
- Content rollback: dispatch `deploy-swa.yml` on a previous `main` commit, or revert on `main` and dispatch.
- Infrastructure rollback: revert the Bicep change on `main` via PR (required check), then dispatch `provision.yml`; what-if shows the reversal before apply.

## 6. Security considerations

Threat model: a public, read-only brochure page with no server, no secrets at runtime, and a world-readable repository. The assets to protect are (1) the Azure subscription and resource group (unauthorized infrastructure changes), (2) the deployment path (unauthorized content), (3) the integrity of what is served, and (4) visitors' browsers (XSS, clickjacking).

Controls:

- **No long-lived credentials anywhere** (NFR-5, S-1, S-10, S-12): the pipeline authenticates with short-lived OIDC tokens exchanged for Azure tokens; there is no client secret and no stored deployment token. The three identifiers stored as environment secrets (client, tenant, subscription IDs) are not secrets in the cryptographic sense but are kept out of files anyway. Secret scanning and push protection are enabled; a test greps `src/` for token-like patterns. The repository must never contain files from `C:/repos/ai-sdlc`.
- **Unauthorized infrastructure changes are prevented by five independent layers**: (a) the federated credential is bound to the `production` environment, so only a job running under that environment can obtain an Azure token; (b) the environment requires a human reviewer and only `main` may deploy to it; (c) workflow files live on `main`, which is protected by a ruleset requiring a PR and the `test` check, so pipeline code cannot be changed silently; (d) Azure RBAC grants Contributor on one resource group only, so even an approved run cannot touch anything else in the subscription or change RBAC/locks; (e) the `CanNotDelete` lock, applied by the human, protects the resource group and its contents from deletion by any principal.
- **Least-privilege CI**: `permissions: contents: read` on the test job; PR jobs have no environment, no secrets, and no `id-token`; third-party actions pinned to full SHAs; Dependabot for `github-actions` and `npm`.
- **Deployment token handling**: fetched only inside an approved `production` job, masked in logs immediately, used once, never written to disk or to a GitHub secret. It remains rotatable in Azure (`az staticwebapp secrets reset-api-key`) if ever suspected leaked.
- **Response headers** via `globalHeaders` (verified post-deploy at Gate 3 with `curl -I`):
  - `Content-Security-Policy: default-src 'none'; script-src 'self' 'sha256-<hash-of-inline-theme-script>'; style-src 'self'; img-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'`
  - `Strict-Transport-Security: max-age=31536000` (human-approved; no `includeSubDomains`, no `preload`, since the hostname is Azure's)
  - `X-Content-Type-Options: nosniff`
  - `Referrer-Policy: no-referrer`
  - `Permissions-Policy: camera=(), microphone=(), geolocation=(), interest-cohort=()`
- **Client-side data**: only `localStorage["theme"]` with a fixed enumerated value; nothing personal (NFR-5, S-6, S-10).
- **Supply chain**: zero runtime dependencies; dev dependencies limited to `@playwright/test` and `@axe-core/playwright` with a committed lockfile.
- **Public identifiers**: `.ai-sdlc/config.json` in this repository already contains the Azure subscription and tenant IDs and will be world-readable once the repository is public. These are identifiers, not credentials, and by themselves grant no access; the coordinator should confirm that publishing them is acceptable or move them out of the tracked tree (risk R13).

## 7. Cost assumptions

| Item | Plan | Monthly cost | Headroom / note |
| --- | --- | --- | --- |
| Azure Static Web Apps | Free | 0 USD | 100 GB bandwidth/subscription; the page is ~10 KB |
| TLS certificate | included | 0 USD | auto-renewing |
| Azure resource group, management lock, role assignment | n/a | 0 USD | no charge |
| Microsoft Entra app registration + federated credential | included in tenant | 0 USD | no licence needed |
| Azure Resource Manager deployments / what-if | n/a | 0 USD | control-plane operations are free |
| GitHub repository | Free, public | 0 USD | |
| GitHub Actions | standard runners, public repo | 0 USD | unmetered |
| GitHub Environments, rulesets, Dependabot, secret scanning | public repo | 0 USD | included |

Cost can become non-zero only by (a) moving SWA to Standard, (b) making the repository private while keeping required checks (needs GitHub Pro), (c) adding custom domains beyond two or any other Azure service. Each requires human approval (NFR-9). The Gate 3 package restates the 0 USD estimate explicitly.

## 8. Risks and mitigations

| # | Risk | Likelihood / impact | Mitigation |
| --- | --- | --- | --- |
| R1 | `Azure/static-web-apps-deploy@v1` requires a deployment token (OIDC-only unconfirmed) | Confirmed limitation / low (was medium) | Token is no longer stored: it is fetched inside the approved job with the RG-scoped identity, masked, and used once. Re-check OIDC-only support at Gate 3 and drop the step if confirmed. |
| R2 | CSP hash drifts when someone edits the inline script | Medium / medium | `budget.spec.js` recomputes the SHA-256 and compares; the local test server applies the same headers so a mismatch fails CI before merge. |
| R3 | SWA alters some `globalHeaders` on Free | Low / medium | Verify with `curl -I` after the first deployment; config-only fix. |
| R4 | Public repository exposes any accidentally committed secret forever | Low / high | No credentials exist by design (OIDC only); push protection and secret scanning enabled; deployment token resettable in seconds. |
| R5 | Default branch mismatch | Resolved | Human decided `main`; local `master` renamed before Milestone 0.3. |
| R6 | Chromium-only testing misses an engine difference | Low / low | Standards-only features; manual smoke in another browser at Gate 3; one-line config to add engines. |
| R7 | Playwright fixed clock may not persist across `page.reload()` | Medium / low | Re-call `setFixedTime` before reload; fallback `addInitScript`. Verified during S-3. |
| R8 | axe cannot measure focus-ring contrast | Medium / low | Palette pre-computed; `a11y.spec.js` computes outline/background ratio with a WCAG helper. |
| R9 | Small layout shift when JS reveals date/toggle | Certain / negligible | Reserve height with `min-height`. |
| R10 | GitHub changes public-repo free features | Low / medium | Re-check at each gate; fallback advisory-only CI. |
| R11 | Non-en-US visitors see a different long-date format | Certain / none | Accepted by Q3/A3; tests pin en-US. |
| R12 | Federated credential subject mismatch (new immutable subject format with owner/repo IDs for repositories created after 15 July 2026) causes `azure/login` to fail at Gate 3 | Medium / low | DevOps agent derives the exact subject from GitHub documentation and a read-only `gh api` lookup of the owner and repository IDs; the human enters it; first login failure is informative and harmless (nothing is provisioned until login succeeds). |
| R13 | Azure subscription and tenant IDs in `.ai-sdlc/config.json` become world-readable | Certain / low | Identifiers only; grant no access. Coordinator to confirm acceptance or untrack the file. All access is gated by the environment-bound federated credential and RG-scoped RBAC. |
| R14 | Contributor on the resource group is broader than strictly needed (can create any resource type inside the RG) | Low / low | Scope is one empty RG; cost alerts/budget at 0 USD can be added by the human; optional zero-cost custom role listed in section 5.2 as later hardening. Lock and RBAC exclusions prevent escalation beyond the RG. |
| R15 | Deployment token briefly present in the job's memory and environment | Low / low | Masked immediately with `::add-mask::`; never echoed, written, or passed between jobs; job runs only after human approval on protected code. |
| R16 | CanNotDelete lock blocks automatic deployment-history cleanup (800-entry limit) | Low / low | Fixed deployment name `swa-infra` reuses one history entry. |
| R17 | Pre-approval what-if cannot be run by a read-only identity (same permissions as deployment) | Certain / low | Gate 3 what-if is run locally by the DevOps agent under the human's existing Azure CLI session (read-only operation, no change); the pipeline repeats what-if in the job summary before applying. |

Starting palette (to be confirmed by the automated checks; values are a design proposal, not binding):

| Token | Light | Dark | Lowest relevant ratio |
| --- | --- | --- | --- |
| background | `#ffffff` | `#121212` | - |
| text | `#1a1a1a` | `#ececec` | ~17:1 light, ~16:1 dark |
| toggle border | `#595959` | `#a6a6a6` | ~7:1 vs background |
| focus ring (2 px solid outline, 2 px offset) | `#005fcc` | `#7cc4ff` | ~6:1 light, ~10:1 dark |

## 9. Quality attributes mapped to NFRs

| NFR | Design response | Verified by |
| --- | --- | --- |
| NFR-1 structure | `lang="en"`, `<header>`/`<main>`, single `<h1>`, `<time>`, real text | `page.spec.js`, axe rules (`a11y.spec.js`) |
| NFR-2 keyboard/focus | Native `<button>`; CSS `:focus-visible` outline in both themes | `theme.spec.js` (Tab, Enter, Space), `a11y.spec.js` |
| NFR-3 contrast | Token palette above; 2 px outline with offset | axe `color-contrast` in both themes; outline ratio helper |
| NFR-4 performance | 3 same-origin files, ~10 KB total, no fonts, CSS is the only blocking request, pre-paint theme | `budget.spec.js`, structural pre-paint check |
| NFR-5 security/privacy | No inputs, no network calls, one storage key, strict CSP + HSTS, no credentials anywhere (OIDC) | `budget.spec.js`, workflow review, GitHub secret scanning |
| NFR-6 maintainability | Three source files, one Bicep file, commented where non-obvious, README with commands and bootstrap runbook | S-1 review |
| NFR-7 automated verification | Playwright suite, headless, exit code, HTML report; Bicep build/lint | `test.yml` |
| NFR-8 hosting | `src/` is the SWA app root; `staticwebapp.config.json`; Bicep + workflows authored in 0.3, executed in 0.4 | S-12 review; Gate 3 verification |
| NFR-9 cost | All free tiers; no paid action, identity, or service | Section 7; Gate 3 package |
| NFR-10 separation | Everything in `ai-sdlc-test`; `.gitignore` and S-1 scan | S-1 review |

## 10. Testing strategy

All tests are Playwright Test specs run headlessly against the local static server (same headers as production). Each test sets its own emulation via `test.use` or a fresh context.

| Spec | Story | Technique |
| --- | --- | --- |
| `page.spec.js` | S-2 | Exactly one `h1` with text "AI-SDLC Demo"; `document.title`; welcome text in viewport at 320 px with no interaction (bounding box inside viewport); `lang`, landmarks, heading order; no `<img>` carrying text. |
| `date.spec.js` | S-3 | `page.clock.setFixedTime()` + `locale: 'en-US'` + `timezoneId` (e.g. `Pacific/Honolulu` at 23:30 local on 14 Oct where UTC is already 15 Oct) -> expects "October 14, 2026" and `datetime="2026-10-14"`; advance fixed time one day, reload, expect new date; static source check that `index.html` contains no date-like literal. |
| `theme.spec.js` | S-4, S-5, S-6 | S-4: click toggles `data-theme` and computed `background-color`; URL unchanged and a `window` marker set before the click survives (no navigation); `aria-pressed` flips; keyboard Tab focus visible (outline style), Enter and Space toggle. S-5: fresh contexts with `colorScheme` `dark`/`light`/`no-preference` and empty storage -> expected theme; structural pre-paint check (inline script precedes stylesheet in `<head>`; `data-theme` present at `domcontentloaded` when storage holds a value). S-6: toggle, `reload()`, theme persists in both OS-preference directions; `addInitScript` making `localStorage` throw -> no console errors, toggle still works, reload falls back to OS preference; `Object.keys(localStorage)` equals `["theme"]`. |
| `layout.spec.js` | S-7 | Viewports 320x568, 768x1024, 1280x800: `scrollWidth <= innerWidth`; bounding boxes of the four primary elements within viewport; toggle `boundingBox()` >= 24x24 and `isVisible()`; no ancestor with `overflow: hidden` clipping the toggle. |
| `a11y.spec.js` | S-8 | `AxeBuilder.withTags(['wcag2a','wcag2aa','wcag22aa'])` -> zero violations in light and in dark; focused-outline contrast ratio >= 3:1 computed from CSS colours; accessible name "Dark theme" present on the toggle. |
| `budget.spec.js` | S-10 | Capture all requests on load: all same-origin, none third-party, none fonts; sum of response body sizes <= 100 KB (uncompressed is stricter than compressed); `Object.keys(localStorage)` after toggle; grep `src/` for secret patterns; CSP hash in `staticwebapp.config.json` equals SHA-256 of the inline script extracted from `index.html`; HSTS and the other headers present on the served response; no console errors or CSP violations during load and toggle. |
| Bicep | S-12 | `az bicep build` and `az bicep lint` in `test.yml`; what-if `NoChange` assertion after apply in `provision.yml` (Milestone 0.4). |
| CI | S-11 | `test.yml` runs the whole suite; the job name `test` is the required status check. |
| S-12 | review | Workflows, Bicep, and SWA config reviewed for credential hygiene; functional verification post-deploy at Gate 3 (HTTPS, headers, four elements, toggle). |

Mutation check (S-9 third criterion) is a review-time step: hardcode a date or remove `localStorage.setItem` locally and observe at least one failing test; documented in the README rather than automated.

## 11. Dependencies

| Dependency | Kind | Version policy | Shipped to visitors |
| --- | --- | --- | --- |
| Node.js current LTS | dev toolchain | pinned in `.nvmrc`, used by `actions/setup-node` | no |
| `@playwright/test` | devDependency | lockfile; Dependabot weekly | no |
| `@axe-core/playwright` | devDependency | lockfile; Dependabot weekly | no |
| Azure CLI + Bicep CLI | CI/tooling (present on `ubuntu-latest`) | runner image | no |
| `actions/checkout`, `actions/setup-node`, `actions/upload-artifact`, `azure/login`, `Azure/static-web-apps-deploy` | GitHub Actions | pinned to full SHA; Dependabot weekly | no |
| Azure Static Web Apps Free; Azure Resource Manager; Microsoft Entra federated identity | platform | n/a | SWA serves the page |
| Browser APIs: `Intl.DateTimeFormat` (`dateStyle`), `matchMedia`, `localStorage`, CSS custom properties, `prefers-color-scheme`, `:focus-visible` | runtime | all supported by A8 browsers | n/a |

## 12. Traceability: stories to components

| Story | Components | Tests |
| --- | --- | --- |
| S-1 | repository layout, `README.md` (incl. bootstrap runbook), `.gitignore`, `package.json`, `.nvmrc` | review |
| S-2 | `index.html` | `page.spec.js`, `a11y.spec.js` |
| S-3 | `app.js` (date), `index.html` `<time>` + `<noscript>` | `date.spec.js` |
| S-4 | `app.js` (toggle), `styles.css` tokens, `index.html` `<button>` | `theme.spec.js` |
| S-5 | `styles.css` media query, inline pre-paint script | `theme.spec.js` |
| S-6 | inline pre-paint script, `app.js` persistence | `theme.spec.js` |
| S-7 | `styles.css` layout | `layout.spec.js` |
| S-8 | `styles.css` palette and focus styles, `index.html` semantics | `a11y.spec.js` |
| S-9 | `tests/`, `playwright.config.js`, `static-server.mjs` | the suite itself |
| S-10 | `staticwebapp.config.json`, asset sizes | `budget.spec.js` |
| S-11 | `.github/workflows/test.yml`, `main` ruleset (human) | CI run |
| S-12 | `infra/main.bicep`, `infra/main.bicepparam`, `.github/workflows/provision.yml`, `.github/workflows/deploy-swa.yml`, `staticwebapp.config.json`, bootstrap runbook | Bicep lint in CI; review; Gate 3 |

## 13. Decisions

| ADR | Title | Status |
| --- | --- | --- |
| [ADR-0001](adr/ADR-0001-plain-static-stack.md) | Plain HTML, CSS, and JavaScript with no framework and no build step | proposed |
| [ADR-0002](adr/ADR-0002-azure-static-web-apps-free.md) | Host on Azure Static Web Apps Free plan | proposed |
| [ADR-0003](adr/ADR-0003-playwright-chromium-axe.md) | Playwright Test with Chromium and axe-core for automated verification | proposed |
| [ADR-0004](adr/ADR-0004-csp-hashed-inline-theme-script.md) | Strict CSP with a SHA-256 hash for the inline pre-paint theme script, plus HSTS | proposed |
| [ADR-0005](adr/ADR-0005-theme-persistence-localstorage.md) | Theme persistence in one localStorage key, OS preference via CSS, two-state toggle | proposed |
| [ADR-0006](adr/ADR-0006-ci-cd-and-deployment-auth.md) | CI/CD workflows, required check, OIDC federated identity scoped to one resource group, approval-gated provision and deploy | proposed |
| [ADR-0007](adr/ADR-0007-swa-resource-definition-bicep.md) | SWA resource defined in Bicep and provisioned idempotently by an approval-gated pipeline in Milestone 0.4 | proposed |

## 14. Open questions and their resolution

All architecture open questions raised on 2026-10-14 have been answered; none remain open for Checkpoint B.

| # | Question | Resolution | Decided by |
| --- | --- | --- | --- |
| 1 | Default branch name | `main`; local `master` renamed before Milestone 0.3 | Human (2026-10-14) |
| 2 | How the SWA resource is defined and provisioned | Bicep (`infra/main.bicep`) as infrastructure as code. Manual `az deployment group create` by the human is REJECTED. Milestone 0.2: generate the Bicep design and record the decision, provision nothing. Milestone 0.4: approval-gated automated provisioning: the DevOps agent validates the Bicep, presents what-if results and the estimated cost (0 USD, SWA Free), obtains explicit Gate 3 approval, then the pipeline applies it with appropriately scoped credentials. Configuration must be re-appliable idempotently and unauthorized infrastructure changes prevented. No unnecessary paid services. (ADR-0006, ADR-0007) | Human (2026-10-14) |
| 3 | Toggle accessible-name pattern | Stable name "Dark theme" with `aria-pressed` | Coordinator-accepted default (not a human-stated preference) |
| 4 | HSTS header | `Strict-Transport-Security: max-age=31536000`, no `includeSubDomains` | Human (2026-10-14) |
| 5 | Live OS preference before a manual choice | Follow live via the CSS media query | Coordinator-accepted default |
| 6 | Favicon | Ship a tiny `favicon.svg` | Coordinator-accepted default |
| 7 | Node.js version for tooling | Current LTS at implementation time, pinned in `.nvmrc` | Coordinator-accepted default |
| 8 | Dependabot cadence | Weekly | Coordinator-accepted default |

Deferred to Gate 3 (Milestone 0.4), not blocking Checkpoint B: Azure region (DevOps recommends East US 2), resource group and SWA names, the exact federated-credential subject string (R12), the Environment reviewer identity, and whether OIDC-only content deployment has become available (R1).
