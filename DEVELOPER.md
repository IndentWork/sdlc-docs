# Developer Guide — Deploy & Run SDLC Platform

Complete sequential guide to deploy the SDLC platform from scratch.

## Prerequisites

- Azure CLI logged in: `az login`
- GitHub CLI logged in: `gh auth login`
- Git cloned: all repos under `/home/ashish/project/sdlc-platform/`

---

## Step 1: Deploy Base Scope (Control Plane)

**Deploy PostgreSQL, FastAPI, ACR, VNet in base resource group:**

```bash
cd /home/ashish/project/sdlc-platform/sdlc-infra

# Plan
gh workflow run "SDLC Infra Terraform" \
  -f scope=base \
  -f env=DEV

# Wait for plan to complete in GitHub Actions
# Review the plan output

# Apply (go to GitHub Actions and approve)
# Or manually apply if you have local permissions:
# terraform apply -auto-approve tfplan
```

**Status:** ✅ Base resources created (PostgreSQL, FastAPI CA, ACR, VNet)

---

## Step 2: Deploy Shared Scope (Data Plane)

**Deploy Service Bus (topic), Storage, VNet peering to base:**

```bash
cd /home/ashish/project/sdlc-platform/sdlc-infra

gh workflow run "SDLC Infra Terraform" \
  -f scope=shared \
  -f env=DEV

# Wait for plan, review, approve apply
```

**Status:** ✅ Shared resources created (Service Bus topic, Storage, VNet peering)

---

## Step 3: Bootstrap ACR Access

**Grant shared MI AcrPull on base ACR (one-time per environment):**

```bash
cd /home/ashish/project/sdlc-platform/sdlc-infra

./scripts/bootstrap-acr-access.sh dev
```

**Status:** ✅ Worker can pull images from ACR

---

## Step 4: Bootstrap Service Bus Subscriptions

**Create tester subscription on sdlc-events topic (one-time per environment):**

```bash
cd /home/ashish/project/sdlc-platform/sdlc-infra

./scripts/bootstrap-subscriptions.sh dev
```

**Status:** ✅ Subscription ready, worker can connect

---

## Step 5: Deploy Control Plane

**Build and deploy FastAPI to base Container App:**

```bash
cd /home/ashish/project/sdlc-platform/sdlc-control-plane

# Trigger deploy pipeline
gh workflow run "Deploy Control Plane" \
  -f env=DEV

# Wait for build, push, and Container App update to complete
```

**Status:** ✅ FastAPI running, test endpoints ready

---

## Step 6: Deploy Worker Tester

**Build and deploy tester worker to base Container App environment:**

```bash
cd /home/ashish/project/sdlc-platform/sdlc-worker-tester

# Push to main triggers auto-deploy, or manually:
gh workflow run deploy.yml

# Wait for build, push, and Container App update
```

**Status:** ✅ Worker listening on tester subscription

---

## Step 7: Create Test Tenant

**Register a test tenant in the control plane:**

```bash
FQDN="ca-sdlc-base-dev.bluerock-3fcfcdd5.centralindia.azurecontainerapps.io"

# Delete old tenant if it exists
curl -X DELETE "https://$FQDN/tenants/9bb670ac-7840-4200-ae0a-14487273b7e3"

# Create fresh tenant
curl -X POST "https://$FQDN/tenants" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Tenant",
    "github_org": "sdlc-tenant",
    "tier": "shared"
  }'

# Note the returned tenant ID and resource_code
```

**Status:** ✅ Tenant created with resource_code (e.g., b310545b)

---

## Step 8: Install GitHub App on Tenant Org

**One-time setup — grant app access to tenant repos:**

```
https://github.com/apps/indentwork-sdlc/installations/new
→ Select sdlc-tenant org
→ Select specific repos (or all)
→ Install
```

**Status:** ✅ GitHub App installed, worker can clone repos

---

## Step 9: Test End-to-End Flow

**Trigger test workflow to validate full path: API → Service Bus → Worker → Storage:**

```bash
cd /home/ashish/project/sdlc-platform/sdlc-tenant/sdlc-config

# Manually trigger test
gh workflow run "Test SDLC Integration" \
  -f env=dev

# Wait for workflow to complete
```

**Expected result:** hello.txt appears in `stsdlcshareddev/configs/b310545b/hello.txt`

**Verify:**
```bash
az storage blob download \
  --account-name stsdlcshareddev \
  --container-name configs \
  --name "b310545b/hello.txt" \
  --auth-mode login \
  --file /tmp/hello.txt && cat /tmp/hello.txt
```

**Status:** ✅ End-to-end flow working

---

## Step 10: Test YAML Upload

**Upload sdlc.yml via tenant repo:**

```bash
cd /home/ashish/project/sdlc-platform/sdlc-tenant/sdlc-config

# Edit sdlc.yml with repo configuration
# Push to main

# Workflow auto-triggers on push, or manually:
gh workflow run "Upload SDLC Config" \
  -f env=dev

# Wait for upload to complete
```

**Expected result:** sdlc.yml appears in `stsdlcshareddev/configs/b310545b/sdlc.yml`

**Status:** ✅ Configuration upload working

---

## Troubleshooting

**Worker not connecting to subscription:**
```bash
# Check worker logs
az containerapp logs show \
  --name ca-sdlc-tester-dev \
  --resource-group rg-sdlc-base-dev \
  --tail 50
```

**Message not appearing in Storage:**
```bash
# Check Service Bus queue
az servicebus topic subscription show \
  --resource-group rg-sdlc-shared-dev \
  --namespace-name sb-sdlc-shared-dev \
  --topic-name sdlc-events \
  --name tester
```

**No ACR access:**
```bash
./scripts/bootstrap-acr-access.sh dev
```

---

## Environment Variables

Set these in Container App or locally for testing:

```bash
export ENV=dev
export SERVICEBUS_NAMESPACE=sb-sdlc-shared-dev.servicebus.windows.net
export AZURE_CLIENT_ID=<shared-mi-client-id>
```

---

## Summary

After following all 10 steps:
- ✅ Infrastructure deployed (base + shared scopes)
- ✅ FastAPI running with test endpoints
- ✅ Worker listening on Service Bus
- ✅ Test tenant created
- ✅ End-to-end flow validated
- ✅ Ready for real indexing work

**Typical deployment time: ~20 minutes**
