# ADR-0002: Host on Azure Static Web Apps Free plan

- Status: proposed
- Date: 2026-10-14
- Requirements: NFR-8, NFR-9, NFR-5, A12, A13

## Context

NFR-8 requires deployability to Azure Static Web Apps on the Free plan over HTTPS; NFR-9 requires 0 USD/month; A12 fixes the default `*.azurestaticapps.net` hostname with no custom domain. The DevOps consultation (verified against learn.microsoft.com) confirms the Free plan provides: automatic HTTPS with managed, auto-renewing certificates; 100 GB bandwidth per subscription per month; 250 MB per app; up to 3 staging environments; 2 custom domains; `staticwebapp.config.json` support including `globalHeaders`; no SLA. Only `networking` and `forwardingGateway` configuration are Standard-only, and neither is needed.

## Decision

Host the page on one Azure Static Web App, SKU Free, in a dedicated resource group, serving the `src/` folder as `app_location` with no build and no API. Use the default hostname. Configure response headers through `src/staticwebapp.config.json` (ADR-0004). Create the resource without a linked GitHub repository so Azure injects neither a workflow nor a token into the public repository; deployment is performed by the repository's own manually dispatched workflow (ADR-0006). The Azure region is chosen at Gate 3 from the Free-plan list; DevOps recommends East US 2.

No Azure resource is created in Milestone 0.2 or 0.3 (A13); provisioning happens only after Gate 3 approval.

## Alternatives considered

- **GitHub Pages**: free and simple, but not Azure; violates NFR-8. Rejected.
- **Azure Storage static website (+ Azure CDN/Front Door for headers and TLS)**: small non-zero cost, no custom response headers without a CDN, weaker managed-certificate story for custom domains. Rejected.
- **Azure Static Web Apps Standard**: adds SLA, custom auth, networking rules at roughly 9 USD/month; nothing in the requirements needs them. Rejected (needs human approval if ever reconsidered, NFR-9).
- **Azure App Service / Container Apps**: server runtime for a page with no server logic. Rejected.

## Consequences

- Positive: 0 USD; HTTPS and global distribution out of the box; headers configurable from a file in the repository; SWA Free is sufficient for several orders of magnitude more traffic than a demo will see.
- Negative: no SLA; limited region list; the deployment credential is a long-lived token unless OIDC support in the deploy action is confirmed (ADR-0006, R1); preview environments exist but are deliberately unused.
- Follow-up: at Gate 3 verify with `curl -I` that the configured headers are served; record the region and resource group in `docs/architecture.md` section 5.
