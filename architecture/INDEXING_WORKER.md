# sdlc-worker-indexing — Architecture & Build Plan

## Purpose

Indexes tenant repos so agents can semantically search code and understand relationships.
Triggered when a tenant pushes `sdlc.yml` via the `upload-sdlc` GitHub Action.

---

## Trigger Flow

```
Tenant pushes sdlc.yml
        ↓
upload-sdlc action → POST /tenant/upload_sdlc (OIDC auth)
        ↓
FastAPI → Service Bus topic (sdlc-events)
  action = upload_sdlc
  resource_code = b310545b
  tier = shared
        ↓
sdlc-worker-indexing listens on "indexing" subscription
  filter: action = 'upload_sdlc' OR action = 'index_repo'
        ↓
Indexing pipeline runs
```

---

## Full Indexing Sequence

```
1. READ sdlc.yml
   Load from Storage: configs/{resource_code}/sdlc.yml
   Parse YAML → list of repos to index

2. GET GITHUB TOKEN
   Read App ID from env (GITHUB_APP_ID)
   Read private key from Key Vault (github-app-private-key)
   GET /app/installations/{installation_id}/access_tokens
   → short-lived token (1 hour)

3. FOR EACH REPO — parallel via asyncio.gather()

   3a. FETCH FILE TREE
       GET /repos/{org}/{repo}/git/trees/main?recursive=1
       Filter: .py files only
       Each file has: path, sha

   3b. LOAD CHECKPOINT
       Storage: checkpoints/{resource_code}/{repo}/_project.json
       Contains: { files: [{path, sha}] }
       If not found → full crawl (first time)

   3c. CRAWL CHANGED FILES
       For each .py file:
         Compare GitHub SHA vs checkpoint SHA
           Same SHA → skip (nothing changed)
           Different or missing:
             → GET /repos/{org}/{repo}/contents/{path} → file content
             → crawl_file(content) → structured dict
             → attach SHA to result
             → save to Storage: checkpoints/{resource_code}/{repo}/{file}.json

   3d. CHUNK → Azure AI Search
       For each crawled file result:
         build_function_text() → upsert to AI Search
         build_class_text()    → upsert to AI Search
         build_method_text()   → upsert to AI Search
         Delete stale chunks (functions/classes removed from file)

   3e. LOAD GRAPH → Cosmos DB
       Pass 1 — create/update nodes (File, Function, Class, Method)
       Pass 2 — create/update relationships (CALLS, IMPORTS, HAS_METHOD)
       Delete stale nodes

   3f. UPDATE CHECKPOINT
       Save updated _project.json with new SHAs to Storage

4. LOG COMPLETION
   Metrics: repos indexed, files crawled, chunks upserted, nodes created
```

---

## Naming Convention

All records in AI Search and Cosmos use the same key format regardless of tier:

```
{resource_code}:{github_org}:{project}:{repo}:{file}:{symbol}
```

**Examples:**
```
b310545b:sdlc-tenant:ecommerce:cart-service:cart.py:add_to_cart
b310545b:sdlc-tenant:ecommerce:cart-service:cart.py:CartService:checkout
```

**Why resource_code not tier:**
- `resource_code = SHA256(github_org)[:8]` — never changes regardless of tier
- Tenant migrates shared → dedicated: infrastructure changes, data keys unchanged
- Zero migration cost on data

---

## Metadata Schema

Every record in AI Search and Cosmos carries this metadata:

```json
{
    "chunk_id":      "b310545b:sdlc-tenant:ecommerce:cart-service:cart.py:add_to_cart",
    "resource_code": "b310545b",
    "github_org":    "sdlc-tenant",
    "tier":          "shared",
    "project":       "ecommerce",
    "repo":          "cart-service",
    "file":          "cart.py",
    "type":          "function|class|method",
    "name":          "add_to_cart",
    "sha":           "a1b2c3d4..."
}
```

`tier` is informational only — never part of the key.

---

## Checkpoint Storage Structure

```
Azure Blob Storage (stsdlcshareddev/checkpoints):

checkpoints/
  {resource_code}/
    {github_org}/
      {project}/
        {repo}/
          _project.json        ← index: list of files + SHAs + crawled_at
          cart.py.json         ← crawl output for cart.py
          models.py.json       ← crawl output for models.py
          order_manager.py.json
```

Example:
```
checkpoints/b310545b/sdlc-tenant/ecommerce/cart-service/_project.json
checkpoints/b310545b/sdlc-tenant/ecommerce/cart-service/cart.py.json
checkpoints/b310545b/sdlc-tenant/ecommerce/cart-service/models.py.json
```

**`_project.json` format:**
```json
{
  "resource_code": "b310545b",
  "github_org":    "sdlc-tenant",
  "project":       "ecommerce",
  "repo":          "cart-service",
  "crawled_at":    "2026-09-05T10:00:00Z",
  "files": [
    {"path": "cart.py",    "sha": "a1b2c3d4"},
    {"path": "models.py",  "sha": "e5f6g7h8"}
  ]
}
```

---

## Repository Structure

```
sdlc-worker-indexing/
  app/
    main.py                        ← Service Bus listener, creates "indexing" subscription
    indexing/
      orchestrator.py              ← reads sdlc.yml, runs repos in parallel
      chunker.py                   ← adapted from python_chunker.py (AI Search)
      graph_loader.py              ← adapted from neo4j_loader.py (Cosmos DB)
      crawlers/
        base.py                    ← BaseCrawler abstract class (interface all crawlers follow)
        registry.py                ← maps file extensions to crawlers
        python_crawler.py          ← .py files (wraps python_ast.py from prototype)
        python_ast.py              ← copied from prototype, zero changes
        markdown_crawler.py        ← .md files (future — add when needed)
    services/
      github.py                    ← get token, file tree, file content via GitHub API
      storage.py                   ← load/save checkpoints from Azure Blob Storage
      keyvault.py                  ← read GitHub App private key at startup
      ai_search.py                 ← upsert/delete chunks in Azure AI Search
      cosmos.py                    ← upsert/delete nodes and relationships in Cosmos DB
  Dockerfile
  pyproject.toml
  uv.lock
  .github/
    workflows/
      deploy.yml
```

---

## Crawler Plugin Architecture

Each file type has its own crawler class. Adding a new crawler = add one file + register it.

```
BaseCrawler (abstract)
  ├── PythonCrawler    → .py  (active)
  ├── MarkdownCrawler  → .md  (future)
  ├── JavaCrawler      → .java (future)
  └── GoLangCrawler    → .go  (future)
```

**Registry routes by extension:**
```python
# registry.py
CRAWLERS = [
    PythonCrawler(),
    # MarkdownCrawler(),  ← uncomment when ready
]

def get_crawler(file_path: str) -> BaseCrawler | None:
    ext = Path(file_path).suffix.lower()
    return next((c for c in CRAWLERS if ext in c.supported_extensions()), None)
```

**Orchestrator is extension-agnostic:**
```python
for file in repo_files:
    crawler = get_crawler(file.path)
    if crawler is None:
        continue  # unsupported extension — skip silently
    result = await crawler.crawl(content, file.path, file.sha)
```

**Adding Markdown support tomorrow:**
1. Create `crawlers/markdown_crawler.py`
2. Add `MarkdownCrawler()` to `CRAWLERS` in `registry.py`
3. Done — zero changes to orchestrator or any other file

---

## Code Reuse from Prototype

| Prototype file | Enterprise file | Change |
|---|---|---|
| `crawlers/python_ast.py` | `indexing/crawler.py` | `crawl_file(path)` → `crawl_file(content)` — one line |
| `chunkers/python_chunker.py` | `indexing/chunker.py` | Swap ChromaDB → Azure AI Search |
| `neo4j_loader/load_project.py` | `indexing/graph_loader.py` | Swap Neo4j → Cosmos DB |
| Local disk read | `services/github.py` | GitHub API |
| Local disk write | `services/storage.py` | Azure Blob Storage |

---

## Service Bus Subscription

Worker creates its own subscription on startup:

```python
TOPIC_NAME        = "sdlc-events"
SUBSCRIPTION_NAME = "indexing"
SQL_FILTER        = "action = 'upload_sdlc' OR action = 'index_repo'"
```

No Terraform changes needed.

---

## Environment Variables

```bash
SERVICEBUS_NAMESPACE   = sb-sdlc-shared-dev.servicebus.windows.net
AZURE_CLIENT_ID        = <shared-mi-client-id>
GITHUB_APP_ID          = 4826692
KEY_VAULT_URL          = https://kv-sdlc-base-dev.vault.azure.net
ENV                    = dev
```

---

## Infrastructure Required

Already provisioned:

| Resource | Status |
|---|---|
| Service Bus topic (sdlc-events) | ✅ exists |
| Storage (checkpoints container) | ✅ exists |
| Shared Managed Identity | ✅ exists |
| Key Vault (github-app-private-key) | ✅ exists |

Still needed:

| Resource | Where |
|---|---|
| Azure AI Search | New Terraform module |
| Cosmos DB | New Terraform module |
| indexing subscription on Service Bus | bootstrap-subscriptions.sh |

---

## Build Order

1. **`sdlc-worker-indexing` repo** — create repo, Dockerfile, pyproject.toml
2. **`indexing/crawlers/base.py`** — BaseCrawler abstract interface
3. **`indexing/crawlers/python_ast.py`** — copy from prototype, zero changes
4. **`indexing/crawlers/python_crawler.py`** — wraps python_ast, one-line change
5. **`indexing/crawlers/registry.py`** — maps extensions to crawlers
6. **`services/keyvault.py`** — read GitHub App private key at startup
7. **`services/github.py`** — GitHub App token + file tree + file content
8. **`services/storage.py`** — checkpoint load/save
9. **`services/ai_search.py`** — upsert/delete chunks (add AI Search Terraform later)
10. **`services/cosmos.py`** — upsert/delete nodes/relationships (add Cosmos Terraform later)
11. **`indexing/chunker.py`** — adapt for AI Search
12. **`indexing/graph_loader.py`** — adapt for Cosmos DB
13. **`indexing/orchestrator.py`** — parallel repo coordinator
14. **`app/main.py`** — Service Bus listener
15. **Terraform** — AI Search + Cosmos DB modules
16. **Deploy pipeline** — same pattern as sdlc-worker-tester

---

## Key Design Decisions

| Decision | Reason |
|---|---|
| `resource_code` as tenant key (not tier) | Zero migration cost shared → dedicated |
| Crawler plugin architecture (BaseCrawler + registry) | Add new file types without touching orchestrator |
| One JSON per Python file in Storage | Mirrors prototype, enables incremental updates |
| GitHub SHA for change detection | More reliable than timestamp across environments |
| `asyncio.gather()` for parallel repos | All repos in sdlc.yml indexed simultaneously |
| 2-pass graph loading (nodes then relationships) | Cross-file CALLS/IMPORTS need all nodes to exist first |
| Worker creates its own subscription | No Terraform changes when adding new workers |
