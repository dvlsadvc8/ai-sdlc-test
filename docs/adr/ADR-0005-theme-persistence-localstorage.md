# ADR-0005: Theme persistence in one localStorage key, OS preference via CSS, two-state toggle

- Status: proposed
- Date: 2026-10-14
- Requirements: FR-6, FR-7, FR-8, FR-9, NFR-2, NFR-4, NFR-5, A5, A10, A11

## Context

FR-7 requires honouring `prefers-color-scheme` on first visit with light as the fallback; FR-8 requires a manual choice to be stored in the browser and to override the OS preference on later loads; A5 fixes `localStorage` with a single key and no cookies, with graceful behaviour when storage is unavailable; A11 fixes exactly two toggle states (no "system" option) and says that once a manual choice exists, later OS changes are ignored; NFR-4 forbids a flash of the wrong theme; NFR-5 permits only the theme preference as persisted data; A10 asks for graceful degradation without JavaScript.

## Decision

- **Storage**: `localStorage` key `theme`, value `"light"` or `"dark"`, nothing else. Written only when the visitor activates the toggle. Reads and writes are wrapped in `try/catch`; on failure the page behaves as if nothing were stored.
- **Resolution order**: stored manual choice > OS preference > light.
- **Mechanism**:
  - `styles.css` declares light tokens by default, dark tokens under `@media (prefers-color-scheme: dark)` scoped to `html:not([data-theme="light"])`, and unconditional dark tokens under `html[data-theme="dark"]`. The OS preference therefore works with no JavaScript at all and is live before any manual choice (permitted by A11).
  - The inline pre-paint script (ADR-0004) sets `data-theme` from storage before the stylesheet applies, so a stored choice is honoured at first paint.
  - `app.js` determines the effective theme (`data-theme` if present, else `matchMedia('(prefers-color-scheme: dark)').matches`), reflects it on the toggle, and on activation writes an explicit `light` or `dark` to both `data-theme` and storage. Writing an explicit value pins the choice so later OS changes are ignored (A11).
  - `color-scheme: light` / `color-scheme: dark` is set with the tokens so native UI (scrollbars, form controls) matches.
- **Toggle control**: a native `<button type="button">` with the stable accessible name "Dark theme" and `aria-pressed` reflecting whether dark is active (coordinator-accepted default for architecture open question 3, not a human-stated preference). Native semantics give keyboard reachability and Enter/Space activation (NFR-2). The button is `hidden` until `app.js` runs so no dead control appears when JavaScript is disabled (A10).
- **Live OS preference before a manual choice**: followed automatically through the CSS media query, with no additional code (coordinator-accepted default for open question 5; A11 permits either behaviour).

## Alternatives considered

- **Cookie**: sent with every request for no reason; excluded by A5. Rejected.
- **`sessionStorage`**: lost when the browser closes; FR-8 expects persistence across visits. Rejected.
- **Three-state toggle (light/dark/system)**: excluded by A11. Rejected.
- **Resolving the OS preference in JavaScript only (no CSS media query)**: the theme would then depend on script even on first visit, breaking A10 and making first paint depend on script execution timing. Rejected.
- **Toggle as a checkbox or `role="switch"`, or a label that flips between "Switch to dark theme" and "Switch to light theme"**: valid alternatives; a toggle button with a stable name and `aria-pressed` is the simplest pattern that satisfies S-4's accessible-name-or-state criterion with a native element and avoids renaming the control under a screen-reader user. Not chosen (open question 3 resolved).
- **Snapshotting the OS preference at load**: adds code to do less than the live media query; A11 does not require it. Rejected (open question 5 resolved).

## Consequences

- Positive: exactly one persisted value (NFR-5, S-6, S-10); no flash in either the stored or the OS case; OS preference works without JavaScript; `matchMedia` and `localStorage` are supported by every A8 browser; roughly 20 lines of JavaScript in total.
- Negative: the two CSS token blocks for dark are declared twice (media query and attribute); kept small by using custom properties. Visitors cannot return to "follow my OS" without clearing site data, which A11 accepts.
- Follow-up: tests in `tests/theme.spec.js` cover dark/light/no-preference first visits, persistence across reload in both directions, blocked storage, and that `Object.keys(localStorage)` is exactly `["theme"]`.
