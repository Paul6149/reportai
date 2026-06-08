# ReportAI — Full Project Progress & State

_Last updated: 2026-05-26_

---

## 🌐 Live URL

**https://reportai-landing.nicecoast-929aa8e0.eastus.azurecontainerapps.io/**

- HTTP `200 OK` ✅
- `/health` endpoint `200 OK` ✅

---

## Project Files (C:\Users\paul6\Desktop\reportai)

| File | Purpose |
|---|---|
| `reportai-landing.html` | The landing page — this is what visitors see |
| `Dockerfile` | Packages the HTML into an `nginx:alpine` Linux container |
| `nginx.conf` | Custom nginx config: security headers, `/health` endpoint, correct MIME types |
| `PROGRESS.md` | This file |

---

## Everything That Has Been Done

### Session 1

#### ✅ Dockerfile & nginx config created
- `Dockerfile` uses `nginx:alpine`, copies `reportai-landing.html` as `index.html`, exposes port 80
- `nginx.conf` adds security headers (`X-Frame-Options`, `X-Content-Type-Options`, etc.) and a `/health` route returning `200 OK`

#### ✅ Docker image built locally
```
docker build -t reportai-app:latest .
```

#### ✅ Local preview confirmed working
```
docker run -d -p 8080:80 --name reportai-preview reportai-app:latest
# Visited http://localhost:8080 — landing page rendered correctly
# /health returned 200 OK
```
Container has since been stopped — safe to ignore.

#### ✅ Azure CLI installed
- Version: **2.86.0** (32-bit)
- Path: `C:\Program Files (x86)\Microsoft SDKs\Azure\CLI2`

#### ✅ Logged in to Azure
- Account: `paul6149@gmail.com`
- Subscription: **Azure subscription 1**
- Subscription ID: `46337230-6e04-4fe7-a995-cc7987420f2c`
- Tenant ID: `b1dbd588-f5b8-4683-a644-9e046c09cc01`

#### ✅ Resource group created
- Name: `reportai-rg` | Region: `eastus`

#### ❌ First ACR creation attempt — failed
- Error: `MissingSubscriptionRegistration` — `Microsoft.ContainerRegistry` provider had never been registered on this brand-new subscription (normal first-time issue)

---

### Session 2

#### ✅ Providers registered (one-time setup)
All three required namespaces registered:
```
Microsoft.ContainerRegistry   → Registered
Microsoft.App                 → Registered
Microsoft.OperationalInsights → Registered
```

#### ✅ Azure Container Registry created
- Name: `reportaiacr`
- Login server: `reportaiacr.azurecr.io`
- SKU: Basic | Admin: enabled

#### ✅ Image rebuilt for linux/amd64 and pushed to ACR
The first push (from Session 1's local build) used a Windows-format manifest which Azure rejected.
Fixed by using Docker Buildx to build and push directly for the correct platform:
```
docker buildx build --platform linux/amd64 \
  --tag reportaiacr.azurecr.io/reportai-app:latest \
  --push .
```
- Image digest: `sha256:c120bd4c0f3d6f84a689815162ffea94ffb44901c3a20b656be2f11d6ac5ace0`

#### ✅ Container Apps environment created
- Name: `reportai-env` | Region: `eastus`
- Auto-created Log Analytics workspace: `workspace-reportairgkB1u`

#### ✅ Container App deployed and verified live
```
az containerapp create \
  --name reportai-landing \
  --resource-group reportai-rg \
  --environment reportai-env \
  --image reportaiacr.azurecr.io/reportai-app:latest \
  --registry-server reportaiacr.azurecr.io \
  --target-port 80 \
  --ingress external \
  --min-replicas 1
```
- FQDN: `reportai-landing.nicecoast-929aa8e0.eastus.azurecontainerapps.io`
- Live check: HTTP 200, `/health` 200, 32,888 chars of HTML served

---

## Current Azure Resource Layout

```
Azure subscription 1  (46337230-6e04-4fe7-a995-cc7987420f2c)
└── reportai-rg  (eastus)
    ├── reportaiacr                   Azure Container Registry (Basic)
    │   └── reportai-app:latest       linux/amd64 nginx image ✅
    ├── reportai-env                  Container Apps Environment
    │   └── workspace-reportairgkB1u  Log Analytics (auto-created)
    └── reportai-landing              Container App ← LIVE ✅
        └── https://reportai-landing.nicecoast-929aa8e0.eastus.azurecontainerapps.io/
```

---

## Quick Reference

| Item | Value |
|---|---|
| **Live URL** | https://reportai-landing.nicecoast-929aa8e0.eastus.azurecontainerapps.io/ |
| Health endpoint | `<live-url>/health` → `200 OK` |
| Azure account | `paul6149@gmail.com` |
| Subscription ID | `46337230-6e04-4fe7-a995-cc7987420f2c` |
| Resource group | `reportai-rg` (eastus) |
| Container Registry | `reportaiacr.azurecr.io` |
| ACR image | `reportaiacr.azurecr.io/reportai-app:latest` |
| Container App name | `reportai-landing` |
| Container App env | `reportai-env` |

---

## Exact Next Steps

The core deployment is **complete** — the landing page is live. Below are the natural next steps in priority order.

---

### Option A — Update the landing page (most common task)

Edit `reportai-landing.html`, then run these two commands:

```powershell
# From C:\Users\paul6\Desktop\reportai

# 1. Rebuild and push the updated image
docker buildx build --platform linux/amd64 `
  --tag reportaiacr.azurecr.io/reportai-app:latest `
  --push .

# 2. Tell Azure to pull the new image (triggers a rolling restart)
az containerapp update `
  --name reportai-landing `
  --resource-group reportai-rg `
  --image reportaiacr.azurecr.io/reportai-app:latest
```

Changes are live in ~30 seconds.

---

### Option B — Add a custom domain & HTTPS certificate

```powershell
# Step 1: Add your domain to the Container App
az containerapp hostname add `
  --name reportai-landing `
  --resource-group reportai-rg `
  --hostname www.yourdomain.com

# Step 2: Get the verification token Azure gives you, add it as a DNS TXT record
az containerapp hostname list `
  --name reportai-landing `
  --resource-group reportai-rg `
  --output table

# Step 3: After DNS is verified, bind a managed TLS certificate (free)
az containerapp hostname bind `
  --name reportai-landing `
  --resource-group reportai-rg `
  --hostname www.yourdomain.com `
  --environment reportai-env `
  --validation-method CNAME
```

You will also need to add a **CNAME record** at your DNS provider:
```
www  →  reportai-landing.nicecoast-929aa8e0.eastus.azurecontainerapps.io
```

---

### Option C — Set up a CI/CD pipeline (auto-deploy on git push)

If you push the project to GitHub, Azure can auto-redeploy on every push:

```powershell
# From inside the reportai folder (must be a git repo first)
git init
git add .
git commit -m "Initial commit"
# Push to GitHub, then:

az containerapp github-action add `
  --name reportai-landing `
  --resource-group reportai-rg `
  --repo-url https://github.com/YOUR_USERNAME/reportai `
  --branch main `
  --registry-url reportaiacr.azurecr.io `
  --registry-username reportaiacr `
  --service-principal-client-id <SP_CLIENT_ID> `
  --service-principal-client-secret <SP_SECRET> `
  --service-principal-tenant-id b1dbd588-f5b8-4683-a644-9e046c09cc01
```

---

### Option D — Scale or reduce cost

Currently set to **min 1 replica** (always on, ~$0.40–0.80/month at idle).

To scale to zero when no traffic (free tier, cold-start ~2s):
```powershell
az containerapp update `
  --name reportai-landing `
  --resource-group reportai-rg `
  --min-replicas 0 `
  --max-replicas 3
```

---

### Option E — Monitor logs

```powershell
# Tail live logs from the running container
az containerapp logs show `
  --name reportai-landing `
  --resource-group reportai-rg `
  --follow

# View recent revision history
az containerapp revision list `
  --name reportai-landing `
  --resource-group reportai-rg `
  --output table
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Platform error on deploy | Always use `docker buildx build --platform linux/amd64` — never plain `docker build` for Azure |
| Provider not registered | `az provider register --namespace <name> --wait` then retry |
| ACR auth fails | Run `az acr login --name reportaiacr` to refresh Docker credentials |
| Container App not updating | Run `az containerapp update` after every new image push |
