# DOMAIN.md — Roles, Rules, Data Models, Business Logic

**Source of truth for:** what this system does, who does what, what the data looks like, and the rules that govern behaviour.

**Prototype location:** `/home/ashish/project/github_agentic_sdlc/`
**Migration reference:** `/home/ashish/project/github_agentic_sdlc/docs/migrate_to_enterprise_standard_to_azure.md`

---

## 1. What This System Does

A user submits a plain-English software change requirement. The system:

1. Creates a GitHub issue and tracks it on a GitHub Project board
2. An **Analyst** agent investigates the codebase and decides if the change is feasible
3. A **Coder** agent implements the change on a feature branch, runs tests, commits and pushes
4. A PR is opened automatically
5. A **Reviewer** agent independently reviews the PR and approves or rejects it
6. The human is notified and approves or rejects via a GitHub PR comment (`/approve` or `/reject <reason>`)
7. On approval: PR is merged, issue closed, board task archived, branch deleted
8. On rejection: feedback flows back to the Coder; the cycle repeats (capped at 2 rework rounds)

**The pipeline is entirely driven by GitHub state.** There is no separate API. Any front door that writes to GitHub (Claude Code skill, GitHub UI, Slack bot) triggers the same pipeline.

---

## 2. Roles

### 2.1 Specialist Agents (LLM-powered)

| Agent | Responsibility | Tools available | Cannot do |
|---|---|---|---|
| **Analyst** | Investigate codebase, judge feasibility, identify affected files | `search_code` (vector), `get_dependencies` (graph), `read_file` | Write files, run tests, commit |
| **Coder** | Implement the change, run tests, commit, push | `read_file`, `write_file`, `run_tests`, `commit_changes`, `push_branch` | Open PRs, access GitHub API, merge |
| **Reviewer** | Review PR diff independently, approve or reject | `get_pr_diff` | Write files, merge, see Coder's reasoning |
| **Synthesizer** | Convert raw step results into a clean summary | (no tools — pure LLM) | Take any action |

**Independence is a design invariant.** The Reviewer sees only the requirement and the PR diff — never the Coder's reasoning. This ensures a genuine second opinion, not a rubber stamp.

### 2.2 Orchestrator (No LLM)

`SDLCPipeline` (prototype) / Handler A, B, C (production) — pure Python coordination logic.

- Calls agents in sequence
- Owns retry loops (caps test failures at 3, reviewer rejections at 2, human rejections at 2)
- Owns all GitHub/git operations that do not require reasoning (create issue, move board, merge PR)
- Never calls an LLM directly — only delegates to agents

**Why no LLM in the orchestrator?** Mechanical, deterministic-outcome steps (create issue, merge PR) do not benefit from LLM reasoning. Using LLM there wastes tokens and adds non-determinism.

### 2.3 Human

The human is in the loop at exactly one point: **PR approval**. They are notified when the Reviewer has approved and the PR is ready. They respond via:
- GitHub PR comment: `/approve` → merge
- GitHub PR comment: `/reject <reason>` → rework cycle begins

The human is **not** the only safety gate — the Reviewer is a second AI gate before the human sees the PR.

---

## 3. Data Models

### 3.1 SDLCState (Pipeline Run State)

The complete in-memory state of one pipeline run. **Must be JSON-serializable at all times** (Invariant 1).

```python
class SDLCState:
    # Identity — set at pipeline start
    project:      str          # e.g. "codeatlas/ecommerce"
    requirement:  str          # plain-English user request

    # Set by Analyst
    repo:         str          # which service repo to change

    # Set by orchestrator — GitHub references
    github_issue: int          # issue number in change_management repo
    branch:       str          # feature branch name
    pr_number:    int          # PR number

    # Set by agents
    analysis:       dict       # analyst output (see §3.2)
    coder_result:   dict       # coder output (see §3.3)
    files_modified: list[str]  # relative paths changed
    review_result:  dict       # reviewer output (see §3.4)
    human_approval: dict       # {approved: bool, reason: str}

    # Set at pipeline end
    final_response: str        # synthesizer's summary (posted on issue)
    status:         str        # "started" | "done" | "failed" | "already_implemented" | "not_feasible"
```

**JSON-serialization rule:** No `datetime` objects — use ISO 8601 strings. No `set()`. No classes with methods embedded in state. No custom types.

### 3.2 Analyst Output

```json
{
  "status": "feasible | already_implemented | not_feasible",
  "repo": "service repo name (e.g. order_service)",
  "files_to_change": ["relative/path/to/file.py"],
  "summary": "what needs to change and why",
  "branch_type": "feat | fix | hotfix | docs | test | chore | refactor",
  "branch_title": "short-hyphenated-lowercase (max 30 chars)"
}
```

**Three status values with distinct handling:**
- `feasible` → orchestrator continues to Coder
- `already_implemented` → orchestrator closes issue cleanly, stops
- `not_feasible` → orchestrator flags to human, closes issue, stops

### 3.3 Coder Output

```json
{
  "status": "done | failed",
  "files_modified": ["relative/path/file.py"],
  "commits": [
    {
      "message": "feat: add empty cart validation",
      "changes": "added validation block to create_order, updated test",
      "test_results": "8/8 pass"
    }
  ]
}
```

### 3.4 Reviewer Output

```json
{
  "status": "approved | rejected",
  "reason": "specific explanation citing files and line numbers"
}
```

### 3.5 Long-Term Memory

Stable facts about a project stored across pipeline runs. Scoped by project string. JSON file (`project_memory.json`) in prototype; migrated to Cosmos DB in production.

```json
{
  "codeatlas/ecommerce": [
    {"key": "test_framework", "value": "pytest"},
    {"key": "test_location",  "value": "tests/"},
    {"key": "naming_convention", "value": "snake_case"}
  ]
}
```

**State vs Memory:**
| | State | Memory |
|---|---|---|
| Scope | One pipeline run | Across all runs, all time |
| Lifetime | Dies when run ends | Persists indefinitely |
| Examples | `branch=feat/issue-42-*-agent`, `pr_number=7` | `test_framework=pytest`, `test_location=tests/` |

Agents save stable discovered facts via `save_memory` tool. Step numbers, current errors, and speculative conclusions are **never** saved to memory.

### 3.6 Metadata Schema — Every Data Record in Production

Every AI Search document, every Cosmos DB state record, every log line, every Service Bus message:

```
org_code : project : repo : file_path : symbol_name
```

- Set at ingestion/creation time by the pipeline
- Applied as a filter on every query
- The LLM never sees these values — injected by the pipeline via closure
- Even when starting with a single tenant, both `tenant_id` and `project_id` fields **must exist** (Invariant 2)

---

## 4. Branch Naming Convention

```
{branch_type}/issue-{issue_number}-{branch_title}-agent
```

Examples:
- `feat/issue-23-add-empty-cart-validation-agent`
- `fix/issue-25-improve-error-message-agent`

**The `-agent` suffix is a guardrail.** Only branches ending in `-agent` can be auto-merged by the pipeline. Human-created branches cannot be accidentally merged.

---

## 5. Business Rules and Guardrails

### 5.1 Retry Caps (Enforced in Code, Not Just Prompt)

| Failure type | Cap | What happens at cap |
|---|---|---|
| Coder test failures | 3 | Pipeline stops, state recorded as failed |
| Reviewer rejections | 2 | Pipeline stops, human notified |
| Human rejections | 2 | Pipeline stops, issue flagged |
| Agent iteration safety net | 30 | Hard-exit from agent loop |

**Why in code, not just prompt?** During development the LLM ignored the prompt instruction and infinite-looped. The Python counter cannot be bypassed by the LLM.

### 5.2 File-Write Guardrails (Layered Defense)

Before any file write is executed, five checks are applied in order:

1. **Path traversal** — `../../.env` is outside the repo root → deny
2. **Sensitive filename** — `.env`, `credentials.json`, `id_rsa`, `id_ed25519` → deny
3. **Sensitive extension** — `.pem`, `.key`, `.p12`, `.pfx` → deny
4. **Sensitive path word** — `secret`, `credential`, `private_key`, `token` in path → deny
5. **File must exist** — cannot create new files, only update existing ones → deny

### 5.3 Commit Message Convention

All commits must follow Conventional Commits format: `type: description`

Valid types: `feat`, `fix`, `hotfix`, `docs`, `test`, `chore`, `refactor`

Enforced deterministically in `commit_changes` tool — invalid format raises an error before git runs.

### 5.4 Protected Branch Rules

The pipeline checks before any write or push:
- Cannot target `main`, `master`, or the repo's configured `default_branch`
- Commit, push, checkout, and create_pr tools all reject protected-branch operations

### 5.5 Repo Validation

Every tool validates that the repo it is asked to touch exists in `project.yml github.repos`. Hallucinated or typo'd repo names are rejected before any git or GitHub operation.

### 5.6 PR Safety Rules

Before merging, `merge_pr` checks:
- PR must be open (not already merged or closed)
- PR must be in a mergeable state (no conflicts)
- Branch name must end in `-agent` (not a human's PR)

### 5.7 Test Command Allowlist

`run_tests` only accepts `pytest` with a specific set of flags. Shell injection via test arguments is blocked.

### 5.8 Board-Column Ordering

The orchestrator moves the GitHub Project board column **only at specific pipeline steps** — not speculatively, not out of order:

| Pipeline step | Board column |
|---|---|
| Issue created | Backlog |
| Analyst: feasible | Ready |
| Branch created | In Progress |
| PR created, Reviewer approved | In Review |
| Merged | Done (archived) |

---

## 6. The Agent Loop Pattern

Every agent uses the same explicit loop (not hidden inside a framework). This is intentional — it keeps every step visible and testable:

```
1. Build messages list: [SystemMessage(SYSTEM_PROMPT), HumanMessage(task)]
2. Call LLM with bound tools
3. If no tool calls → LLM is done → parse JSON from response → return
4. For each tool call:
   a. Execute the tool
   b. Record result (success or error) and duration in observability trace
   c. Append ToolMessage(result) to messages
5. Go to step 2
```

**Why manual loop instead of LangGraph or AgentExecutor?** This is a learning project — visibility is the goal. Every decision is inspectable. LangGraph migration is a planned future iteration.

---

## 7. Observability

The pipeline tracks token usage, cost, tool call durations, and total latency across all agents in a single trace object.

**GPT-4o-mini pricing baked in:**
- Input: $0.150 per 1M tokens
- Output: $0.600 per 1M tokens
- Full happy-path run: ~$0.002 (~10,000 tokens total)

The trace is created once by the orchestrator and passed to every agent. The final report shows the entire pipeline as a unified timeline.

---

## 8. GitHub as the API

**There is no pipeline-facing API.** All user interaction goes through GitHub:

| User action | GitHub event | Pipeline event |
|---|---|---|
| Submit requirement | Create issue with `sdlc-agent` label | Handler A triggered |
| Approve PR | Comment `/approve` on PR | Handler C triggered → merge |
| Reject PR | Comment `/reject <reason>` on PR | Handler C triggered → rework |

Any future front door (Slack bot, IDE plugin, web dashboard) is additive — it just needs to write the right GitHub event.

---

## 9. Two Front Doors (Production)

### Front Door A — Claude Code Skill

A markdown skill file at `~/.claude/skills/sdlc/SKILL.md`. Teaches Claude Code to translate natural language into GitHub REST API calls via `curl`.

- Requires: Claude Code installed, `GITHUB_TOKEN` env var with `repo` scope
- Does **not** require: `gh` CLI, MCP server, custom SDK, or any package install
- Auth: user's personal access token

### Front Door B — GitHub UI

User opens a browser, creates an issue with label `sdlc-agent`, and comments `/approve` or `/reject <reason>` on the PR when notified.

Both front doors converge on the same GitHub state — the same webhook fires, the same pipeline runs.

---

## 10. Auth Model

| Context | Auth mechanism |
|---|---|
| Claude Code skill (local user) | User's PAT via `GITHUB_TOKEN` env var |
| GitHub UI (local user) | Browser session |
| Azure pipeline (cloud) | GitHub App installation token (short-lived, per-installation) |
| Azure ↔ Azure services | Managed Identity (no static credentials anywhere) |
| Pipeline → Azure OpenAI | Managed Identity + private endpoint |

User-side auth and cloud-side auth are entirely separate systems.

---

## 11. Vector Store Architecture (Production)

Replaces local ChromaDB. The vector store is used as a **file locator**, not full RAG. The Analyst takes top-N semantic hits and reads those files properly.

**Two deployment tiers, one code path:**

| Tier | Default for | Isolation | Cost |
|---|---|---|---|
| **Shared** | Most orgs | Metadata filter per `org_code` | Lower — shared resources |
| **Private** | Compliance/large orgs | Dedicated Azure AI Search in `rg-sdlc-private-<org_code>` | Higher — dedicated resources |

Same metadata schema on every document. Pipeline resolves which endpoint to use via config lookup — no forked code.

**Known risks of shared tier:** noisy neighbour, GDPR erasure complexity, backup granularity, cost attribution, migration to private. These are documented trade-offs, not reasons to reject the design.

---

## 12. Invariants — Never Violate

### Invariant 1: State is JSON-serializable from day one

Every field in `SDLCState` and all pipeline state must be a plain dict, list, string, number, boolean, or None. No `datetime` objects (use ISO 8601 strings). No `set()`. No classes with methods.

Every handler must be able to: (a) load state from a JSON blob, (b) do work, (c) write state back as a JSON blob.

**Payoff:** Persisting state in Cosmos DB later is a one-day job, not a rewrite.

### Invariant 2: Every record has `tenant_id` and `project_id` from day one

Even if both start as `"default"`. Every AI Search document, every Cosmos state record, every log line, every event message carries these fields and every query filters on them.

**Payoff:** Adding the second tenant is trivial. Retrofitting multi-tenancy later is a rewrite.

---

## 13. Deferred Decisions (Do Not Pre-Design)

Do not invent answers for these. Build a configuration seam and leave the decision for when real requirements force it:

- Notification channel (email vs Teams vs both) and content format
- Who can approve (issue creator only vs any team member)
- Auto-close after N days without human response
- Exact retry counts, backoff, timeouts, circuit breakers
- Observability dashboards, alerts, SLOs
- CI/CD promotion strategy
- HA/DR RPO/RTO values
- Compliance certifications
- Cost governance, quotas, chargeback
- Full agent guardrails beyond the prototype set
- Cosmos DB partition key expression
- AI Search shared vs per-tenant index topology details

**Rule:** if you feel like designing one of these before the code forces the question, write code instead. Come back when the code makes the question concrete.
