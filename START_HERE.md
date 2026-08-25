# START HERE — sdlc-platform Handoff Document

**Purpose:** This document is written so that a fresh Claude Code session, opened from `/home/ashish/project/sdlc-platform/` with zero prior context, can read this file and pick up exactly where the previous session left off.

**Read this file first, in full, before doing anything.**

---

## 0. What This Project Is

We are building an **enterprise-standard multi-tenant Agentic SDLC platform** on Azure. A user submits a change request as a GitHub issue; specialist AI agents analyse, code, test, review it; a pull request is opened; a human approves; the change is merged and the code knowledge is re-indexed.

This is a **rebuild for production**, not a fresh project. A working local prototype already exists elsewhere on disk (see §2). We are moving to Azure with enterprise concerns: multi-tenancy, event-driven state, persistent workflows, GitHub App auth, real observability, and safe secrets.

**Working philosophy for this rebuild:** start building. Do not plan the whole enterprise stack upfront. Learn by doing. Carry two invariants (§7) that keep every future option open, and grow the rest incrementally.

---

## 1. Location of This Project

**This project:** `/home/ashish/project/sdlc-platform/`

Structure:
```
sdlc-platform/
├── docs/                    ← architecture, ADRs, session handoffs (THIS FOLDER)
│   └── START_HERE.md       ← this file
└── (repos below will be added as we build)
```

Each service will be its own **separate git repository** inside this folder. Do NOT create a monorepo. Each repo has its own `pyproject.toml`, its own CI, its own lifecycle. Draft layout in §8.

---

## 2. Where the Working Prototype Lives (READ IT)

**Prototype path:** `/home/ashish/project/github_agentic_sdlc/`

The prototype is a fully working local Python pipeline that has been validated end-to-end on real GitHub. It contains all the concepts we need to port to Azure.

### Must-read files before writing any code

Read these in this order to understand what already works:

| File | What it teaches |
|---|---|
| `docs/notes/sdlc_process.md` | The 20-step SDLC lifecycle with visual flow chart and decision branches |
| `docs/project_writeup.md` | Comprehensive component-by-component reference for everything in the prototype |
| `docs/migrate_to_enterprise_standard_to_azure.md` | The migration plan we developed — read this in full, it's the primary reference for THIS project |
| `docs/journal.md` (Sessions 1-3) | The learning journal — decisions and rationale behind the prototype |
| `docs/interview.md` | Core concepts in interview-ready form (agent loop, state vs memory, specialist agent pattern, etc.) |
| `docs/notes/codeatlas_capstone_plan.md` | Original 20-iteration capstone plan |
| `GitHub_Agentic_SDLC_Enterprise_Architecture_Complete.md` (in prototype repo root) | External enterprise architecture doc (30 sections) — read it for the fuller enterprise picture |

### Prototype code to skim (understand the shape, don't copy blindly)

| Path | Role |
|---|---|
| `src/sdlc.py` | `SDLCPipeline` class — the orchestrator. Read `run()` to see the whole sequence |
| `src/state.py` | `SDLCState` class — all state fields with documented lifecycle |
| `src/agents/analyst.py` | Read-only investigator agent |
| `src/agents/coder.py` | Writes code, runs tests, commits, pushes (with retry caps) |
| `src/agents/reviewer.py` | Reads PR diff, returns approved/rejected |
| `src/agents/synthesizer.py` | Turns step results into a final response |
| `src/tools/codebase.py` | `@tool` functions: search, read, write, run tests, memory |
| `src/tools/git.py` | Local git operations with protected-branch guardrails |
| `src/tools/github.py` | GitHub REST API tools with idempotency/guardrails |
| `src/tools/github_project_board.py` | GraphQL for Projects v2 board |
| `src/hitl/approve_pr.py` | Terminal HITL — will be REPLACED by PR-comment webhook |
| `src/guardrails/write_files.py` | Deterministic file-write safety checks |
| `src/observability/tracer.py`, `reporter.py` | Token/cost/latency trace across a run |
| `src/evaluation/*` | Two-layer evaluation: deterministic + LLM judge |

**Trust the prototype for what already works.** Do not rebuild what's proven; port it.

---

## 3. What the Prototype Already Does (Working Today)

- Takes a natural-language requirement from the CLI
- Creates a GitHub issue in the `change_management` repo
- Adds the issue to a GitHub Project board and moves it through columns (Backlog → Ready → In Progress → In Review → Done)
- Runs Analyst agent (read-only) → decides feasible / already_implemented / not_feasible
- Creates a feature branch (naming convention: `{type}/issue-{N}-{title}-agent`)
- Runs Coder agent → reads files, writes minimal changes, runs tests, retries on failure (hard cap 3), commits, pushes
- Opens a PR linking `Closes #N`
- Runs Reviewer agent → reads PR diff, returns approved or rejected + reason
- Blocks for human APPROVE/REJECT via terminal
- Merges PR, closes issue, archives board task, deletes branch
- If human rejects: feedback flows back to Coder, cycle repeats (max 2 rework rounds)
- Full observability: tokens, cost, per-tool duration

**Validated on real GitHub across three scenarios:**
- Happy path (issue #23)
- Early stop when already implemented (issue #24)
- Rework loop with human feedback then approval (issue #25)

---

## 4. What This Rebuild Adds Over the Prototype

The prototype is a single-user synchronous script. The rebuild is a **multi-tenant asynchronous cloud service**. Here are the specific differences:

| Prototype (existing) | This rebuild |
|---|---|
| CLI trigger | GitHub webhook trigger + optional Claude Code skill front door |
| Terminal `input()` for HITL | `/approve` and `/reject <reason>` PR comment webhooks |
| In-memory state, dies with process | Persistent state in Cosmos DB, survives across handlers |
| One linear `run()` method | Three event-driven handlers (A: analyst+coder+PR, B: reviewer, C: HITL response / merge) |
| Local ChromaDB files | Azure AI Search — shared or dedicated per tenant |
| Local Neo4j | Cosmos DB Gremlin (deferred until reintroduced) |
| Personal access token | GitHub App with per-installation short-lived tokens |
| OpenAI direct | Azure OpenAI (GPT-4o-mini) via Managed Identity |
| `.env` file | Azure Key Vault |
| Single tenant, single project | Multi-tenant with shared and dedicated deployment modes |
| One repo | Multiple repos (control plane, orchestrator, indexing worker, webhook function, shared libs, IaC) |

---

## 5. Closed Architecture Decisions (Do Not Re-Litigate)

These are baked in. If a decision needs reversal, treat it as a big architectural change and document why.

**Front doors (two, converge on same GitHub state):**
1. **Claude Code + skill** — skill file uses `curl` + GitHub REST API with the user's PAT. No `gh` CLI, no MCP, no package install.
2. **Direct GitHub UI** — user creates issue with label `sdlc-agent`, comments `/approve` or `/reject <reason>` on PR.

**Pipeline is event-driven, split into three handlers:**
- **Handler A** — triggered by labelled-issue webhook. Runs analyst → create branch → coder → PR. Persists state, exits.
- **Handler B** — triggered by Handler A completion or PR-opened. Runs reviewer. Persists state. Sends notification. Exits.
- **Handler C** — triggered by `/approve` or `/reject` PR-comment webhook. Merges or triggers rework loop.

**Azure service choices:**
- Compute: Azure Container Apps (KEDA autoscale, min replicas = 0)
- Trigger ingress: GitHub webhook → API Management → Service Bus
- State store: Cosmos DB (persist `SDLCState` between handlers)
- Vector store: Azure AI Search — shared service default; dedicated per tenant on request. Same metadata schema in both.
- Graph store: **Deferred** — provisional choice is Cosmos DB for Apache Gremlin when reintroduced. Requires rewriting Cypher queries to Gremlin.
- LLM: Azure OpenAI Service (GPT-4o-mini)
- Secrets: Azure Key Vault, accessed via Managed Identity
- Message queue: Azure Service Bus (three queues: `change-request`, `pr-feedback`, `repository-changed`)

**Tenancy:**
- Multi-tenant from day one
- Two modes: SHARED (default, metadata-isolated) and DEDICATED (dedicated resources in the tenant's own resource group `rg-sdlc-private-<org_code>`)
- Same code path for both modes — only endpoint config differs
- Metadata schema on every data record: `org_code : project : repo : file_path : symbol_name`
- Access is enforced by GitHub upstream — not by the pipeline. The metadata filter is scoping, not permissions.

**Auth model:**
| Context | Auth |
|---|---|
| Claude Code skill (local user) | User's PAT via `GITHUB_TOKEN` |
| GitHub UI (local user) | Browser session |
| Azure pipeline (cloud) | GitHub App installation token |
| Azure ↔ Azure | Managed Identity |
| Pipeline → Azure OpenAI | Managed Identity + private endpoint |

**Retry caps enforced in code (not just prompt):**
- Coder test failures: 3 attempts
- Reviewer rejections: 2 rework rounds
- Human rejections: 2 rework rounds
- Agent iteration safety net: 30 iterations

---

## 6. Explicitly Deferred Decisions (Decide Only When Forced)

Do NOT preemptively design these. They come up in build and get decided with real data.

- Notification channel (email vs Teams vs both) and content format
- Approver policy (issue creator only vs any team member)
- Auto-close after N days without human response
- Exact retry counts, backoff, timeouts, circuit breakers
- Observability dashboards, alerts, SLOs
- CI/CD promotion strategy (blue/green, canary, etc.)
- HA/DR RPO/RTO values
- Compliance certifications (SOC 2, HIPAA, etc.) — driven by customer needs
- Cost governance budgets, quotas, chargeback
- Support model, on-call rotation, runbooks
- Full agent guardrails beyond the prototype's existing set
- Exact Cosmos partition key expression
- AI Search shared vs per-tenant index topology (both are supported by the metadata schema; pick when scale demands)
- Repository layout details beyond the draft in §8

**Rule:** if you feel yourself wanting to design one of these before code forces the question, stop. Write code. Come back when the code makes the question concrete.

---

## 7. Two Invariants — MAINTAIN FROM THE FIRST LINE OF CODE

These are not decisions to plan — they are coding disciplines. Every commit must respect them.

### Invariant 1: State must be JSON-serializable from day one

- `SDLCState` and all pipeline state must be composed of plain dicts, lists, strings, numbers, booleans, and None
- No classes with methods, no `datetime` objects (use ISO 8601 strings), no set(), no custom types
- Every handler must be able to: (a) load state from a JSON blob, (b) do work, (c) write state back as a JSON blob
- Payoff: persisting state in Cosmos DB later is a one-day job, not a rewrite

### Invariant 2: Every data record has `tenant_id` and `project_id` from day one

- Even if we start with `tenant_id = "default"` and `project_id = "default"`, the fields exist and are filtered on
- Every AI Search document, every Cosmos state record, every log line, every event message
- Every query includes `$filter=tenant_id eq '<X>' and project_id eq '<Y>'` (or equivalent)
- Payoff: adding the second tenant is trivial. Retrofitting multi-tenancy later is a rewrite.

**If a new session finds code that violates either invariant, fix it immediately.**

---

## 8. Draft Repo Layout Under sdlc-platform/

Each subfolder is its own git repository. Create them as they become needed — do not scaffold all at once.

```
sdlc-platform/
├── docs/                        ← this folder (architecture + ADRs, NOT a repo)
│
├── control-plane/               ← repo: tenant catalog, resource resolver
├── sdlc-orchestrator/           ← repo: three handlers (A, B, C) + pipeline logic
├── indexing-worker/             ← repo: post-merge incremental reindex
├── webhook-function/            ← repo: Azure Function receiving GitHub webhooks
├── platform-shared/             ← repo: shared libs (TenantContext, GitHubGateway, VectorRetrieval, etc.)
├── infrastructure/              ← repo: Terraform modules (control-plane, shared, per-tenant)
├── tenant-config-schema/        ← repo: infra.yml + access.yml JSON schemas
└── agents/                      ← repo (or several): analyst, coder, reviewer, synthesizer
```

**Start with two repos only:**
1. `sdlc-orchestrator/` — port the prototype's `SDLCPipeline` and agents here, adapted to the two invariants
2. `platform-shared/` — the interfaces (`TenantContext`, `GitHubGateway`, etc.) shared across services

Everything else is created when needed.

---

## 9. Build Sequence (Smallest Working Slices)

Do not build in vertical layers ("finish auth, then state, then …"). Build **horizontal slices** — each slice is a running, testable thing.

**Slice 1 — Prove Azure OpenAI works**
- New repo `sdlc-orchestrator`
- Swap `ChatOpenAI` → `AzureChatOpenAI` in one agent
- Run one agent end-to-end against Azure OpenAI
- Manual Managed Identity setup or use API key + Key Vault
- **Exit criteria:** one LLM call successfully returns from Azure OpenAI, cost visible in Azure portal

**Slice 2 — Persistent state**
- Add `state_store.py` that reads/writes `SDLCState` as JSON to a local file (later Cosmos DB)
- Refactor `SDLCPipeline` so each handler loads state at start, saves at end, exits
- Prove: run Handler A, exit process, run Handler B, state is preserved
- **Exit criteria:** two handlers run in separate processes and successfully share state via JSON file

**Slice 3 — Webhook trigger locally**
- Use `ngrok` or similar to expose a local Flask/FastAPI endpoint to GitHub
- Register a GitHub webhook for `issues.labeled`
- Handler A runs when a real GitHub issue is labeled
- **Exit criteria:** labeling an issue triggers Handler A locally, no CLI involved

**Slice 4 — Split HITL into PR-comment webhook**
- Subscribe to `issue_comment` webhook
- Parse `/approve` and `/reject <reason>`
- Delete terminal `input()` code
- **Exit criteria:** a PR comment triggers Handler C, pipeline merges or reworks accordingly

**Slice 5 — Move state to Cosmos DB**
- Same handlers, same JSON, different backing store
- Add `tenant_id` and `project_id` to every state record (invariant §7.2)
- **Exit criteria:** two handlers in separate processes on different machines can share state via Cosmos DB

**Slice 6 — Move vector store to Azure AI Search**
- Replace ChromaDB client with AI Search client in the analyst
- Every doc has `tenant_id`, `project_id`, `repo_id` metadata
- Every query filters on those
- **Exit criteria:** analyst runs against AI Search, returns equivalent results

**Slice 7 — Deploy handlers to Azure Container Apps**
- Container image for each handler
- Service Bus queues wire them together
- Managed Identity auth for all Azure services
- **Exit criteria:** end-to-end pipeline runs in Azure, driven by GitHub webhook, no local processes

**Slice 8 — GitHub App auth**
- Replace PAT with GitHub App installation tokens
- **Exit criteria:** pipeline runs against a real repo using GitHub App auth

**Slice 9 — Second tenant**
- Second `tenant_id` in the config
- Prove tenant isolation with negative tests
- **Exit criteria:** two tenants coexist, tenant A cannot see tenant B's data

Everything past Slice 9 is real-customer polish — observability, notifications, quotas, HA/DR, compliance. Handle when the first real customer forces the question.

---

## 10. Claude Working Rules for This Project

Copy or adapt these into `CLAUDE.md` at the sdlc-platform root when we create it.

- **Read `docs/START_HERE.md` first at the start of every conversation.** This file is authoritative.
- **Do not silently reverse a closed decision in §5.** If a reversal is genuinely needed, propose it explicitly and get agreement.
- **Do not invent answers for §6 deferred decisions.** Build interfaces/configuration seams so the decision can be added later.
- **Never violate §7 invariants.** JSON-serializable state, `tenant_id` on every record.
- **Reference the prototype (`/home/ashish/project/github_agentic_sdlc/`)** before writing new code — port, don't reinvent.
- **Explain before you code.** Small explanation of what you're about to build and why, then code. Not a lecture — 2-4 sentences.
- **Move slowly. One slice at a time.** Do not chain three slices in one session.
- **Keep the code simple.** Learning project rules still apply — no premature abstraction, no unnecessary frameworks.
- **Comment generously.** Explain WHY. Assume the next reader is a fresh Claude session with no context.
- **Ask before creating a git commit or pushing to a repo.** Never auto-commit.
- **Never add `Co-Authored-By: Claude` in commit messages.**
- **Do not update this handoff document silently.** If a session's decisions should update this file, ask first.

---

## 11. What the Next Session Should Do First

1. Read this file (`docs/START_HERE.md`) in full
2. Read the prototype references in §2 in the order given
3. Confirm understanding by summarizing back:
   - What the prototype does today
   - What the four closed architecture pillars are (front doors, event-driven handlers, tenancy, auth)
   - What the two invariants are
4. Ask which slice from §9 to start with (probably Slice 1)
5. Set up the first repo (`sdlc-orchestrator/`) with a minimal Python + `uv` project
6. Wire up Azure OpenAI credentials
7. Build Slice 1

Do NOT start building without steps 1-3.

---

## 12. Where the Full Migration Doc Lives

The primary architectural reference is:
`/home/ashish/project/github_agentic_sdlc/docs/migrate_to_enterprise_standard_to_azure.md`

That doc contains, in more depth than this handoff:
- The event-driven three-handler flow (§5)
- Azure service choices with rationale (§6)
- Vector store shared/private tiers with risks and mitigations (§7)
- Graph store provisional decision with a text diagram (§8)
- Notifications concept (§9)
- Code changes at a high level (§10)

**This handoff summarises it. That doc is the source of truth. If they disagree, that doc wins.**

---

## 13. Where the External Enterprise Architecture Doc Lives

`/home/ashish/project/github_agentic_sdlc/GitHub_Agentic_SDLC_Enterprise_Architecture_Complete.md`

A separate 30-section document from a broader enterprise architecture exercise. Covers everything: control plane, tenant enrollment, Access as Code, GitOps, HA/DR baselines, security tests. **Use it as a reference when a specific area needs depth.** Its own §26 lists items deliberately open — those match the deferred decisions in §6 of this handoff.

---

## 14. Session History (Update at End of Each Session)

**Session 0 — 2026-08-23** — this handoff document created. Nothing built yet. Empty `sdlc-platform/docs/` folder. Prototype at `/home/ashish/project/github_agentic_sdlc/` remains the working reference.

*(Append future sessions here as one-liner: date, what was built, what's next.)*

---

**End of START_HERE.md. If reading fresh: now open the migration doc in §12 and read it in full before writing any code.**
