# SDLC Platform — Ecosystem Documentation

An enterprise-standard multi-tenant Agentic SDLC platform on Azure.
Tenants register their repos, the platform indexes their code, and AI agents handle the full software development lifecycle — from issue to merged PR.

---

## Architecture Overview

```
OUTSIDE (public internet)
  - Tenant's abc_sdlc repo  (project.yml + workflow file)
  - GitHub Actions           (calls our sdlc_actions)
  - onboard-client repo      (tenant onboarding via issues)

BASE VNET  (management plane)
  - FastAPI (Container App)  → only public-facing endpoint
  - PostgreSQL               → tenant registry
  - Key Vault                → tenant codes, DB admin password
  - Managed Identity         → passwordless auth for FastAPI

SHARED VNET  (shared tier tenants)
  - Service Bus              → index-request, change-request queues
  - Container App            → indexing worker
  - Cosmos DB                → pipeline state + graph
  - Azure AI Search          → vector search (filtered by tenant_id)

PRIVATE VNET  (one per dedicated tenant)
  - Same as shared but all resources dedicated to one tenant
  - Resource group: rg-sdlc-{org_code}-{env}
```

---

## Repos

| Repo | Purpose | Status |
|---|---|---|
| [sdlc-docs](https://github.com/IndentWork/sdlc-docs) | This repo — ecosystem documentation | Active |
| [sdlc-bootstrap](https://github.com/IndentWork/sdlc_bootstrap) | Bootstrap Azure foundation (storage, service principals) | Active |
| [sdlc-infra](https://github.com/IndentWork/sdlc-infra) | Terraform for all Azure infrastructure | Active |
| [sdlc-shared](https://github.com/IndentWork/sdlc-shared) | Shared config — resource names per environment | Planned |
| [sdlc-control-plane](https://github.com/IndentWork/sdlc-control-plane) | FastAPI management API + Alembic migrations | Planned |
| [sdlc-actions](https://github.com/IndentWork/sdlc-actions) | Reusable GitHub Actions for tenants | Planned |
| [onboard-client](https://github.com/IndentWork/onboard-client) | Tenant onboarding via GitHub issues | Planned |
| [sdlc-orchestrator](https://github.com/IndentWork/sdlc-orchestrator) | Handler A/B/C — Analyst, Coder, Reviewer agents | Deferred |

---

## Tenant Tiers

| Tier | Resources | Isolation |
|---|---|---|
| **Shared** | Shared VNet, shared Cosmos DB + AI Search filtered by tenant_id | Metadata isolation |
| **Dedicated** | Own VNet, own Cosmos DB + AI Search, own resource group | Full isolation |

---

## Naming Convention

All Azure resources follow: `{prefix}-sdlc-{scope}-{env}`

### Prefix meanings

| Prefix | Azure Resource | Example |
|---|---|---|
| `rg` | Resource Group | `rg-sdlc-base-dev` |
| `vnet` | Virtual Network | `vnet-sdlc-base-dev` |
| `snet` | Subnet | `snet-sdlc-base-dev-postgres` |
| `kv` | Key Vault | `kv-sdlc-base-dev` |
| `psql` | PostgreSQL Flexible Server | `psql-sdlc-base-dev` |
| `id` | Managed Identity | `id-sdlc-base-dev` |
| `cae` | Container App Environment | `cae-sdlc-base-dev` |
| `ca` | Container App | `ca-sdlc-base-dev` |
| `cr` | Container Registry | `crsdlcdev` (no hyphens — Azure rule) |
| `sb` | Service Bus | `sb-sdlc-shared-dev` |
| `cosmos` | Cosmos DB | `cosmos-sdlc-shared-dev` |
| `srch` | Azure AI Search | `srch-sdlc-shared-dev` |
| `sp` | Service Principal | `sp-sdlc-terraform-dev` |
| `dns-link` | Private DNS Zone VNet Link | `dns-link-psql-sdlc-base-dev` |

| scope | Meaning |
|---|---|
| `base` | Management plane (FastAPI, PostgreSQL, Key Vault) |
| `shared` | Shared tier tenant resources |
| `{org_code}` | Dedicated tenant (e.g. `abc123`) |

**Examples:**
```
rg-sdlc-base-dev
vnet-sdlc-base-dev
psql-sdlc-base-dev
kv-sdlc-base-dev
ca-sdlc-base-dev
crsdlcdev              (Container Registry — no hyphens allowed)
```

---

## GitHub Secrets

All secrets at org level (`IndentWork`), named `SDLC_{ENV}_{SERVICE}`:

```
SDLC_DEV_AZURE_CLIENT_ID_TERRAFORM
SDLC_DEV_AZURE_CLIENT_SECRET_TERRAFORM
SDLC_DEV_AZURE_TENANT_ID_TERRAFORM
SDLC_DEV_AZURE_SUBSCRIPTION_ID_TERRAFORM
```

---

## Bootstrap a New Environment

```bash
cd sdlc_bootstrap
cp .env.example .env
# Fill in SUBSCRIPTION_ID and LOCATION in .env
bash create.sh dev
```

This creates:
- Azure Storage Account for Terraform state
- Service Principal with Contributor + User Access Administrator roles
- GitHub org secrets for the Terraform pipeline

---

## Deploy Infrastructure

```bash
# Local
source /path/to/sdlc-infra/set_env.sh dev
cd sdlc-infra/environments/dev
terraform init && terraform plan && terraform apply
```

Or via GitHub Actions: `IndentWork/sdlc-infra` → Actions → `SDLC Infra Terraform` → Run workflow → select `DEV`

---

## Deploy Control Plane API

```bash
# Via pipeline in sdlc-control-plane repo
# Builds Docker image → pushes to ACR → updates Container App
# Alembic migrations run automatically on container startup
```

---

## Tenant Onboarding Flow

```
1. Tenant opens issue in onboard-client repo
2. Onboarding pipeline:
   - Generates unique tenant_code
   - Stores SHA256(code) in PostgreSQL
   - Stores plaintext code in Key Vault (admin retrieves + shares securely)
   - Creates Managed Identity + WIF federated credential
   - If dedicated → Terraform provisions Private VNet
3. Admin retrieves code from Key Vault → shares with tenant
4. Tenant stores SDLC_TENANT_KEY in their GitHub repo secrets
5. Tenant installs our GitHub App on their org
```

---

## Key Design Decisions

- **FastAPI is the only public endpoint** — everything else is inside VNet
- **Managed Identity for all auth** — no static passwords for app-to-service connections
- **GitHub App tokens** — for cloning private tenant repos (no PATs)
- **WIF for tenant auth** — tenants authenticate to Azure without static credentials
- **SHA256 in PostgreSQL** — tenant codes stored as hash, never plaintext
- **Alembic for migrations** — runs on Container App startup, lives in control-plane repo
- **sdlc-shared for config** — single source of truth for resource names across all pipelines

---

## PostgreSQL Role Design

| Role | Access | Used by |
|---|---|---|
| `pgadmin` | Superuser — emergency only | Admin via Key Vault password |
| `id-sdlc-base-dev` (Managed Identity) | DML only — SELECT, INSERT, UPDATE, DELETE | FastAPI at runtime |
| `migration_user` | DDL — CREATE/ALTER/DROP TABLE | Alembic migrations |
