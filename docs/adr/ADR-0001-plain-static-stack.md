# ADR-0001: Plain HTML, CSS, and JavaScript with no framework and no build step

- Status: proposed
- Date: 2026-10-14
- Requirements: FR-1..FR-11, NFR-4, NFR-6, NFR-10, A8, A10

## Context

The product is a single static page with three fixed strings, one computed string (the local date), and one bit of persisted state (the theme). NFR-6 requires plain HTML, CSS, and JavaScript unless the Architect records a concrete reason for a framework; NFR-4 caps the initial transfer at 100 KB compressed with zero third-party runtime scripts. The human has stated a preference for plain HTML/CSS/JS. Supported browsers (A8) are the current and previous major versions of Chrome, Edge, Firefox, and Safari, all of which implement every platform feature the page needs: CSS custom properties, `prefers-color-scheme`, `:focus-visible`, `matchMedia`, `localStorage`, and `Intl.DateTimeFormat` with `dateStyle`.

## Decision

Author the page as three hand-written files served verbatim from `src/`:

- `index.html`: semantic document, one inline pre-paint theme script (about 10 lines), `<link>` to the stylesheet, deferred `<script src="app.js">`.
- `styles.css`: design tokens in CSS custom properties, light and dark themes, fluid layout from 320 px, focus styles.
- `app.js`: classic script (no modules needed) that renders the date and wires the toggle; ES2017-level syntax; JSDoc comments where behaviour is not self-evident.

No framework, no CSS framework, no TypeScript, no bundler, no minifier, no transpiler. The deployable artifact is the `src/` folder itself (`skip_app_build: true` in the deploy action). Node.js and npm are used only for the test toolchain (ADR-0003) and nothing from `node_modules` is ever deployed.

## Alternatives considered

- **UI framework (React, Vue, Svelte, Lit)**: adds a build step, a dependency tree to keep patched in a public repo, and more bytes than the entire page. No requirement needs components, routing, or reactive state beyond one attribute. Rejected.
- **Static-site generator (Eleventy, Astro)**: useful for many pages or shared layouts; there is one page. Rejected.
- **Tailwind or Bootstrap**: either a build step or 20-200 KB of CSS for a page that needs roughly 60 lines. Rejected.
- **TypeScript**: type safety for ~40 lines does not justify a compile step; JSDoc gives editor hints without one. Rejected.
- **Minification/bundling step**: the three files total roughly 10 KB uncompressed and SWA compresses in transit; a build step would add tooling and a CSP-hash recomputation path for no measurable gain. Rejected.
- **Web Components**: no reuse case. Rejected.

## Consequences

- Positive: zero runtime dependencies; trivially within the performance budget; no supply-chain exposure on the served page; any editor can change it; the pre-paint theme script can be hashed for a strict CSP (ADR-0004); `view-source` shows exactly what runs.
- Negative: no type checking; features must be verified against A8 browser support by hand (all currently are); if the page ever grows into a multi-page site this decision should be revisited.
- Follow-up: the README documents the two commands (`npm run serve`, `npm test`); S-1 verifies that no framework dependency exists.
