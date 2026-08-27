# PROGRESS.md — What Has Been Built and What Is Pending

**Last updated:** 2026-08-27 (Session 5)

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
| Neo4j knowledge graph (dropped from MVP — see migration doc) | Done (dropped) |
| Two-layer evaluation framework (deterministic + LLM judge) | Done |

---

## Repos Built

Each folder below is its own GitHub repo under `IndentWork`.

| Repo | Purpose | Status |
|---|---|---|
| [sdlc-docs](https://github.com/IndentWork/sdlc-docs) | Ecosystem docs, architecture, naming conventions | ✅ Live |
| [sdlc_bootstrap](https://github.com/IndentWork/sdlc_bootstrap) | Bootstrap Azure foundation (SP, storage, org secrets) | ✅ Live |
| [sdlc-infra](https://github.com/IndentWork/sdlc-infra) | Terraform — 7 modules, plan/apply/destroy pipelines | ✅ Live |
| [sdlc-shared](https://github.com/IndentWork/sdlc-shared) | Python package — BaseConfig + SelectConfig (v0.1.0) | ✅ Live |
| [sdlc-control-plane](https://github.com/IndentWork/sdlc-control-plane) | FastAPI + Alembic + SHA256 tenant auth | ✅ Live |
| [sdlc-local-dev](https://github.com/IndentWork/sdlc-local-dev) | Docker Compose local dev with mocks + init.sql | ✅ Live |
| [sdlc-e2e](https://github.com/IndentWork/sdlc-e2e) | 31 e2e tests (api / infra / security / flows) | ✅ Live |
| [sdlc-onboard-client](https://github.com/IndentWork/sdlc-onboard-client) | Onboarding via GitHub issues (WIP — parser needs fixing) | ⚠️ Partial |
| sdlc_actions | Reusable GitHub Actions for tenants | ❌ Not started |
| sdlc-orchestrator | Handler A/B/C — Analyst, Coder, Reviewer | ❌ Deferred |

---

## Azure Infrastructure Deployed (dev)

**Resource group:** `rg-sdlc-base-dev`

- `vnet-sdlc-base-dev` — VNet 10.0.0.0/16
  - `snet-sdlc-base-dev-postgres` (delegated /24)
  - `snet-sdlc-base-dev-container-app` (delegated /23)
- `kv-sdlc-base-dev` — Key Vault (RBAC)
- `id-sdlc-base-dev` — Managed Identity
- `psql-sdlc-base-dev` — PostgreSQL 16 with `sdlc` database
- `crsdlcdev` — Container Registry
- `cae-sdlc-base-dev` — Container App Environment
- `ca-sdlc-base-dev` — FastAPI Container App

**Ephemeral — created/destroyed as needed** to control cost.

---

## Pipelines

| Pipeline | Repo | Purpose |
|---|---|---|
| SDLC Infra Terraform | sdlc-infra | Plan → approve → apply, dev/prod |
| SDLC Infra Destroy | sdlc-infra | Plan destroy → approve → destroy |
| Deploy Control Plane | sdlc-control-plane | Build (if new) → push → deploy to Container App |
| E2E Tests | sdlc-e2e | 31 tests against deployed environment |
| Process Onboarding | sdlc-onboard-client | Issue → create tenant → store key in KV (WIP) |

---

## Session Log

| # | Date | What was done | Next |
|---|---|---|---|
| 0 | 2026-08-23 | Read prototype + migration docs, wrote DOMAIN.md/PROGRESS.md/TASK_PLAN.md | Bootstrap |
| 1 | 2026-08-23 | Designed 3-VNet architecture, built `sdlc_bootstrap`, created SP + storage | Start `sdlc-infra` |
| 2 | 2026-08-24 | Built `sdlc-infra` — 5 modules (RG/VNet/KV/MI/postgres) + pipelines | Container App module |
| 3 | 2026-08-25 | Built container-app + container-registry modules, `sdlc-shared`, `sdlc-control-plane` (FastAPI + Alembic + tenants CRUD), `sdlc-local-dev` (Docker), `sdlc-e2e` (28 tests) | Fix deploy pipeline |
| 4 | 2026-08-26 | Fixed deploy pipeline (registry auth), created `sdlc` database, all 27 e2e tests passing on Azure | Real POST /index |
| 5 | 2026-08-27 | Implemented SHA256 tenant auth (POST /tenants returns key, POST /index verifies). Refactored control-plane into schemas/security modules. All 31 e2e tests passing on Azure. Started `sdlc-onboard-client` — form + pipeline created, parser needs Python rewrite | Fix onboarding parser, then build shared-vnet + dedicated-vnet modules |

---

## What's Next

**Immediate:**
1. Rewrite onboarding pipeline using Python script (`scripts/process_onboard.py`) — the YAML parser action returned nulls
2. Fix duplicate trigger — only run on `opened` event

**After onboarding works:**
3. Add columns to `tenants` table: `org_code`, `tier`
4. Build `shared-vnet` Terraform module (Service Bus at minimum)
5. Wire FastAPI POST /index to put messages on Service Bus
6. Build `private-vnet` (dedicated) Terraform module — provisioned per dedicated tenant during onboarding
7. Build `sdlc-orchestrator` (Handler A/B/C) — port from prototype

---

## Key Files to Read Before Writing Code

| File | What it teaches |
|---|---|
| `sdlc-docs/README.md` | Ecosystem overview, naming conventions, all repos |
| `sdlc-docs/architecture/DOMAIN.md` | Roles, data models, business rules, guardrails |
| `sdlc-docs/START_HERE.md` | Project rules, closed decisions, invariants |
| `/home/ashish/project/github_agentic_sdlc/docs/migrate_to_enterprise_standard_to_azure.md` | Migration decisions — event-driven handlers, Azure choices, vector store tiers |
| `/home/ashish/project/github_agentic_sdlc/src/sdlc.py` | Prototype orchestrator — full pipeline sequence |
| `/home/ashish/project/github_agentic_sdlc/src/agents/` | Four specialist agents to port |
