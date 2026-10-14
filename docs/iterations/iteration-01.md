# Iteration 1: AI-SDLC Demo Website, complete first increment

Repository: dvlsadvc8/ai-sdlc-test (public, default branch `main`)
Project: https://github.com/users/dvlsadvc8/projects/2
Requirements approval: 61afd0a7be55 | Architecture approval: 54b6dbfc4f53
Status: proposed (Gate 1 approval pending)

## Objectives

1. Deliver the complete, verifiable AI-SDLC Demo Website as specified in `docs/requirements.md` (FR-1 to FR-11, NFR-1 to NFR-10) following the approved architecture (ADR-0001 to ADR-0007).
2. Prove every functional and non-functional requirement with automated, headless tests that run as a required status check on every pull request.
3. Author (but not execute) the infrastructure and deployment artifacts so that Milestone 0.4 can provision and deploy through the approval-gated pipeline without further design work.

The application is a single static page, so all 12 stories form one coherent increment. Splitting them would create partial pages that cannot be independently verified against the acceptance criteria.

## Stories

| ID | Issue | Title | Priority |
| --- | --- | --- | --- |
| S-1 | #7 | Repository scaffold and project structure | 1 |
| S-2 | #8 | Heading, tab title, and welcome message | 1 |
| S-3 | #9 | Display the visitor's local date on page load | 2 |
| S-4 | #10 | Theme toggle switches themes without reload | 2 |
| S-5 | #11 | Honour the operating-system theme preference on first visit | 3 |
| S-6 | #12 | Remember the manually selected theme across reloads | 3 |
| S-7 | #13 | Responsive layout from 320 px upward | 3 |
| S-8 | #14 | Accessibility conformance in both themes | 4 |
| S-9 | #15 | Automated test suite for date, theme, and responsive layout | 4 |
| S-10 | #16 | Performance and security budget checks | 5 |
| S-11 | #17 | Continuous integration runs the tests on pull requests | 5 |
| S-12 | #18 | Deployment configuration for Azure Static Web Apps Free plan | 5 |

Epics covered: E-1 (#1), E-2 (#2), E-3 (#3), E-4 (#4), E-5 (#5), E-6 (#6).

## Dependencies

Implementation order follows the backlog dependency graph; no story depends on anything outside this iteration.

- S-1 has no dependencies and comes first.
- S-2, S-3, S-12 depend on S-1.
- S-4 depends on S-2. S-5 and S-6 depend on S-4.
- S-7 depends on S-2 and S-4. S-8 depends on S-2, S-4, S-7.
- S-9 depends on S-3, S-5, S-6, S-7. S-10 depends on S-2 and S-4. S-11 depends on S-9.

Suggested sequence: S-1 → S-2 → S-3, S-4 → S-5, S-6 → S-7 → S-8 → S-9 → S-10, S-11 → S-12.

External dependencies: Node.js LTS, Playwright (Chromium only) and @axe-core/playwright as dev dependencies, Azure CLI for Bicep build and lint in CI. No runtime dependencies, no paid services.

## Expected deliverables

- `src/index.html`, `src/styles.css`, `src/app.js`, `src/favicon.svg`: the page (ADR-0001, ADR-0005), with the inline pre-paint theme script hashed in the CSP (ADR-0004).
- `src/staticwebapp.config.json`: security headers including CSP and HSTS (ADR-0004).
- `tests/` with Playwright specs covering date display, theme toggle, OS preference, persistence, 320 px layout, accessibility (axe), performance budget, zero third-party requests, CSP hash consistency; `tests/static-server.mjs` (ADR-0003).
- `package.json`, lockfile, `.nvmrc`, `playwright.config.*`, `.gitignore`, `README.md`.
- `.github/workflows/test.yml` (required check), `.github/workflows/provision.yml` and `.github/workflows/deploy-swa.yml` (workflow_dispatch only, `production` Environment) (ADR-0006).
- `infra/main.bicep` and `infra/main.bicepparam` with SKU Free and staging environments disabled (ADR-0007). Authored only; never applied in this iteration.
- `.github/dependabot.yml` (npm and github-actions, weekly).
- Branch protection on `main` requiring a pull request and the `test` check (configured by the human or with explicit authorization).

## Risks

- OIDC-only content deployment is unconfirmed; the deploy workflow fetches and masks the deployment token at run time. Verified at Gate 3, not in this iteration.
- CSP hash drift could break the pre-paint script; mitigated by an automated hash check in the test suite.
- Playwright fixed-clock behaviour across reload is not documented; fallback is `addInitScript`.
- Chromium-only testing may miss engine differences; standards-only features and a manual smoke test at Gate 3.
- The repository is public: any committed secret is permanent. No secrets by design; GitHub secret scanning and push protection are enabled for public repos.
- The `.ai-sdlc/config.json` file exposes Azure subscription and tenant identifiers (accepted by the human on 2026-10-14; identifiers only).

## Acceptance conditions

The iteration is done when all of the following hold:

1. Every story's acceptance criteria in `docs/backlog.md` pass automated tests, and the full suite is green on `main`.
2. Each story has been independently verified by the QA agent against its acceptance criteria and recommended for acceptance; QA findings are resolved or recorded as known defects.
3. The `test` workflow runs on every pull request and is configured as a required status check on `main`.
4. `infra/main.bicep` builds and lints cleanly in CI; no Azure resource has been created.
5. Documentation (`README.md`, architecture, ADR statuses) reflects what was built.
6. Gate 2 (integration) approval by the human has been given for the merged stories, naming the pull requests.

## Out of scope

- Provisioning Azure resources, deploying the site, the one-time Entra and resource-group bootstrap, choosing the Azure region or resource names (Milestone 0.4, Gate 3).
- Custom domains, analytics, authentication, APIs, databases, paid services, multi-browser CI.
- Any change to the approved requirements or architecture (requires re-approval).
