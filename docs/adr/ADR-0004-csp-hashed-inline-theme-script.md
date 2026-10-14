# ADR-0004: Strict CSP with a SHA-256 hash for the inline pre-paint theme script

- Status: proposed
- Date: 2026-10-14
- Requirements: NFR-4, NFR-5, FR-7, FR-8, S-5, S-6, S-10

## Context

NFR-4 requires the theme to be applied before first paint with no flash of the wrong theme, and no render-blocking external requests. Applying a stored manual preference before paint requires a synchronous script that runs before the stylesheet is applied; the natural place is an inline `<script>` at the top of `<head>`. NFR-5 and the public repository call for a hardened security posture, and a Content-Security-Policy that forbids inline script is the main browser-side defence against injected script. On a static host there is no per-request server to mint CSP nonces, so the usual nonce approach is unavailable. Azure Static Web Apps allows arbitrary response headers through `globalHeaders` in `staticwebapp.config.json` on the Free plan.

## Decision

Serve a strict CSP from `src/staticwebapp.config.json`:

```
default-src 'none'; script-src 'self' 'sha256-<base64 hash of the inline theme script>'; style-src 'self'; img-src 'self'; base-uri 'none'; form-action 'none'; frame-ancestors 'none'
```

together with `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, a restrictive `Permissions-Policy`, and `Strict-Transport-Security: max-age=31536000` without `includeSubDomains` or `preload` (approved by the human on 2026-10-14, architecture open question 4; the hostname is Azure's shared `*.azurestaticapps.net`, so subdomain and preload directives are not meaningful).

The only inline code on the page is the pre-paint theme script, kept deliberately tiny and stable (read one `localStorage` key inside `try/catch`, set `data-theme` if the value is `light` or `dark`). Its SHA-256 is listed in `script-src`. All CSS lives in `styles.css` (no `style` attributes, no `<style>` blocks) so `style-src 'self'` holds. `app.js` is external and same-origin.

Guard against drift: a test in `tests/budget.spec.js` extracts the inline script from `index.html`, computes its SHA-256, and asserts it equals the hash in the CSP; the local test server applies the same headers so any mismatch also surfaces as a CSP console violation in every other test. The failure message prints the expected hash so a developer can update the config in one step.

## Alternatives considered

- **`script-src 'self' 'unsafe-inline'`**: trivially allows the inline script but neutralises CSP against XSS. Rejected.
- **External `theme-init.js` loaded synchronously before the stylesheet**: avoids the hash but adds a blocking request on the critical path, which is exactly what the pre-paint script exists to avoid. Rejected (acceptable fallback if hash maintenance proves burdensome).
- **Nonce-based CSP**: requires per-response generation; not possible on a static host. Rejected.
- **No CSP**: SWA serves no CSP by default. Weakens the security posture for no benefit. Rejected.
- **CSP via `<meta http-equiv>`**: works for most directives but cannot express `frame-ancestors`, and headers are the authoritative mechanism. Rejected; the header is used.
- **Storing the theme attribute via CSS only (no script)**: CSS cannot read `localStorage`, so the manual preference could not be applied before paint. Rejected.

## Consequences

- Positive: strict CSP with no `unsafe-*` keywords; inline script allowed by identity, not by class; clickjacking blocked by `frame-ancestors 'none'`; MIME sniffing and referrer leakage disabled; all enforced from a versioned file in the repository.
- Negative: every edit to the inline script (including whitespace) requires updating the hash; mitigated by the automated check (R2). Browsers that ignore CSP hashes simply run the script, so there is no functional regression.
- Follow-up: verify the headers, including HSTS, on the live site with `curl -I` at Gate 3 (R3); `budget.spec.js` asserts their presence on the locally served response.
