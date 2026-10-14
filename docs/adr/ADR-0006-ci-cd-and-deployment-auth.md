# ADR-0006: CI/CD workflows, required check, OIDC federated identity scoped to one resource group, approval-gated provision and deploy

- Status: proposed
- Date: 2026-10-14 (revised the same day after the human's answers to open questions 1 and 2)
- Requirements: NFR-5, NFR-7, NFR-8, NFR-9, Q1 (revised), Q2, A13, S-11, S-12

## Context

The human decided (Q2) that tests run in GitHub Actions on every pull request and default-branch push as a required status check, and (Q1 revised) that the repository is PUBLIC on GitHub Free so that branch protection, rulesets, Environments with required reviewers, and required checks are available at zero cost and Actions minutes are unmetered. The default branch is `main` (open question 1). A13 requires the deployment configuration to be authored during implementation but not executed until Gate 3. NFR-5 forbids any credential in files, logs, or history of the world-readable repository.

The human further directed (open question 2) that the framework must automate infrastructure provisioning and application deployment after explicit human approval, with no manual Azure commands per release, using appropriately scoped credentials, idempotent re-apply, and protection against unauthorized infrastructure changes. This ADR therefore covers the identity the pipeline uses against Azure as well as the GitHub side.

Verified facts (learn.microsoft.com, docs.github.com):

- The Azure-generated SWA workflow triggers on `push` and `pull_request`, deploys every PR to a preview environment, and passes the deployment token to PR jobs.
- `Azure/static-web-apps-deploy@v1` requires `azure_static_web_apps_api_token`; `github_id_token` is optional and OIDC-only content deployment is not confirmed.
- `az staticwebapp secrets list` returns the deployment token (`properties.apiKey`); it needs `Microsoft.Web/staticSites/listSecrets/action`, which `Contributor` includes.
- `azure/login@v2` supports OIDC with a Microsoft Entra app registration or a user-assigned managed identity carrying a federated identity credential; issuer `https://token.actions.githubusercontent.com`, audience `api://AzureADTokenExchange`.
- Repositories created after 15 July 2026 (this one was created in October 2026) receive the immutable OIDC subject format that includes owner and repository IDs, for example `repo:OWNER@OWNER-ID/REPO@REPO-ID:environment:production`; the IDs cannot be removed from the subject.
- Built-in role `Website Contributor` grants `Microsoft.Web/sites/*`, `serverFarms/read`, `serverFarms/join/action`, `certificates/*` and `Microsoft.Resources/deployments/*`, but not `Microsoft.Web/staticSites/*`; it cannot manage a Static Web App. `Contributor` can, and excludes `Microsoft.Authorization/*` (no RBAC or lock changes).
- Creating management locks requires `Microsoft.Authorization/locks/*` (Owner or User Access Administrator); `Contributor` cannot create or remove locks.
- What-if makes no changes but requires the same permissions as a deployment.

## Decision

### Workflows

1. **`.github/workflows/test.yml`** (S-11): triggers `pull_request` and `push` to `main`; `permissions: contents: read`; `runs-on: ubuntu-latest`; checkout, `actions/setup-node` from `.nvmrc`, `npm ci`, `npx playwright install --with-deps chromium`, `npx playwright test`, `az bicep build --file infra/main.bicep` and `az bicep lint` (no Azure login), upload `playwright-report` only `if: failure()`. Job name `test` is the required status check. No environment, no secrets, no `id-token`.
2. **`.github/workflows/provision.yml`** (S-12): trigger `workflow_dispatch` only; `if: github.ref == 'refs/heads/main'`; `environment: production`; `permissions: contents: read, id-token: write`; steps: checkout, `azure/login` (OIDC with `client-id`, `tenant-id`, `subscription-id` from environment secrets), `az deployment group what-if --mode Incremental` written to the job summary, `az deployment group create --name swa-infra --mode Incremental --template-file infra/main.bicep --parameters infra/main.bicepparam`, then a second what-if asserting `NoChange`. Details of the template are in ADR-0007.
3. **`.github/workflows/deploy-swa.yml`** (S-12): trigger `workflow_dispatch` only; `main` only; `environment: production`; `permissions: contents: read, id-token: write`; steps: checkout, run the test suite, `azure/login` (OIDC), fetch the deployment token with `az staticwebapp secrets list --query properties.apiKey -o tsv` and immediately `::add-mask::` it, then `Azure/static-web-apps-deploy` with `action: upload`, `app_location: src`, `output_location: ""`, `skip_app_build: true`, `skip_api_build: true`, `azure_static_web_apps_api_token` from the fetched value. No `pull_request` trigger and no `close_pull_request_job`, so SWA preview environments are never used (and `stagingEnvironmentPolicy` is `Disabled` in Bicep). If OIDC-only content deployment is confirmed at Gate 3, the token step is removed in a follow-up ADR.

### Identity and authorization

4. **Identity**: one Microsoft Entra app registration with a service principal and no client secret, carrying one federated identity credential bound to this repository's `production` environment (subject in the immutable owner-ID/repo-ID format; exact string confirmed by the DevOps agent from GitHub documentation and a read-only `gh api` lookup before the human enters it). Rationale for an app registration over a user-assigned managed identity: a UAMI placed in the application resource group could have federated credentials added by the pipeline's own Contributor role (`Microsoft.ManagedIdentity/userAssignedIdentities/federatedIdentityCredentials/write`), an escalation path; an app registration has no ARM footprint and is immune to RG-scoped roles. A UAMI in a separate identity-only resource group is an acceptable alternative if the human prefers to avoid Entra app registrations.
5. **Role assignment**: built-in `Contributor` on the single application resource group only. Never at subscription scope, never Owner or User Access Administrator. Optional later hardening at zero cost: a custom role limited to `Microsoft.Web/staticSites/*`, `Microsoft.Resources/deployments/*`, `Microsoft.Resources/subscriptions/resourceGroups/read`, `Microsoft.Authorization/*/read`.
6. **GitHub Environment `production`**: required reviewer (the human); deployment branches limited to `main`; environment secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` (identifiers, stored as secrets per Microsoft guidance for public repositories); environment variables `AZURE_RESOURCE_GROUP`, `AZURE_LOCATION`, `SWA_NAME`. No deployment token, client secret, or any other credential is stored in GitHub.
7. **Deployment token**: never stored. It exists only in Azure and, transiently and masked, inside an approved `production` job. It can be reset at any time with `az staticwebapp secrets reset-api-key` by the human or by the pipeline identity.

### Preventing unauthorized infrastructure changes

8. Five independent layers: (a) the federated credential trusts only tokens whose subject names the `production` environment, so no PR job, fork, or other branch can obtain an Azure token; (b) the environment requires human approval and admits only `main`; (c) `main` is protected by a ruleset requiring a pull request and the `test` check, so workflow and Bicep changes are reviewed before they can run with privileges; (d) Azure RBAC confines the identity to one resource group and excludes `Microsoft.Authorization/*`; (e) a `CanNotDelete` lock on the resource group, applied by the human, blocks deletion by any principal including the pipeline.

### One-time bootstrap (human, outside the conversation, Milestone 0.4)

9. Because they require Owner/User Access Administrator rights on the subscription and Entra application-creation rights, which the pipeline must never hold, the human performs these once: create the resource group; create the app registration and federated credential; assign `Contributor` on the resource group; apply the `CanNotDelete` lock; create the GitHub `production` environment with reviewer, branch restriction, secrets, and variables; confirm the `main` ruleset. The README carries this as a runbook. The AI agents only read the outcome (read-only `az`/`gh`).

### Hygiene and milestone boundary

10. Every third-party action pinned to a full commit SHA with a version comment; `permissions:` block on every job; `.github/dependabot.yml` (weekly) for `npm` and `github-actions`; committed lockfile; all npm packages dev-only.
11. All three workflow files and the Bicep are authored in Milestone 0.3. `test.yml` runs as soon as it is on a PR. `provision.yml` and `deploy-swa.yml` cannot run before Gate 3: they have no automatic trigger, the `production` environment and its secrets do not exist until the human creates them, and no federated credential exists for Azure to honour.

## Alternatives considered

- **Portal-generated SWA workflow**: deploys every PR branch to a preview environment using the deployment token available to PR jobs. Rejected.
- **Human runs `az deployment group create` manually**: rejected by the human; the framework must automate provisioning after approval.
- **Deployment token stored as a GitHub environment secret** (previous proposal): works, but the human must copy and rotate a long-lived credential by hand. Superseded by run-time retrieval with the OIDC identity; no stored credential remains.
- **Client-secret service principal**: long-lived secret in GitHub. Rejected in favour of federated credentials.
- **OIDC-only content deployment with no token**: not confirmed to work with the action; the run-time fetch removes the storage problem, and the step is dropped if OIDC-only is confirmed at Gate 3.
- **User-assigned managed identity in the application resource group**: escalation path via the pipeline's own Contributor role. Rejected; acceptable only in a separate identity resource group.
- **`Website Contributor` role**: verified not to cover `Microsoft.Web/staticSites`. Rejected.
- **Subscription-scoped Contributor or Owner**: violates least privilege. Rejected.
- **Pipeline applies the resource lock**: requires `Microsoft.Authorization/locks/*`, which must not be granted to the pipeline. The human applies the lock at bootstrap.
- **Automatic deploy on push to `main`**: simpler, but A13 and the human's gate requirement call for an explicit approval per release; manual dispatch with an environment reviewer provides it at no cost.
- **Single combined workflow**: mixes the unprivileged PR path with privileged paths; harder to review. Rejected.

## Consequences

- Positive: no long-lived credential anywhere in GitHub; PR jobs cannot reach Azure at all; every privileged run requires a human click and a reviewer approval on protected code; Azure blast radius is one resource group; infrastructure is reproducible and idempotent; everything is zero cost.
- Negative: one-time bootstrap requires the human's elevated rights and about six manual steps; `Contributor` on the RG is broader than the single resource type needs (R14); the deployment token still exists in Azure and is briefly in job memory (R15); the new immutable subject format must be entered exactly (R12).
- Follow-up: at Gate 3 the DevOps agent prepares the approval package (Bicep, lint, local what-if, 0 USD cost statement, subject and role to be used); after the human's bootstrap and approval, `provision.yml` then `deploy-swa.yml` are dispatched and approved; verify HTTPS, headers, and the four primary elements on the live site; record region, names, and the OIDC-only finding in `docs/architecture.md`.
