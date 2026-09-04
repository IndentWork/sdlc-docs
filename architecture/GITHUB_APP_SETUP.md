# GitHub App Setup

## App Details

| Field | Value |
|---|---|
| App Name | `indentwork-sdlc` |
| App ID | `4826692` |
| Client ID | `Iv23lizHfHRELtcPzDxE` |
| Owner | `IndentWork` org |
| Created | 2026-09-04 |

## Permissions Granted

| Permission | Level |
|---|---|
| Contents | Read & Write |
| Pull requests | Read & Write |
| Issues | Read & Write |
| Metadata | Read (mandatory) |

## Secrets Stored in Key Vault (`kv-sdlc-base-dev`)

| Secret Name | Value |
|---|---|
| `github-app-private-key` | `.pem` private key file |
| `github-app-id` | `4826692` |

## Key URLs

| Purpose | URL |
|---|---|
| App settings (IndentWork) | `https://github.com/organizations/IndentWork/settings/apps/indentwork-sdlc` |
| App installation page | `https://github.com/apps/indentwork-sdlc/installations/new` |
| sdlc-tenant installation settings | `https://github.com/organizations/sdlc-tenant/settings/installations/159023911` |

## Tenant Onboarding — GitHub App Steps

### One-time setup (tenant admin):
1. Go to `https://github.com/apps/indentwork-sdlc/installations/new`
2. Select their org
3. Under **Repository access** → select specific repos to expose to the platform
4. Click **Install**
5. Note the installation ID from the URL: `https://github.com/organizations/{org}/settings/installations/{installation_id}`
6. Store `installation_id` in the control plane DB for that tenant

### Adding a new repo later:
1. Go to `https://github.com/organizations/{org}/settings/installations`
2. Click **Configure** on `indentwork-sdlc`
3. Add the new repo under Repository access
4. Also add the repo to `sdlc-config/sdlc.yml` and push

### Revoking access:
- Remove repo from App installation settings — platform loses access instantly
- To fully offboard: uninstall the App from the org

## Security Model

- Installation is **org-level** — not tied to any individual user
- Tenant admin controls exactly which repos are exposed — no blanket access
- Tenant can revoke access at any time without involving IndentWork
- Platform can ONLY access repos the tenant has explicitly approved
- App uses **short-lived installation tokens** (1 hour) — no long-lived credentials

## Current Installations

| Tenant | Org | Installation ID | Repos Granted |
|---|---|---|---|
| SDLC Tenant | `sdlc-tenant` | `159023911` | Selected repos |
