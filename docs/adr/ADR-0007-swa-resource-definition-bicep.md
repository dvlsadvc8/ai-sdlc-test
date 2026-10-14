# ADR-0007: SWA resource defined in Bicep and provisioned idempotently by an approval-gated pipeline in Milestone 0.4

- Status: proposed
- Date: 2026-10-14 (revised the same day after the human's answer to open question 2)
- Requirements: NFR-8, NFR-9, NFR-5, A13, S-12

## Context

S-12 asks that the repository contain configuration targeting Azure Static Web Apps with the Free plan (SKU Free). The GitHub Actions deploy workflow cannot express the SKU; it is a property of the Azure resource. The human directed (open question 2) that infrastructure be defined as code in Bicep, that nothing be provisioned in Milestone 0.2, and that in Milestone 0.4 provisioning be automated and approval-gated: the DevOps agent validates the Bicep, presents what-if results and estimated cost, obtains explicit Gate 3 approval, and the authorized deployment then runs with appropriately scoped credentials. Manual execution of Azure commands per release was explicitly rejected. The configuration must be re-appliable idempotently, unauthorized changes must be prevented, SWA Free is to be used, and no unnecessary paid services may be introduced.

Verified facts (learn.microsoft.com): `Microsoft.Web/staticSites` latest stable API `2025-03-01`; `sku: { name: 'Free', tier: 'Free' }`; `properties.stagingEnvironmentPolicy` accepts `Enabled`/`Disabled`; `allowConfigFileUpdates` controls whether `staticwebapp.config.json` is honoured; omitting `repositoryUrl`/`branch`/`repositoryToken` creates an unlinked app (no Azure-generated workflow). What-if makes no changes and reports `Create`, `Modify`, `NoChange`, etc.; it needs the same permissions as a deployment and requires the target resource group to exist. A `CanNotDelete` lock on a resource group prevents automatic deletion of deployment history (800-entry limit).

## Decision

### Template

- `infra/main.bicep` (resource-group scope) declares exactly one resource:

  ```bicep
  resource swa 'Microsoft.Web/staticSites@2025-03-01' = {
    name: name                       // deterministic, from main.bicepparam, e.g. stapp-ai-sdlc-demo
    location: location               // Free-plan region decided at Gate 3, e.g. eastus2
    tags: tags
    sku: { name: 'Free', tier: 'Free' }
    properties: {
      stagingEnvironmentPolicy: 'Disabled'   // no PR preview environments, ever
      allowConfigFileUpdates: true           // honour src/staticwebapp.config.json
      // no repositoryUrl / branch / repositoryToken: unlinked app, no Azure-injected workflow
    }
  }
  output defaultHostname string = swa.properties.defaultHostname
  ```

  No secure values, no `listSecrets()` in outputs, no `utcNow()` or other non-deterministic functions (they would make what-if noisy and break the `NoChange` idempotency check).
- `infra/main.bicepparam` supplies `name`, `location`, and `tags`. Values are fixed so that repeated applies target the same resource.

### Provisioning model

- Milestone 0.2: the template design above is recorded; nothing is provisioned, validated against Azure, or logged into.
- Milestone 0.3: the files are authored and `az bicep build` + `az bicep lint` run in `test.yml` on every PR (no Azure login).
- Milestone 0.4 (Gate 3), in order:
  1. The human performs the one-time bootstrap (ADR-0006, section "One-time bootstrap"): resource group, Entra app registration with federated credential, `Contributor` on the resource group, `CanNotDelete` lock, GitHub `production` environment.
  2. The DevOps agent runs `az bicep build`, `az bicep lint`, and `az deployment group what-if` against the empty resource group using the human's existing read-only Azure CLI session (what-if changes nothing), and presents to the human: the what-if output (expected: `1 to create`, `Microsoft.Web/staticSites/<name>`, nothing else), the cost estimate (SWA Free 0 USD; resource group, lock, role assignment, app registration 0 USD), the subject claim and role assignment in use.
  3. On the human's explicit Gate 3 approval, the human dispatches `provision.yml` on `main` and approves the `production` environment run. The job logs in with OIDC, re-runs what-if into the job summary, applies with `az deployment group create --name swa-infra --mode Incremental`, and re-runs what-if asserting `NoChange`.
  4. Content deployment follows via `deploy-swa.yml` (ADR-0006).

### Idempotency

- ARM incremental mode; deterministic resource names from the parameters file; fixed deployment name `swa-infra` so the deployment history holds one entry (relevant because the `CanNotDelete` lock disables automatic history cleanup); no non-deterministic template functions. Re-running `provision.yml` with an unchanged template yields `NoChange` and modifies nothing; the job fails if the post-apply what-if reports anything else.

### Preventing unauthorized changes

- Covered by ADR-0006: environment-bound federated credential, required reviewer, `main`-only deployment branch, protected `main` with required `test` check (so Bicep edits are reviewed and linted before they can be applied), RBAC confined to the resource group without `Microsoft.Authorization/*`, and the human-applied `CanNotDelete` lock.

## Alternatives considered

- **Human applies the Bicep manually at each release** (previous proposal): rejected by the human.
- **Portal creation documented in a runbook**: not infrastructure as code; SKU and settings would drift and S-12's criterion would rest on prose. Rejected.
- **`az staticwebapp create` imperative script**: not declarative, no what-if, weaker idempotency guarantees. Rejected.
- **Terraform**: adds state storage and a provider for one resource. Rejected.
- **Azure Deployment Stacks**: offers deny settings and managed cleanup, but adds a newer construct for a single resource; can be adopted later if the footprint grows. Not now.
- **Subscription-scope Bicep that also creates the resource group, role assignment, and lock**: would require the pipeline to hold subscription-level and `Microsoft.Authorization/*` rights, contradicting least privilege. Those steps stay in the human bootstrap.
- **Linking the SWA to the GitHub repository in Bicep (`repositoryUrl`, `repositoryToken`)**: makes Azure generate a workflow and store a GitHub token; contradicts ADR-0006. Rejected.
- **Pipeline applies the lock**: needs `Microsoft.Authorization/locks/*`. Rejected; human applies it at bootstrap.

## Consequences

- Positive: the SKU Free, staging-environment policy, and unlinked configuration are versioned, linted on every PR, previewed with what-if, applied automatically only after explicit approval, and re-appliable with a provable `NoChange`; no human runs Azure commands per release; zero cost.
- Negative: one more file type (Bicep) and a third workflow; the pipeline identity holds `Contributor` on one resource group (R14); the bootstrap is a one-time manual step requiring the human's elevated rights; the first what-if can only be run after the resource group exists.
- Follow-up: region, resource group name, and SWA name are fixed in `infra/main.bicepparam` at Gate 3 and recorded in `docs/architecture.md` section 5; after the first apply, verify in the portal that the SKU is Free, staging environments are disabled, and no GitHub workflow was added to the repository.
