# Multi-User Deployment Guide

This guide explains how to deploy the SAP CAP RAG application for individual users, each with their own HDI container and independent Cloud Foundry applications.

## Overview

Each user deployment uses the **CF MTA namespace feature** to create isolated deployments in a **shared CF space**. Multiple users can deploy simultaneously without affecting each other.

### How Namespaces Work

The `--namespace` flag during deployment automatically prefixes all app and service names with your username, creating complete isolation:

```bash
# Each user deploys with their username as namespace
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace jsmith
```

### CF Applications (auto-prefixed via namespace)
| Component | Pattern | Example |
|-----------|---------|---------|
| **Service App** | `<NAMESPACE>-genai-hana-rag-srv` | `jsmith-genai-hana-rag-srv` |
| **Web App** | `<NAMESPACE>-genai-hana-rag-app` | `jsmith-genai-hana-rag-app` |
| **DB Deployer** | `<NAMESPACE>-genai-hana-rag-db-deployer` | `jsmith-genai-hana-rag-db-deployer` |

### Service Instances (auto-prefixed via namespace)
| Component | Pattern | Example |
|-----------|---------|---------|
| **HDI Container** | `<NAMESPACE>-genai-hana-rag-db` | `jsmith-genai-hana-rag-db` |
| **AI Core** | Shared instance (not namespaced) | `ch-sbb-aicore` |

### URLs (custom routes in extension file)
| Component | Pattern |
|-----------|---------|
| **App URL** | `https://<USERNAME>-genai-hana-rag-app.cfapps.<REGION>.hana.ondemand.com` |
| **Service URL** | `https://<USERNAME>-genai-hana-rag-srv.cfapps.<REGION>.hana.ondemand.com` |

> **Important**: The namespace isolates your entire MTA deployment. Undeploying with `--namespace jsmith` will ONLY remove jsmith's apps, leaving other users' deployments intact.

All users share the same **Generative AI Hub service** instance (specified during configuration).

---

## Prerequisites

Before starting, ensure you have:

1. **SAP Business Application Studio (BAS)** access or local development environment
2. **Cloud Foundry CLI** installed and configured
3. **MBT (MTA Build Tool)** installed (`npm install -g mbt`)
4. **SAP BTP Account** with:
   - HANA Cloud instance (note down the Database ID)
   - SAP AI Core / Generative AI Hub service instance
   - Cloud Foundry space with deployment permissions

### Finding Your Configuration Values

| Value | Where to Find It |
|-------|------------------|
| **Database ID** | BTP Cockpit → HANA Cloud → Click your instance → "Manage Configuration" → Copy the Instance ID |
| **Region** | BTP Cockpit → Cloud Foundry Environment → API Endpoint shows region (e.g., `api.cf.eu10.hana.ondemand.com` → region is `eu10`) |
| **AI Core Service** | BTP Cockpit → Services → Instances → Find your AI Core service instance name |

---

## Deployment Options

Choose **ONE** of the following options:

| Option | Best For | Difficulty |
|--------|----------|------------|
| **Option A: Setup Script** | Quick deployment, first-time users | Easy |
| **Option B: Manual MTA Extension** | Full control, CI/CD pipelines | Medium |

---

## Option A: Using the Setup Script (Recommended)

### Step 1: Edit Configuration File

Open `user-config.json` and replace the placeholder values:

```json
{
  "USERNAME": "jsmith",
  "REGION": "eu10",
  "DATABASE_ID": "1159f744-6592-4c54-a96e-a6a924da3fbb",
  "AICORE_SERVICE_NAME": "my-genai-hub-service"
}
```

**Example configurations for different users:**

```json
// User: John Smith
{
  "USERNAME": "jsmith",
  "REGION": "eu10",
  "DATABASE_ID": "abc12345-1234-5678-9abc-def012345678",
  "AICORE_SERVICE_NAME": "shared-genai-hub"
}

// User: Jane Doe
{
  "USERNAME": "jdoe",
  "REGION": "eu10",
  "DATABASE_ID": "xyz98765-9876-5432-1abc-fedcba987654",
  "AICORE_SERVICE_NAME": "shared-genai-hub"
}
```

### Step 2: Run the Setup Script

**Option 2a: Using the config file**
```bash
./setup-deployment.sh --config
```

**Option 2b: Interactive mode (prompts for values)**
```bash
./setup-deployment.sh
```

The script will:
- Validate your inputs
- Generate `my-deployment.mtaext`
- Update `app/webapp/config.js` with your service URL

### Step 3: Login to Cloud Foundry

```bash
cf login -a https://api.cf.<REGION>.hana.ondemand.com
```

Example:
```bash
cf login -a https://api.cf.eu10.hana.ondemand.com
```

Select your org and space when prompted.

### Step 4: Build the Application

```bash
mbt build
```

This creates: `mta_archives/genai-hana-rag_1.0.0.mtar`

### Step 5: Deploy with Your Configuration

**IMPORTANT**: Always use the `--namespace` flag with your username to isolate your deployment:

```bash
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace <USERNAME>
```

Example for user "jsmith":
```bash
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace jsmith
```

### Step 6: Verify Deployment

```bash
# Check running apps (should see your username-prefixed app names)
cf apps

# Check services (should see your HDI container)
cf services

# View app logs (use CF app name, not route)
cf logs <USERNAME>-genai-hana-rag-srv --recent
```

---

## Option B: Manual MTA Extension File

### Step 1: Copy the Template

```bash
cp user-config.mtaext my-deployment.mtaext
```

### Step 2: Edit the Extension File

Open `my-deployment.mtaext` and replace ALL placeholders:

**Before (template with placeholders):**
```yaml
modules:
  - name: genai-hana-rag-srv
    parameters:
      app-name: <YOUR_USERNAME>-genai-hana-rag-srv
      host: <YOUR_USERNAME>-hana-rag-srv
      routes:
        - route: <YOUR_USERNAME>-hana-rag-srv.cfapps.<YOUR_REGION>.hana.ondemand.com

  - name: genai-hana-rag-db-deployer
    parameters:
      app-name: <YOUR_USERNAME>-genai-hana-rag-db-deployer

  - name: genai-hana-rag-app
    parameters:
      app-name: <YOUR_USERNAME>-genai-hana-rag-app
      host: <YOUR_USERNAME>-hana-rag
      routes:
        - route: <YOUR_USERNAME>-hana-rag.cfapps.<YOUR_REGION>.hana.ondemand.com

resources:
  - name: genai-hana-rag-db
    parameters:
      service-name: <YOUR_USERNAME>-hana-hdi-rag
      config:
        database_id: <YOUR_DATABASE_ID>

  - name: genai-hana-rag-aicore
    parameters:
      service-name: <YOUR_AICORE_SERVICE>
```

**After (Example for user "jsmith"):**
```yaml
modules:
  - name: genai-hana-rag-srv
    parameters:
      app-name: jsmith-genai-hana-rag-srv
      host: jsmith-hana-rag-srv
      routes:
        - route: jsmith-hana-rag-srv.cfapps.eu10.hana.ondemand.com

  - name: genai-hana-rag-db-deployer
    parameters:
      app-name: jsmith-genai-hana-rag-db-deployer

  - name: genai-hana-rag-app
    parameters:
      app-name: jsmith-genai-hana-rag-app
      host: jsmith-hana-rag
      routes:
        - route: jsmith-hana-rag.cfapps.eu10.hana.ondemand.com

resources:
  - name: genai-hana-rag-db
    parameters:
      service-name: jsmith-hana-hdi-rag
      config:
        database_id: 1159f744-6592-4c54-a96e-a6a924da3fbb

  - name: genai-hana-rag-aicore
    parameters:
      service-name: ch-sbb-aicore
```

### Step 3: Update Frontend Configuration

Edit `app/webapp/config.js`:

```javascript
window.RAG_CONFIG = {
    apiBaseUrl: "https://jsmith-hana-rag-srv.cfapps.eu10.hana.ondemand.com"
};
```

### Step 4: Build and Deploy

```bash
# Login to CF
cf login -a https://api.cf.eu10.hana.ondemand.com

# Build
mbt build

# Deploy with extension AND namespace
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace <YOUR_USERNAME>
```

---

## Complete Example: User "jsmith" in EU10

### Configuration Values
- Username: `jsmith`
- Region: `eu10`
- Database ID: `1159f744-6592-4c54-a96e-a6a924da3fbb`
- AI Core Service: `ch-sbb-aicore`

### Step-by-Step Commands

```bash
# 1. Navigate to project
cd /path/to/genai_hana_rag_user

# 2. Edit user-config.json with values above

# 3. Run setup script
./setup-deployment.sh --config

# 4. Login to Cloud Foundry
cf login -a https://api.cf.eu10.hana.ondemand.com

# 5. Build the application
mbt build

# 6. Deploy with namespace (IMPORTANT!)
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace jsmith

# 7. Verify
cf apps
cf mtas  # Shows your namespaced MTA
```

### Expected Output

```
name                              requested state   processes   routes
jsmith-genai-hana-rag-srv         started           web:1/1     jsmith-genai-hana-rag-srv.cfapps.eu10.hana.ondemand.com
jsmith-genai-hana-rag-app         started           web:1/1     jsmith-genai-hana-rag-app.cfapps.eu10.hana.ondemand.com
jsmith-genai-hana-rag-db-deployer stopped           web:0/1     (no route - deployer task)
```

### Access Your Application

- **Application UI**: `https://jsmith-hana-rag.cfapps.eu10.hana.ondemand.com`
- **API Endpoint**: `https://jsmith-hana-rag-srv.cfapps.eu10.hana.ondemand.com`

---

## Files Reference

| File | Purpose | When to Edit |
|------|---------|--------------|
| `user-config.json` | User configuration values | Before running setup script |
| `user-config.mtaext` | Template extension file | Copy and edit for manual option |
| `my-deployment.mtaext` | Generated extension file | Auto-generated by script |
| `app/webapp/config.js` | Frontend API URL | Auto-updated by script or manual |
| `mta.yaml` | Base MTA descriptor | Do NOT edit (use extension instead) |

---

## Troubleshooting

### Common Issues

**1. "Route already exists" error**
```bash
# Someone else is using this route. Choose a different username.
cf delete-route cfapps.eu10.hana.ondemand.com --hostname <conflicting-hostname>
```

**2. "Service instance not found" error**
```bash
# Verify AI Core service exists
cf services

# Check the service name matches your configuration
cf service <your-aicore-service-name>
```

**3. "Database not found" error**
- Verify your Database ID is correct in BTP Cockpit
- Ensure HANA Cloud instance is running
- Check you have access to the database

**4. Build fails with npm errors**
```bash
# Clear npm cache and rebuild
rm -rf node_modules
npm ci
mbt build
```

**5. Frontend shows "Network Error"**
- Check `app/webapp/config.js` has correct service URL
- Verify the `-srv` app is running: `cf apps`
- Check CORS settings in browser console

### Viewing Logs

```bash
# Recent logs (use the CF app name, not the route)
cf logs <USERNAME>-genai-hana-rag-srv --recent

# Stream logs in real-time
cf logs <USERNAME>-genai-hana-rag-srv
```

### Redeploying

```bash
# Rebuild and redeploy (always use --namespace)
mbt build
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace <USERNAME>
```

### Undeploying

```bash
# Remove YOUR deployment only (won't affect other users)
# Use --namespace with your username
cf undeploy genai-hana-rag --namespace <USERNAME> --delete-services --delete-service-keys

# Example for user "jsmith":
cf undeploy genai-hana-rag --namespace jsmith --delete-services --delete-service-keys
```

---

## Quick Reference Card

```bash
# === SETUP (one-time) ===
# Edit user-config.json with your values
./setup-deployment.sh --config

# === BUILD & DEPLOY ===
cf login -a https://api.cf.<REGION>.hana.ondemand.com
mbt build
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace <USERNAME>

# === VERIFY ===
cf apps
cf services
cf mtas  # Shows your namespaced MTA

# === ACCESS ===
# UI:  https://<USERNAME>-genai-hana-rag-app.cfapps.<REGION>.hana.ondemand.com
# API: https://<USERNAME>-genai-hana-rag-srv.cfapps.<REGION>.hana.ondemand.com

# === CLEANUP (only removes YOUR deployment) ===
cf undeploy genai-hana-rag --namespace <USERNAME> --delete-services --delete-service-keys
```

---

## Naming Convention Summary

### Namespace-Based Isolation

Deploy with: `cf deploy ... --namespace <USERNAME>`
Undeploy with: `cf undeploy genai-hana-rag --namespace <USERNAME> --delete-services`

### CF Applications (auto-prefixed via --namespace)

| Component | Naming Pattern | Example |
|-----------|---------------|---------|
| **Service App** | `<NAMESPACE>-genai-hana-rag-srv` | `jsmith-genai-hana-rag-srv` |
| **Web App** | `<NAMESPACE>-genai-hana-rag-app` | `jsmith-genai-hana-rag-app` |
| **DB Deployer** | `<NAMESPACE>-genai-hana-rag-db-deployer` | `jsmith-genai-hana-rag-db-deployer` |

### Service Instances (auto-prefixed via --namespace)

| Component | Naming Pattern | Example |
|-----------|---------------|---------|
| **HDI Container** | `<NAMESPACE>-genai-hana-rag-db` | `jsmith-genai-hana-rag-db` |
| **AI Core** | Shared service (not namespaced) | `ch-sbb-aicore` |

### URLs (custom routes in extension file)

| Component | Pattern | Example |
|-----------|---------|---------|
| **App URL** | `<USERNAME>-genai-hana-rag-app.cfapps.<REGION>.hana.ondemand.com` | `jsmith-genai-hana-rag-app.cfapps.eu10.hana.ondemand.com` |
| **Service URL** | `<USERNAME>-genai-hana-rag-srv.cfapps.<REGION>.hana.ondemand.com` | `jsmith-genai-hana-rag-srv.cfapps.eu10.hana.ondemand.com` |
