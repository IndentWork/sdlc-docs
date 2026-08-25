# PROGRESS.md — What Has Been Built and What Is Pending

**Last updated:** 2026-08-23 (Session 1)

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

**Validated end-to-end on real GitHub — three scenarios:**
1. Happy path (issue #23) — full lifecycle from requirement to merged PR
2. Early stop (issue #24) — analyst detects already-implemented, closes cleanly
3. Rework loop (issue #25) — human rejects, feedback flows to Coder, re-approved

---

## This Repo — sdlc-platform/

**Goal:** Rebuild the prototype as an enterprise-standard multi-tenant Azure service.

**What is built here so far:** `sdlc_bootstrap` — Azure bootstrap scripts, pushed to GitHub.

### Repo Layout

Each subfolder below is its own git repository (not a monorepo):

```
sdlc-platform/
├── docs/                        ← this folder (architecture + ADRs, NOT a repo)
├── sdlc_bootstrap/              ← bootstrap scripts (create.sh dev/prod, destroy.sh dev/prod) ✓ DONE
├── sdlc-infra/                  ← Terraform for all VNets and Azure resources
├── control-plane/               ← FastAPI + PostgreSQL (tenant registry + routing API)
├── indexing-worker/             ← crawl + chunk + graph + upload logic
├── sdlc_actions/                ← reusable GitHub Actions (index + sdlc workflows)
├── onboard_client/              ← issue templates for tenant onboarding
└── sdlc-orchestrator/           ← Handler A, B, C (DEFERRED)
```

---

## Build Sequence — 9 Slices

Work in horizontal slices — each slice is a running, testable thing. Do not build in vertical layers. Do not chain multiple slices in one session.

| Slice | Description | Exit criteria | Status |
|---|---|---|---|
| **1** | Prove Azure OpenAI works — swap `ChatOpenAI` → `AzureChatOpenAI` in one agent, run end-to-end | One LLM call returns from Azure OpenAI, cost visible in Azure portal | **Not started** |
| **2** | Persistent state — `state_store.py` reads/writes SDLCState JSON to local file; split pipeline into handlers | Two handlers run in separate processes and share state via JSON file | **Not started** |
| **3** | Webhook trigger locally — ngrok + Flask/FastAPI endpoint; real GitHub webhook fires Handler A | Labelling an issue triggers Handler A locally, no CLI | **Not started** |
| **4** | Split HITL into PR-comment webhook — parse `/approve` and `/reject`, delete `input()` | PR comment triggers Handler C, pipeline merges or reworks | **Not started** |
| **5** | Move state to Cosmos DB — same handlers, same JSON, different backing store; add `tenant_id` + `project_id` | Two handlers in separate processes on different machines share state via Cosmos DB | **Not started** |
| **6** | Move vector store to Azure AI Search — replace ChromaDB client; every doc has tenant metadata; every query filters on it | Analyst runs against AI Search, returns equivalent results | **Not started** |
| **7** | Deploy handlers to Azure Container Apps — Service Bus wires them together, Managed Identity auth | End-to-end pipeline runs in Azure, driven by GitHub webhook, no local processes | **Not started** |
| **8** | GitHub App auth — replace PAT with GitHub App installation tokens | Pipeline runs against a real repo using GitHub App auth | **Not started** |
| **9** | Second tenant — prove isolation with negative tests | Two tenants coexist, tenant A cannot see tenant B's data | **Not started** |

---

## Session Log

| Session | Date | What was done | What is next |
|---|---|---|---|
| 0 | 2026-08-23 | Created `docs/START_HERE.md`. Read prototype and migration docs. Created `DOMAIN.md`, `PROGRESS.md`, `TASK_PLAN.md`. | Bootstrap infra. |
| 1 | 2026-08-23 | Designed full architecture (3 VNets, tenant onboarding, sdlc_actions pattern, WIF auth, GitHub App for private repo cloning, org-level secrets). Built `sdlc_bootstrap` — modular create/destroy scripts with dev/prod env support. Created Azure Storage Account (`stsdlcindentdev`) + SP (`sp-sdlc-terraform-dev`). Org secrets (`SDLC_DEV_AZURE_*_TERRAFORM`) auto-added to IndentWork via `gh` CLI. Tested `bash create.sh dev` end to end. Pushed to `IndentWork/sdlc_bootstrap`. | Start `sdlc-infra` Terraform repo. |

---

## Key Files to Read Before Writing Any Code

| File | What it teaches |
|---|---|
| `docs/START_HERE.md` (this repo) | Project rules, closed decisions, invariants, build sequence |
| `docs/DOMAIN.md` (this repo) | Roles, data models, business rules, guardrails |
| `/home/ashish/project/github_agentic_sdlc/docs/migrate_to_enterprise_standard_to_azure.md` | Migration decisions — event-driven handlers, Azure service choices, vector store tiers |
| `/home/ashish/project/github_agentic_sdlc/src/sdlc.py` | The orchestrator — see the full pipeline sequence |
| `/home/ashish/project/github_agentic_sdlc/src/state.py` | SDLCState — all fields with documented lifecycle |
| `/home/ashish/project/github_agentic_sdlc/src/agents/` | All four specialist agents |
| `/home/ashish/project/github_agentic_sdlc/src/tools/` | Tool modules — codebase, git, github, github_board |
| `/home/ashish/project/github_agentic_sdlc/src/guardrails/write_files.py` | File-write guardrail — understand before porting |
