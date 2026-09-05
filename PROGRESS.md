# PROGRESS.md — What Has Been Built and What Is Pending

**Last updated:** 2026-09-05 (Session 6)

---

## Working Prototype (Reference — Not This Repo)

**Location:** `/home/ashish/project/github_agentic_sdlc/`

A fully validated local Python pipeline. Everything below is working on real GitHub today:

| Capability | Status |
|---|---|
| Create GitHub issue from requirement | Done |
| Add issue to GitHub Project board (GraphQL) | Done |
| Move board through columns (Backlog → Ready → In Progress → In Review → Done) | Done |
| Analyst agent — investigate codebase, judge feasibility | Done |
| Create feature branch (local + GitHub) | Done |
| Coder agent — implement, test, commit, push (3-attempt retry) | Done |
| Open PR with `Closes #N` | Done |
| Reviewer agent — independent PR review | Done |
| Terminal HITL (blocking `input()`) | Done |
| Rework loop — feedback → Coder → Reviewer (2-round cap) | Done |
| Merge PR, close issue, archive board task, delete branch | Done |
| Synthesizer — final summary posted as issue comment | Done |
| Full observability — tokens, cost, per-tool duration | Done |
| ChromaDB vector search (semantic file locator) | Done |
| Neo4j knowledge graph | Done |
| Two-layer evaluation framework (deterministic + LLM judge) | Done |

---

## Repos Built

Each folder below is its own GitHub repo under `IndentWork`.

| Repo | Purpose | Status |
|---|---|---|
| [sdlc-docs](https://github.com/IndentWork/sdlc-docs) | Ecosystem docs, architecture, naming conventions | ✅ Live |
| [sdlc_bootstrap](https://github.com/IndentWork/sdlc_bootstrap) | Bootstrap Azure foundation (SP, storage, org secrets) | ✅ Live |
| [sdlc-infra](https://github.com/IndentWork/sdlc-infra) | Terraform — modules, plan/apply/destroy pipelines | ✅ Live |
| [sdlc-shared](https://github.com/IndentWork/sdlc-shared) | Python package — BaseConfig + SelectConfig (v0.1.0) | ✅ Live |
| [sdlc-control-plane](https://github.com/IndentWork/sdlc-control-plane) | FastAPI + OIDC auth + Service Bus publisher | ✅ Live |
| [sdlc-local-dev](https://github.com/IndentWork/sdlc-local-dev) | Docker Compose local dev with mocks + init.sql | ✅ Live |
| [sdlc-e2e](https://github.com/IndentWork/sdlc-e2e) | 31 e2e tests (api / infra / security / flows) | ✅ Live |
| [sdlc-actions](https://github.com/IndentWork/sdlc-actions) | Reusable GitHub Actions for tenants | ✅ Live |
| [sdlc-worker-tester](https://github.com/IndentWork/sdlc-worker-tester) | Test worker — Service Bus → Storage | ✅ Live |
| [sdlc-worker-indexing](https://github.com/IndentWork/sdlc-worker-indexing) | Indexing worker — repos → AI Search + Cosmos DB | ✅ Live |
| sdlc-worker-orchestrator | Handler A/B/C — Analyst, Coder, Reviewer | ❌ Not started |

---

## Azure Infrastructure Deployed (dev)

**Base scope — `rg-sdlc-base-dev`:**
- `vnet-sdlc-base-dev` — VNet 10.0.0.0/16
- `kv-sdlc-base-dev` — Key Vault (RBAC)
- `id-sdlc-base-dev` — Base Managed Identity
- `psql-sdlc-base-dev` — PostgreSQL 16
- `crsdlcdev` — Container Registry (ACR)
- `cae-sdlc-base-dev` — Container App Environment
- `ca-sdlc-base-dev` — FastAPI control plane
- `ca-sdlc-tester-dev` — Tester worker (manual bootstrap)
- `ca-sdlc-indexing-dev` — Indexing worker (manual bootstrap)

**Shared scope — `rg-sdlc-shared-dev`:**
- `sb-sdlc-shared-dev` — Service Bus (topic: sdlc-events)
  - Subscription: `tester` (filter: test_storage, upload_sdlc)
  - Subscription: `indexing` (filter: index_repos)
- `stsdlcshareddev` — Storage Account
  - Container: `sdlc` (path: `{resource_code}/{github_org}/...`)
- `id-sdlc-shared-dev` — Shared Managed Identity
- `kv-sdlc-shared-dev` — Shared Key Vault
- `cosmos-sdlc-shared-dev` — Cosmos DB (nodes + edges containers)
- `srch-sdlc-shared-dev` — Azure AI Search (code-chunks index)

---

## GitHub App

- **App name:** indentwork-sdlc
- **App ID:** 4826692
- **Private key:** stored in `kv-sdlc-base-dev` as `github-app-private-key`
- **Installation:** sdlc-tenant org (installation looked up dynamically via GitHub API)

---

## Storage Path Convention

All blobs in `sdlc` container follow this hierarchy:
```
{resource_code}/{github_org}/sdlc.yml              ← config
{resource_code}/{github_org}/index/{project}/{repo}/{file}.json  ← indexing checkpoints
```

`resource_code = SHA256(github_org)[:8]` — stable internal identifier.

---

## AI Search Schema

Index: `code-chunks`
- `doc_key` — base64 URL-safe encoded document key
- `resource_code`, `github_org`, `project`, `repo`, `file` — filterable metadata
- `type` (function/class/method), `name`, `class_name` — filterable metadata
- `content` — searchable text (signature + docstring + code)
- `content_vector` — vector field (OpenAI embeddings — not yet wired)

Filtering pattern:
```python
filter = "resource_code eq 'b310545b' and repo eq 'cart-service' and type eq 'function'"
```

---

## Cosmos DB Schema

Database: `sdlc`
- `nodes` container — File, Function, Class, Method nodes (partition: `/resource_code`)
- `edges` container — CALLS, IMPORTS, HAS_METHOD relationships (partition: `/resource_code`)

Node/edge IDs: base64 URL-safe encoded compound path.

---

## Bootstrap Scripts (sdlc-infra/scripts/)

Run once per environment in this order:
```bash
./scripts/bootstrap-acr-access.sh {env}       # AcrPull for shared MI on ACR
./scripts/bootstrap-workers.sh {env}           # Create Container Apps with correct clientId
./scripts/bootstrap-subscriptions.sh {env}     # Create Service Bus subscriptions
```

---

## Pipelines

| Pipeline | Repo | Purpose |
|---|---|---|
| SDLC Infra Terraform | sdlc-infra | Plan → approve → apply, dev/prod |
| Terraform Plan Only | sdlc-infra | Read-only plan — review changes safely |
| Deploy Control Plane | sdlc-control-plane | Build → push → deploy to Container App |
| Deploy sdlc-worker-tester | sdlc-worker-tester | Build → push → deploy |
| Deploy sdlc-worker-indexing | sdlc-worker-indexing | Build → push → deploy |
| Upload SDLC Config | sdlc-config (tenant) | Upload sdlc.yml to Storage |
| Trigger Indexing | sdlc-config (tenant) | Trigger indexing worker via API |

---

## End-to-End Flow (Working in dev)

```
1. Tenant pushes sdlc.yml → Upload SDLC Config workflow
   → POST /tenant/upload_sdlc (OIDC auth)
   → Service Bus (action=upload_sdlc)
   → Tester worker saves to Storage: {resource_code}/{github_org}/sdlc.yml

2. Tenant triggers "Trigger Indexing" workflow
   → POST /tenant/index (OIDC auth)
   → Service Bus (action=index_repos)
   → Indexing worker:
     a. Read sdlc.yml from Storage
     b. Get GitHub App installation token
     c. For each repo in parallel:
        - Fetch file tree from GitHub API
        - Compare SHA vs checkpoint (skip unchanged)
        - Crawl .py files via AST parser
        - Upsert chunks → Azure AI Search
        - Upsert nodes/edges → Cosmos DB
        - Save checkpoint → Storage
```

---

## Session Log

| # | Date | What was done | Next |
|---|---|---|---|
| 0 | 2026-08-23 | Read prototype + migration docs, wrote DOMAIN.md/PROGRESS.md/TASK_PLAN.md | Bootstrap |
| 1 | 2026-08-23 | Designed 3-VNet architecture, built `sdlc_bootstrap`, created SP + storage | Start `sdlc-infra` |
| 2 | 2026-08-24 | Built `sdlc-infra` — 5 modules (RG/VNet/KV/MI/postgres) + pipelines | Container App module |
| 3 | 2026-08-25 | Built container-app + container-registry modules, `sdlc-shared`, `sdlc-control-plane` (FastAPI + Alembic + tenants CRUD), `sdlc-local-dev` (Docker), `sdlc-e2e` (28 tests) | Fix deploy pipeline |
| 4 | 2026-08-26 | Fixed deploy pipeline (registry auth), created `sdlc` database, all 27 e2e tests passing on Azure | Real POST /index |
| 5 | 2026-08-27 | Implemented SHA256 tenant auth (POST /tenants returns key, POST /index verifies). Refactored control-plane into schemas/security modules. All 31 e2e tests passing on Azure. Started `sdlc-onboard-client` | Fix onboarding parser |
| 6 | 2026-09-05 | Built GitHub OIDC auth, Service Bus topic+subscription pattern, GitHub App integration, sdlc-worker-tester, sdlc-worker-indexing (AST crawler → AI Search + Cosmos DB), sdlc-actions (trigger-test, upload-sdlc, trigger-indexing). Full indexing pipeline working end-to-end in dev. | Build sdlc-worker-orchestrator (agents) |

---

## What's Next

**Immediate:**
1. Wire OpenAI embeddings into indexing worker (content_vector field in AI Search)
2. Build `sdlc-worker-orchestrator` — port Analyst, Coder, Reviewer agents from prototype
3. Add worker Container Apps to Terraform (currently manual via bootstrap-workers.sh)
4. Move shared boilerplate (Service Bus listener, JSON logging) to `sdlc-shared` package

**Key decisions made:**
- Single `sdlc` Storage container — paths handle all data separation (no new containers needed)
- `resource_code` as stable tenant key (not tier) — zero migration cost shared → dedicated
- Crawler plugin architecture (BaseCrawler + registry) — add new file types without touching orchestrator
- GitHub file SHA for incremental indexing (not timestamp)
- Base64 URL-safe encoding for AI Search doc_key and Cosmos DB IDs

---

## Key Files to Read Before Writing Code

| File | What it teaches |
|---|---|
| `sdlc-docs/README.md` | Ecosystem overview, naming conventions, all repos |
| `sdlc-docs/architecture/DOMAIN.md` | Roles, data models, business rules, guardrails |
| `sdlc-docs/architecture/INDEXING_WORKER.md` | Indexing worker architecture and build plan |
| `sdlc-docs/DEVELOPER.md` | Sequential deployment guide (10 steps) |
| `/home/ashish/project/github_agentic_sdlc/src/sdlc.py` | Prototype orchestrator — full pipeline sequence |
| `/home/ashish/project/github_agentic_sdlc/src/agents/` | Four specialist agents to port |
| `/home/ashish/project/github_agentic_sdlc/crawlers/python_ast.py` | AST crawler — already ported to sdlc-worker-indexing |
