# TASK_PLAN.md — Build Plan for SDLC Platform

**Last updated:** 2026-08-23 (Session 1)
**Status:** Bootstrap tested and complete. Next: sdlc-infra Terraform.

---

## Architecture Summary

Three VNets, one GitHub App, one tenant registry, one indexing pipeline.

```
OUTSIDE (public internet)
  - Client's abc_sdlc repo (project.yml + workflow file)
  - GitHub Actions (calls our sdlc_actions)
  - onboard_client repo (tenant onboarding via issues)

CONTROL PLANE VNET
  - FastAPI          → only public endpoint, handles all API calls
  - PostgreSQL       → tenant registry (tenant_id, tier, installation_id, endpoints)
  - Azure Key Vault  → stores tenant codes for admin retrieval
  - Azure Storage    → Terraform state backend

SHARED VNET (for shared tier tenants)
  - Azure Function       → receives /approve /reject PR comment webhooks
  - Azure Service Bus    → queues (index-request, change-request, pr-feedback)
  - Azure Container App  → indexing-worker + Handler A, B, C (later)
  - Azure Cosmos DB      → pipeline state + graph (all shared tenants)
  - Azure AI Search      → vector search (all shared tenants, filtered by tenant_id)

PRIVATE VNET (one per dedicated tenant → rg-sdlc-private-<short_code>)
  - Same as Shared VNet but all resources dedicated to one tenant
```

---

## Tenant Onboarding Flow

```
1. Tenant opens issue in onboard_client repo
   Fields: tenant_name, tier (shared/dedicated), github_org, github_repo, email

2. Onboarding pipeline:
   - Generates unique tenant_code
   - Stores SHA256(code) in PostgreSQL
   - Stores plaintext code in Azure Key Vault (admin retrieves + shares with tenant)
   - Creates Azure Managed Identity for tenant
   - Adds WIF federated credential (trust tenant's github_org/repo)
   - Records GitHub App installation_id in PostgreSQL
   - If dedicated → triggers Terraform to provision Private VNet
   - Posts on issue: "Onboarded. Contact admin for your tenant key."

3. Admin retrieves code from Key Vault → shares with tenant securely

4. Tenant stores SDLC_TENANT_KEY in their GitHub repo secrets

5. Tenant installs our GitHub App on their org
   → we store installation_id in PostgreSQL
```

## Tenant Project Registration Flow

```
1. Tenant creates abc_sdlc repo with:
   - project.yml  (project name + list of repos)
   - .github/workflows/sdlc.yml  (calls our sdlc_actions)

2. Tenant merges project.yml to main
   → GitHub Actions triggers
   → calls our sdlc_actions/index
   → POST /index to FastAPI with tenant_key + repo list

3. FastAPI:
   - Verifies SHA256(tenant_key) against PostgreSQL
   - Gets tenant's endpoints (AI Search, Cosmos DB)
   - Gets GitHub App installation_id
   - Puts message on Service Bus queue "index-request"

4. indexing-worker (Container App, inside VNet):
   - Picks up message from Service Bus
   - Gets short-lived GitHub token using installation_id
   - Clones each private repo using that token
   - Runs AST crawler → JSON
   - Chunks code → uploads to Azure AI Search (with tenant metadata)
   - Builds graph nodes + edges → uploads to Cosmos DB Gremlin
   - Marks repos as indexed in PostgreSQL

5. Ongoing: every merged PR triggers re-indexing of changed files only
```

---

## GitHub Repos to Create

| Repo | Purpose | Status |
|---|---|---|
| `sdlc_bootstrap` | Bootstrap scripts (create/destroy Azure foundation) | ✓ Done — `IndentWork/sdlc_bootstrap` |
| `sdlc-infra` | Terraform for all VNets and Azure resources | Not started |
| `control-plane` | FastAPI app + PostgreSQL schema | Not started |
| `indexing-worker` | Crawl + chunk + graph + upload logic | Not started |
| `sdlc_actions` | Reusable GitHub Actions (index + sdlc workflows) | Not started |
| `onboard_client` | Issue templates for tenant onboarding, add_repo, rotate_key, offboard | Not started |
| `sdlc-orchestrator` | Handler A, B, C — **deferred, build later** | Deferred |

---

## Build Tasks

### Phase 1 — GitHub Repos

- [x] **1.0** Create `sdlc_bootstrap` repo — `IndentWork/sdlc_bootstrap` ✓
  - Modular create/destroy scripts with dev/prod env support
  - Auto-creates SP + org-level GitHub secrets (`AZURE_*_SDLC_DEV/PROD`)
  - `.env` + `.env.example` + `.gitignore`

- [ ] **1.1** Create `onboard_client` repo
  - Issue templates: `onboard_new_tenant.yml`, `add_repo.yml`, `rotate_key.yml`, `offboard_tenant.yml`, `upgrade_to_dedicated.yml`

- [ ] **1.2** Create `sdlc_actions` repo
  - `index.yml` reusable workflow
  - `sdlc.yml` reusable workflow (placeholder for now)

- [ ] **1.3** Create `sdlc-infra` repo
  - Folder structure: `modules/control-plane`, `modules/shared-vnet`, `modules/private-vnet`, `environments/dev`, `environments/prod`

- [ ] **1.4** Create `control-plane` repo
  - FastAPI project skeleton with `uv`

- [ ] **1.5** Create `indexing-worker` repo
  - Python project skeleton with `uv`

---

### Phase 2 — Azure Foundation

- [x] **2.1** Azure Storage Account for Terraform state ✓
  - Resource group: `rg-sdlc-terraform-dev` / `rg-sdlc-terraform-prod`
  - Storage account: `stsdlcindentdev` / `stsdlcindentprod`
  - Container: `tfstate`
  - State files: `dev.tfstate`, `prod.tfstate`, `tenant-<code>.tfstate`

- [x] **2.2** Service Principal created ✓
  - `sp-sdlc-terraform-dev` / `sp-sdlc-terraform-prod`
  - Org secrets: `AZURE_CLIENT_ID_SDLC_DEV`, `AZURE_CLIENT_SECRET_SDLC_DEV`, etc.
  - Written to `.env` automatically

- [ ] **2.3** Configure Terraform backend in `sdlc-infra/`
  - `backend.tf` pointing to `stsdlcindentdev`, key = `dev.tfstate`

---

### Phase 3 — Terraform: Control Plane VNet

- [ ] **3.1** VNet + subnets for control plane
- [ ] **3.2** Azure Database for PostgreSQL Flexible Server
  - Inside VNet (private endpoint)
- [ ] **3.3** Azure Key Vault
  - Inside VNet (private endpoint)
  - RBAC: admin access only
- [ ] **3.4** Azure Container App (FastAPI)
  - VNet integrated
  - Public ingress on port 443 (only public-facing resource)
- [ ] **3.5** Managed Identity for FastAPI
  - Access to PostgreSQL, Key Vault, Service Bus

---

### Phase 4 — Terraform: Shared VNet

- [ ] **4.1** VNet + subnets for shared tier
- [ ] **4.2** Azure Service Bus namespace
  - Queues: `index-request`, `change-request`, `pr-feedback`, `repository-changed`
- [ ] **4.3** Azure Container App (indexing-worker)
  - VNet integrated
  - No public ingress
  - KEDA autoscale on Service Bus queue depth
- [ ] **4.4** Azure Cosmos DB (Gremlin API)
  - Private endpoint inside VNet
  - Collections: `tenants`, `pipeline_state`, `graph`
- [ ] **4.5** Azure AI Search
  - Private endpoint inside VNet
  - Shared index with tenant metadata filtering
- [ ] **4.6** Azure Function (webhook receiver)
  - Receives /approve /reject PR comment webhooks
  - VNet integrated

---

### Phase 5 — Terraform: Private VNet Module

- [ ] **5.1** Reusable Terraform module `modules/private-vnet`
  - Takes `org_code` as input variable
  - Creates: VNet + Service Bus + Container App + Cosmos DB + AI Search + Azure Function
  - All resources named with `org_code` (e.g. `rg-sdlc-private-<org_code>`)
  - Called by onboarding pipeline when a dedicated tenant is onboarded

---

### Phase 6 — Control Plane Code (FastAPI)

- [ ] **6.1** PostgreSQL schema
  - `tenants` table: tenant_id, name, tier, sha256_code, managed_identity_id, installation_id, status, created_at
  - `projects` table: tenant_id, project_name, repos[], last_indexed_at
  - `tenant_endpoints` table: tenant_id, ai_search_endpoint, cosmos_endpoint, service_bus_namespace

- [ ] **6.2** `POST /index` endpoint
  - Verify tenant_key (SHA256 lookup)
  - Put message on Service Bus `index-request` queue
  - Return job_id

- [ ] **6.3** `GET /index/{job_id}` endpoint
  - Return indexing job status

- [ ] **6.4** `POST /onboard` endpoint
  - Called by onboarding pipeline (not by tenants directly)
  - Creates tenant record, Managed Identity, WIF credential

- [ ] **6.5** `POST /add-repo` endpoint
  - Adds new federated credential to existing tenant's Managed Identity

---

### Phase 7 — GitHub Actions (sdlc_actions)

- [ ] **7.1** `index.yml` reusable workflow
  - Input: tenant_key (secret)
  - Reads `project.yml` from calling repo
  - Calls `POST /index` on control plane FastAPI
  - Polls `GET /index/{job_id}` until done
  - Reports result

- [ ] **7.2** `sdlc.yml` reusable workflow (placeholder)
  - Will trigger SDLC pipeline once sdlc-orchestrator is built

- [ ] **7.3** GitHub App
  - Create GitHub App in our org
  - Scopes: repo read, contents read, pull requests write, issues write
  - Store App ID + private key in Key Vault

---

### Phase 8 — Indexing Worker

- [ ] **8.1** Port AST crawler from prototype
  - `crawlers/python_ast.py` → adapted for Azure (no local file paths)

- [ ] **8.2** Port chunker from prototype
  - Replace ChromaDB client with Azure AI Search client
  - Add tenant metadata to every document
  - Every query filters on tenant_id + project + repo

- [ ] **8.3** Port graph loader from prototype
  - Replace Neo4j client with Cosmos DB Gremlin client
  - Rewrite Cypher queries as Gremlin queries
  - Add tenant metadata to every node + edge

- [ ] **8.4** Service Bus consumer
  - Listen to `index-request` queue
  - Parse message → run crawler + chunker + graph loader
  - Update job status in PostgreSQL

- [ ] **8.5** GitHub App token fetcher
  - Given installation_id → fetch short-lived token
  - Use token to clone private repos

- [ ] **8.6** Incremental indexing
  - Track file timestamps in PostgreSQL
  - On `repository-changed` queue message → re-index changed files only

---

### Phase 9 — sdlc-orchestrator (DEFERRED)

Handler A, B, C — Analyst → Coder → Reviewer → HITL → Merge.
Build after indexing pipeline is working and tenants can register their code.

---

## Key Design Rules (Do Not Break)

1. **Every data record has `tenant_id` and `project_id`** — even if default
2. **All state is JSON-serializable** — no datetime objects, no custom types
3. **FastAPI is the only public endpoint** — everything else is inside VNet
4. **GitHub App tokens only** — no static PATs for cloning repos
5. **SHA256 in PostgreSQL** — never store plaintext tenant codes in DB
6. **Container App does the upload** — GitHub Actions never touches private VNet resources directly
7. **WIF for tenant auth to Azure** — no static credentials for tenants
8. **Terraform for all infra** — no manual Azure portal provisioning (except the initial storage account for tfstate)

---

## Open Questions (Decide When Forced)

- Notification channel when indexing completes (email, Teams, GitHub comment?)
- Language support beyond Python (JavaScript, TypeScript, Go?)
- How tenant is notified when dedicated infra provisioning is complete
- GitHub App — one global app or one per environment (dev/prod)?
- PostgreSQL — single instance or HA with read replica?
