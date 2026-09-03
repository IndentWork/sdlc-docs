# INFRASTRUCTURE.md — Azure Resource Inventory

Source of truth for what Azure resources exist per scope, why they exist, and what is deferred.

---

## Scopes

The platform has three infrastructure scopes. Each scope has its own Terraform state file.

| Scope | State file | Created when |
|---|---|---|
| `base` | `base.tfstate` | Once at platform start |
| `shared` | `shared.tfstate` | Once at platform start |
| `{resource_code}` | `{resource_code}.tfstate` | Per dedicated tenant at onboarding |

---

## Base Scope — Control Plane

Management plane. Houses the FastAPI control-plane API, tenant registry, and shared platform services.
All names follow: `{resource}-sdlc-base-{env}`

| Resource | Name (dev) | Purpose |
|---|---|---|
| Resource Group | `rg-sdlc-base-dev` | Container for all base resources |
| VNet | `vnet-sdlc-base-dev` | Network — 10.0.0.0/16 |
| Managed Identity | `id-sdlc-base-dev` | FastAPI authenticates to Azure services without credentials |
| Key Vault | `kv-sdlc-base-dev` | Platform secrets — postgres admin password, etc. |
| PostgreSQL | `psql-sdlc-base-dev` | Control-plane DB — tenants, projects, repos, resource mappings |
| Container Registry | `crsdlcdev` | Docker image registry for all Container Apps across all scopes |
| Container App Environment | `cae-sdlc-base-dev` | Runtime platform for base Container Apps |
| Container App | `ca-sdlc-base-dev` | FastAPI control-plane API |

**FastAPI control-plane API responsibilities:**
- Tenant onboarding and registry (`POST /tenants`)
- GitHub OIDC token validation for incoming requests
- Routes tenant requests to the correct Service Bus (shared or dedicated) based on `tier` + `resource_code`
- `PUT /tenants/repos` — syncs allowed repos from `sdlc-config/sdlc.yml`

**Flow — tenant pipeline calling the API:**
```
Tenant GitHub pipeline (OIDC token)
        ↓
POST /index → FastAPI (base)
        ↓
Validate token → look up tenant tier + resource_code
        ↓
tier = shared   → enqueue on sb-sdlc-shared-{env}   / repo-index
tier = dedicated → enqueue on sb-sdlc-{resource_code}-{env} / repo-index
        ↓
Indexing Worker picks up → reads repo → AI Search + Cosmos
```

---

## Shared Scope — Shared Data Plane

Shared infrastructure used by all `tier = shared` tenants.
All names follow: `{resource}-sdlc-shared-{env}`

| Resource | Name (dev) | Purpose |
|---|---|---|
| Resource Group | `rg-sdlc-shared-dev` | Container for all shared resources |
| VNet | `vnet-sdlc-shared-dev` | Network isolation — 10.1.0.0/16 |
| Managed Identity | `id-sdlc-shared-dev` | Passwordless auth for shared workers |
| Key Vault | `kv-sdlc-shared-dev` | Shared secrets |
| Service Bus | `sb-sdlc-shared-dev` | Message queuing — see queue list below |
| Storage Account | `stsdlcshareddev` | Blob storage — audit records, dormant workflow checkpoints, agent traces |
| Azure AI Search | `srch-sdlc-shared-dev` | Code chunks + vectors. Replaces local ChromaDB. Every document filtered by `tenant_id` + `project_id` |
| Cosmos DB | `cosmos-sdlc-shared-dev` | Code relationship graph (Gremlin). Cross-repo traversal scoped to project. Every node/edge carries `tenant_id` + `project_id` + `repo_id` |
| Container App Environment | `cae-sdlc-shared-dev` | Runtime platform for shared workers |
| Indexing Worker | `ca-sdlc-indexing-shared-dev` | Reads repos from GitHub, chunks + embeds code, writes to AI Search + Cosmos |

**Service Bus queues:**

| Queue | Consumer | Purpose |
|---|---|---|
| `repo-index` | Indexing Worker | Index a repo into AI Search + Cosmos. Triggered when tenant registers repos via `sdlc-config` or after a PR merge |

**Deferred queues (added when orchestrator is built):**

| Queue | Consumer | Purpose |
|---|---|---|
| `change-request` | SDLC Orchestrator | New GitHub issue → trigger Analyst → Coder → Reviewer |
| `pr-feedback` | SDLC Orchestrator | Human requests changes on PR → resume workflow |
| `repository-changed` | Indexing Worker | Post-merge incremental re-index of changed files |

**Deferred resources (added when orchestrator is built):**

| Resource | Purpose |
|---|---|
| Redis Cache | Hot workflow state between Handler A → B → C |
| SDLC Orchestrator Container App | Runs Handler A/B/C → Analyst, Coder, Reviewer agents |

**Tenant isolation in shared tier:**
Every AI Search query and Cosmos traversal is deterministically scoped by `tenant_id` + `project_id` injected by the platform. The LLM cannot remove or override these filters.

---

## Dedicated Scope — Per-Tenant Data Plane

One set of resources per dedicated tenant. Named using the tenant's `resource_code` (SHA256(github_org)[:8]).
All names follow: `{resource}-sdlc-{resource_code}-{env}`

| Resource | Name (dev) | Purpose |
|---|---|---|
| Resource Group | `rg-sdlc-{resource_code}-dev` | Container — drop this RG to fully offboard the tenant |
| VNet | `vnet-sdlc-{resource_code}-dev` | Dedicated network isolation |
| Managed Identity | `id-sdlc-{resource_code}-dev` | Passwordless auth for this tenant's workers |
| Key Vault | `kv-sdlc-{resource_code}-dev` | Secrets scoped to this tenant |
| Service Bus | `sb-sdlc-{resource_code}-dev` | Same queues as shared, dedicated to this tenant |
| Storage Account | `stsdlc{resource_code}dev` | Blob storage dedicated to this tenant |
| Azure AI Search | `srch-sdlc-{resource_code}-dev` | Dedicated code vector index — no shared-tier filtering needed |
| Cosmos DB | `cosmos-sdlc-{resource_code}-dev` | Dedicated code relationship graph |
| Container App Environment | `cae-sdlc-{resource_code}-dev` | Dedicated runtime platform |
| Indexing Worker | `ca-sdlc-indexing-{resource_code}-dev` | Dedicated indexing worker |

**Deferred (same as shared):** Redis, SDLC Orchestrator Container App.

**Dedicated tenant Terraform:**
Each dedicated tenant gets their own `{resource_code}.tfvars` file and `{resource_code}.tfstate`.
Created automatically during onboarding when `tier = dedicated`.

---

## Terraform State Layout

```
Storage account: stsdlcindentdev  (dev)
Container: tfstate

  base.tfstate              ← control plane
  shared.tfstate            ← shared data plane
  a3f1c2b4.tfstate          ← dedicated tenant
  b9c2e1f7.tfstate          ← another dedicated tenant
```

---

## Resource Routing — How FastAPI Picks the Right Service Bus

`resource_code` and `tier` are stored in the `tenants` table (control-plane DB).
FastAPI derives the Service Bus endpoint at runtime — no hardcoded endpoints:

```python
if tenant["tier"] == "shared":
    sb_namespace = f"sb-sdlc-shared-{env}"
else:
    sb_namespace = f"sb-sdlc-{tenant['resource_code']}-{env}"
```

No extra DB column needed. `tier` + `resource_code` is sufficient to resolve any tenant's resources.

---

## What Does NOT Live in Shared or Dedicated

| Resource | Where it lives | Why |
|---|---|---|
| PostgreSQL (control-plane DB) | Base | Tenant config, project/repo registry, resource mappings — platform data, not tenant data |
| Container Registry | Base | One ACR serves all scopes — images are not tenant-specific |
| Azure OpenAI | Base or shared | One LLM endpoint serves all agents — model calls are stateless |
| GitHub App registration | Platform (external) | One App installation per tenant GitHub org |
