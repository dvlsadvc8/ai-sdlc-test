# ADR-0003: Playwright Test with Chromium and axe-core for automated verification

- Status: proposed
- Date: 2026-10-14
- Requirements: NFR-7, NFR-1, NFR-2, NFR-3, NFR-4, NFR-9, S-8, S-9, S-10, S-11

## Context

NFR-7 demands headless, non-interactive tests that verify (a) the date equals the device-local date, (b) the toggle switches themes without reload, (c) the manual choice survives reload, (d) the OS preference is honoured on first visit, and (e) a 320 px layout check. S-8 adds an automated WCAG 2.2 AA audit in both themes; S-10 adds transfer-size, request-origin, storage-key, and secret checks. These need a real browser with control over the clock, time zone, locale, colour-scheme emulation, viewport, storage, and network observation. The tooling must be a dev-only dependency (S-9) and run in GitHub Actions on `ubuntu-latest` in a few minutes (S-11).

Playwright capabilities were verified against playwright.dev: `colorScheme` accepts `light`, `dark`, `no-preference`; `locale`, `timezoneId`, and `viewport` can be set per test with `test.use`; `page.clock.setFixedTime()` overrides `Date.now()` and `new Date()`; `page.reload()` keeps `localStorage`; `page.on('request')` observes all requests; `webServer` starts a local server before tests.

## Decision

- Test runner: `@playwright/test`, Chromium project only.
- Accessibility: `@axe-core/playwright` with tags `wcag2a`, `wcag2aa`, `wcag22aa`, run in both themes.
- Local server: `tests/static-server.mjs`, a dependency-free Node `http` server that serves `src/` and applies the `globalHeaders` from `src/staticwebapp.config.json`, started by Playwright's `webServer`. This makes CSP violations and header regressions visible in tests with no extra dependency.
- Determinism: tests pin `locale: 'en-US'`, use `timezoneId` and `page.clock.setFixedTime()` for the date tests, and use fresh contexts with explicit `colorScheme` for theme tests.
- Node.js current LTS (at implementation time) pinned in `.nvmrc`; `npm ci` against a committed `package-lock.json`; both packages are `devDependencies`; Dependabot keeps them current on a weekly schedule (coordinator-accepted defaults for architecture open questions 7 and 8, not human-stated preferences).
- The suite is the single `npm test` command used locally and by `.github/workflows/test.yml`.

## Alternatives considered

- **Cypress**: larger install, weaker colour-scheme and time-zone emulation, component of a commercial dashboard; no advantage for this scope. Rejected.
- **Puppeteer**: Chromium automation without a test runner, assertions, fixtures, or reporter; axe integration is manual. Rejected.
- **jsdom + Vitest/Jest**: no layout engine, so viewport overflow, bounding boxes, focus visibility, and contrast cannot be measured; S-7 and S-8 would be unverifiable. Rejected.
- **Playwright with Chromium + Firefox + WebKit**: roughly triples CI time and downloads; the page uses only standards-track features. Rejected for now; enabling additional projects is a one-line config change if a defect motivates it.
- **Lighthouse CI for performance**: heavier, flaky on shared runners, and the budget is simple enough to assert directly from captured responses. Rejected.
- **Serving via `npx serve`/`http-server`**: an extra dependency that would not apply SWA headers. Rejected in favour of the ~40-line built-in server.

## Consequences

- Positive: one toolchain covers functional, responsive, accessibility, performance-budget, and security-header checks; CI job of 3-5 minutes; unmetered on a public repository (NFR-9).
- Negative: Chromium-only coverage (R6); Playwright downloads a browser on every CI run (cacheable later if duration matters); axe cannot measure focus-indicator contrast, so a small WCAG luminance helper in the suite covers that check (R8).
- Follow-up: verify during S-3 implementation that the fixed clock persists across `page.reload()`; fall back to `addInitScript` if not (R7). Record the deliberate-regression ("mutation") procedure in the README for S-9's third acceptance criterion.
