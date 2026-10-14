# Backlog: AI-SDLC Demo Website

Derived from docs/requirements.md (analyzed 2026-10-14). Requirement ids FR-n and NFR-n refer to that document. Milestone 0.2 boundary: this backlog is a plan only; no story is implemented, merged, provisioned, or deployed until the respective later-milestone approvals are granted.

## E-1: Static page foundation
Requirements: FR-1, FR-2, FR-3, NFR-1, NFR-6, NFR-10
Create the application repository structure and the single semantic HTML page with its heading, tab title, and welcome message, using plain HTML, CSS, and JavaScript.

### S-1: Repository scaffold and project structure
Priority: 1
Depends on: none
Requirements: NFR-6, NFR-10
As a developer, I want a straightforward, documented project structure in the application repository, so that the page, styles, scripts, and tests have clear homes separate from the AI-SDLC framework repository.

Acceptance criteria:
- Given the application repository, when it is inspected, then it contains a `README.md` that states the project purpose and gives the commands to serve the page locally and to run the automated tests.
- Given the repository layout, when the Architect's structure is compared to it, then source, styles, scripts, tests, and `docs/` architecture artifacts each have a dedicated location and no framework (library) dependency is present unless `docs/` records a justification.
- Given the repository, when it is scanned, then it contains no files belonging to the AI-SDLC framework repository (C:/repos/ai-sdlc) and no secrets, tokens, or credentials.

### S-2: Heading, tab title, and welcome message
Priority: 1
Depends on: S-1
Requirements: FR-1, FR-2, FR-3, NFR-1
As a visitor, I want to see the page title and a welcome message immediately when the page loads, so that I know I have reached the AI-SDLC demo website.

Acceptance criteria:
- Given the page is loaded, when the document is inspected, then there is exactly one `<h1>` element and its visible text is "AI-SDLC Demo".
- Given the page is loaded, when the browser tab is inspected, then `document.title` is "AI-SDLC Demo".
- Given the page is loaded at any viewport width of 320 px or wider, when no user interaction has occurred, then the text "Welcome to the AI-SDLC demo website." is visible in the initial viewport.
- Given the document source, when it is validated, then it declares `lang="en"`, uses landmark elements (at least `<header>` or `<main>`), has no heading-level skips, and renders all text as real text rather than images.

## E-2: Local date display
Requirements: FR-4, FR-5
Show the visitor's current calendar date, computed from the device clock and time zone at page load, in a readable long format.

### S-3: Display the visitor's local date on page load
Priority: 2
Depends on: S-1
Requirements: FR-4, FR-5
As a visitor, I want to see today's date in my own time zone in a readable format, so that the page shows live, device-local information rather than a static placeholder.

Acceptance criteria:
- Given the page is loaded with the browser clock at a known date and time zone, when the date element is read, then it shows that device-local calendar date (for example "October 14, 2026" in an en-US locale) with full month name, day, and four-digit year.
- Given the HTML source file, when it is inspected, then it contains no hardcoded calendar date; the value is produced by client-side script at load time.
- Given the browser clock is advanced to a different day and the page is reloaded, when the date element is read, then it shows the new date.
- Given the browser time zone is set so that UTC and local dates differ (for example 23:30 local on 14 October in UTC-10), when the page is loaded, then the displayed date is the local date (14 October), not the UTC date.

## E-3: Light and dark theme
Requirements: FR-6, FR-7, FR-8, FR-9, NFR-2, NFR-4
Provide a clearly labeled, keyboard-operable toggle between light and dark themes that honours the operating-system preference on first visit and remembers a manual choice across reloads, without a flash of the wrong theme.

### S-4: Theme toggle switches themes without reload
Priority: 2
Depends on: S-2
Requirements: FR-6, FR-9, NFR-2
As a visitor, I want a clearly labeled control that switches the page between light and dark themes instantly, so that I can read the page comfortably in my environment.

Acceptance criteria:
- Given the page is in the light theme, when the toggle is activated with a mouse click, then the page background, text, heading, date, and toggle switch to the dark theme and the URL and page load count are unchanged (no navigation or reload).
- Given the page is in the dark theme, when the toggle is activated again, then the page returns to the light theme.
- Given the toggle, when its accessible name and state are read by assistive technology, then the name clearly describes the theme action (for example "Switch to dark theme") or the control exposes a role with a pressed/checked state that reflects the current theme.
- Given keyboard-only use, when the visitor presses Tab, then the toggle receives focus with a visible focus indicator, and pressing Enter or Space switches the theme.
- Given either theme is active, when the readability of background, text, and the toggle is checked, then all remain legible (contrast thresholds verified in S-8).

### S-5: Honour the operating-system theme preference on first visit
Priority: 3
Depends on: S-4
Requirements: FR-7, NFR-4
As a first-time visitor, I want the page to open in the theme matching my operating-system setting, so that the page feels consistent with my device without any configuration.

Acceptance criteria:
- Given no stored theme preference and the browser reports `prefers-color-scheme: dark`, when the page is loaded, then the dark theme is applied.
- Given no stored theme preference and the browser reports `prefers-color-scheme: light`, when the page is loaded, then the light theme is applied.
- Given no stored theme preference and the browser reports no colour-scheme preference (or does not support the media query), when the page is loaded, then the light theme is applied.
- Given the OS preference is dark and no manual choice has been made, when the page is loaded, then there is no visible flash of the light theme before the dark theme is applied (theme applied before first paint).

### S-6: Remember the manually selected theme across reloads
Priority: 3
Depends on: S-4
Requirements: FR-8, NFR-4
As a returning visitor, I want my chosen theme to be remembered in this browser, so that I do not need to re-select it on every visit.

Acceptance criteria:
- Given the OS preference is light, when the visitor toggles to dark and reloads the page in the same browser, then the dark theme is applied on load.
- Given the OS preference is dark, when the visitor toggles to light and reloads the page in the same browser, then the light theme is applied on load (manual choice overrides OS preference).
- Given a stored manual preference, when the page is loaded, then there is no visible flash of the other theme before the stored theme is applied.
- Given browser storage is cleared or unavailable, when the page is loaded, then the page falls back to the OS-preference behaviour of S-5 and the toggle still switches themes for the current page view without errors.
- Given the stored data is inspected, when its contents are read, then the only persisted value is the theme preference and no personal information or other data is stored.

## E-4: Responsive and accessible layout
Requirements: FR-10, FR-11, NFR-1, NFR-2, NFR-3, FR-9
Make the page readable and usable from 320 px viewports upward without horizontal scrolling, and verify WCAG 2.2 AA structure, keyboard operation, focus visibility, and contrast in both themes.

### S-7: Responsive layout from 320 px upward
Priority: 3
Depends on: S-2, S-4
Requirements: FR-10, FR-11
As a mobile visitor, I want the page to fit my narrow screen, so that I can read the heading, welcome message, and date and reach the theme toggle without scrolling sideways.

Acceptance criteria:
- Given a viewport width of 320 px, when the page is loaded, then `document.documentElement.scrollWidth` is less than or equal to the viewport width (no horizontal page scrolling).
- Given a viewport width of 320 px, when the page is loaded, then the heading, welcome message, date, and theme toggle are each fully within the viewport bounds, not clipped, and the toggle is clickable and focusable.
- Given viewport widths of 320 px, 768 px, and 1280 px, when the page is rendered at each, then all four primary elements are visible and text wraps rather than overflowing.
- Given a 320 px viewport, when the toggle is located, then it is directly visible (not behind a menu or disclosure) and its hit target is at least 24 by 24 CSS pixels.

### S-8: Accessibility conformance in both themes
Priority: 4
Depends on: S-2, S-4, S-7
Requirements: NFR-1, NFR-2, NFR-3, FR-9
As a visitor who relies on keyboard navigation or has low vision, I want the page to meet WCAG 2.2 AA for structure, focus, and contrast in both themes, so that I can read and operate it.

Acceptance criteria:
- Given either theme, when colour contrast is measured, then normal text is at least 4.5:1 against its background, large text at least 3:1, and the toggle boundary and focus indicator at least 3:1 against adjacent colours.
- Given either theme, when the toggle is focused via keyboard, then a visible focus indicator is present and meets the 3:1 contrast threshold.
- Given the page, when an automated accessibility audit (for example axe-core rules at WCAG 2.2 AA level) is run in both themes, then it reports zero violations.
- Given the document, when its structure is checked, then it has exactly one `<h1>`, a declared language, landmark elements, and every interactive control has an accessible name.

## E-5: Automated verification, performance, and security
Requirements: NFR-7, NFR-4, NFR-5
Provide headless automated checks for date, theme, and responsive behaviour, run them in continuous integration, and enforce the performance and security budget.

### S-9: Automated test suite for date, theme, and responsive layout
Priority: 4
Depends on: S-3, S-5, S-6, S-7
Requirements: NFR-7
As a developer, I want automated tests that verify the date display, theme behaviour, and responsive layout, so that regressions are caught without manual checking.

Acceptance criteria:
- Given the repository, when the documented test command is run headlessly without any manual step, then it completes with a pass/fail exit code and a readable report.
- Given the test suite, when its cases are listed, then they cover at minimum: date equals the device-local date in the expected format; toggle switches theme without navigation; manual choice survives reload; OS dark/light/no-preference is honoured on first visit; at a 320 px viewport there is no horizontal overflow and the four primary elements are visible.
- Given a deliberate regression (for example hardcoding the date or removing persistence), when the tests are run, then at least one test fails.
- Given the test tooling chosen by the Architect, when it is added, then its justification is recorded in `docs/` and it is a development-only dependency not shipped to the page.

### S-10: Performance and security budget checks
Priority: 5
Depends on: S-2, S-4
Requirements: NFR-4, NFR-5
As a maintainer, I want the page to stay lightweight and free of secrets or personal data handling, so that it loads quickly and poses no security or privacy risk.

Acceptance criteria:
- Given the built page and its assets, when total initial transfer size is measured (compressed), then it is at most 100 KB.
- Given the page, when its network requests on load are captured, then there are no third-party runtime scripts, fonts, or render-blocking external requests.
- Given the repository and client-side code, when scanned for secrets (API keys, tokens, connection strings), then none are found, and no form, input, or script collects personal information.
- Given the page, when client-side storage is inspected after use, then only the theme preference key exists.

### S-11: Continuous integration runs the tests on pull requests
Priority: 5
Depends on: S-9
Requirements: NFR-7, NFR-9
As a maintainer, I want the automated tests to run in GitHub Actions on every pull request and default-branch push, so that only verified changes are merged (confirmed by the human answer to clarification Q2: required status check).

Acceptance criteria:
- Given a pull request to the default branch, when it is opened or updated, then a GitHub Actions workflow runs the test suite headlessly and reports a status check.
- Given a failing test, when the workflow runs, then the status check fails and, because the check is configured as required, the pull request cannot be merged.
- Given the workflow, when its monthly usage is estimated, then it stays within the free GitHub Actions quota at zero cost.

## E-6: Azure Static Web Apps deployment readiness
Requirements: NFR-8, NFR-9, NFR-10
Prepare the configuration needed to deploy the page to Azure Static Web Apps on the Free plan over HTTPS, without provisioning or deploying until approved in a later milestone.

### S-12: Deployment configuration for Azure Static Web Apps Free plan
Priority: 5
Depends on: S-1
Requirements: NFR-8, NFR-9, NFR-10
As a maintainer, I want the repository to contain the Azure Static Web Apps deployment configuration, so that the approved deployment in a later milestone can proceed without additional code changes.

Acceptance criteria:
- Given the repository, when it is inspected, then it contains a deployment workflow or configuration targeting Azure Static Web Apps with the Free plan (SKU Free) and the static app root pointing at the page's output directory.
- Given the deployment configuration, when it is reviewed, then it contains no secrets or tokens in plain text and references only GitHub secrets or OIDC to be supplied at deployment approval time.
- Given Milestone 0.2 and the implementation milestone, when the configuration is merged, then no Azure resource has been created and no deployment has run; the workflow is gated on explicit human approval (for example manual trigger or a protected environment).
- Given an approved deployment in a later milestone, when the page is deployed, then it is reachable over HTTPS on the default azurestaticapps.net hostname and serves the heading, welcome message, date, and working toggle.
