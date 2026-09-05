# Tenant Onboarding Guide

Steps a tenant must complete to use the SDLC platform.

---

## Prerequisites

- A GitHub organization (e.g. `sdlc-tenant`)
- Admin access to that organization

---

## Step 1: Install the SDLC GitHub App

Install `indentwork-sdlc` App on your GitHub organization:

```
https://github.com/apps/indentwork-sdlc/installations/new
```

**During installation:**
- Select "Only select repositories"
- Choose repos the platform should have access to:
  - All code repos you want indexed
  - All issue repos where you'll file SDLC requests

---

## Step 2: Create sdlc-config repo

Create a repository named `sdlc-config` in your org:

```bash
gh repo create sdlc-tenant/sdlc-config --public
```

This repo holds:
- `sdlc.yml` — declares projects, code repos, and issue repos
- `.github/workflows/upload-sdlc.yml` — uploads sdlc.yml to platform
- `.github/workflows/trigger-indexing.yml` — triggers indexing

Copy these files from the template:
```
https://github.com/IndentWork/sdlc-config-template
```

---

## Step 3: Register issue repos and code repos in sdlc.yml

Edit `sdlc.yml`:

```yaml
org: sdlc-tenant

projects:
  - name: ecommerce
    description: E-commerce platform
    issue_repo: https://github.com/sdlc-tenant/ecommerce-issues
    repos:
      - name: cart-service
        url: https://github.com/sdlc-tenant/cart-service
      - name: order-service
        url: https://github.com/sdlc-tenant/order-service

  - name: blog
    description: Blog platform
    issue_repo: https://github.com/sdlc-tenant/blog-issues
    repos:
      - name: blog-api
        url: https://github.com/sdlc-tenant/blog-api
```

Push to main — this triggers `upload-sdlc.yml` which uploads to the platform.

---

## Step 4: Trigger initial indexing

```
Actions → "Trigger Indexing" → Run workflow
```

The platform will clone your code repos and build the semantic index + code graph.

---

## Step 5: Set up each issue repo

**For each issue repo listed in sdlc.yml:**

### 5a. Add the trigger workflow

Add this file to the issue repo:

**Path:** `.github/workflows/sdlc-trigger.yml`

```yaml
name: Trigger SDLC Agent

on:
  issues:
    types: [labeled]

jobs:
  trigger:
    if: github.event.label.name == 'sdlc'
    runs-on: ubuntu-latest

    permissions:
      id-token: write
      contents: read

    steps:
      - uses: IndentWork/sdlc-actions/trigger-sdlc@main
        with:
          issue_number: ${{ github.event.issue.number }}
          issue_repo:   ${{ github.event.repository.name }}
          env:          dev
```

### 5b. Create the `sdlc` label

```bash
gh label create sdlc \
  --repo sdlc-tenant/ecommerce-issues \
  --description "Triggers SDLC AI agent" \
  --color 0075ca
```

Repeat for each issue repo.

---

## Step 6: Test end-to-end

1. Create an issue in any registered issue repo
2. Describe your requirement in the issue body
3. Assign the `sdlc` label
4. Watch the SDLC agent process it

The agent will:
- Analyze the requirement
- Find relevant files via semantic search
- Implement the change on a feature branch
- Open a PR
- Wait for your review via `/approve` or `/reject` comments

---

## What happens under the hood

```
Issue labeled 'sdlc' in ecommerce-issues
        ↓
sdlc-trigger.yml (in tenant repo) fires
        ↓
IndentWork/sdlc-actions/trigger-sdlc@main:
  - Gets OIDC token from GitHub
  - POST https://ca-sdlc-base-dev.../tenant/orchestrate
    with { issue_number, issue_repo }
        ↓
Control-plane verifies OIDC token → identifies tenant
        ↓
Publishes to Service Bus (action=orchestrate)
        ↓
sdlc-worker-orchestrator picks it up
        ↓
Analyst → Coder → Reviewer → Human HITL
```

---

## Frequently Asked Questions

**Q: Why do I need to add the workflow to every issue repo?**

Privacy. Only issues with the `sdlc` label trigger anything. Without the workflow file, no issue events leave your GitHub organization. You have hard control over what data reaches the SDLC platform.

**Q: Can I use a different label name?**

Yes — change `github.event.label.name == 'sdlc'` in the workflow file. But we recommend using `sdlc` consistently across all your issue repos.

**Q: What if I don't want to install the GitHub App on all repos?**

That's fine — only select the repos you want indexed and the issue repos where you'll file requests. The App only sees those.

**Q: How do I remove a repo from the platform?**

Remove it from `sdlc.yml` and push to main. The next indexing run will skip it.
