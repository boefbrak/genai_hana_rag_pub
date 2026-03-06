# SAP BAS Deployment Guide
## Complete Step-by-Step Guide for SAP Business Application Studio

This guide walks you through importing this project into SAP Business Application Studio (BAS), configuring it with your username, and deploying to SAP BTP Cloud Foundry.

---

## 📋 Prerequisites

Before starting, ensure you have:

1. ✅ **SAP BAS access** - Access to SAP Business Application Studio
2. ✅ **SAP BTP Account** with:
   - HANA Cloud instance running
   - SAP AI Core / Generative AI Hub service instance created
   - Cloud Foundry space with deployment permissions
3. ✅ **GitHub Enterprise access** - Access to https://github.tools.sap

---

## 🚀 Part 1: Import Project into SAP BAS

### Step 1: Open SAP Business Application Studio

1. Navigate to your SAP BTP Cockpit
2. Go to **Services** → **Instances and Subscriptions**
3. Click on **SAP Business Application Studio** to open it
4. Create or open a **Full Stack Cloud Application** Dev Space
5. Wait for the Dev Space to start (status should be "RUNNING")

### Step 2: Clone the Repository

1. In SAP BAS, click on the **hamburger menu** (≡) → **Terminal** → **New Terminal**

2. Clone the repository:
```bash
git clone https://github.tools.sap/ICA-PnT-Academies/MB_genAI_hana_cap.git
```

3. Navigate to the project directory:
```bash
cd MB_genAI_hana_cap
```

4. Open the project in the workspace:
   - Click **File** → **Open Folder**
   - Select the `MB_genAI_hana_cap` folder
   - Click **OK**

### Step 3: Install Dependencies

In the terminal, run:
```bash
npm install
```

This will install all required Node.js packages.

---

## 🔧 Part 2: Configure Your Username and Settings

### Step 4: Gather Required Information

You need to collect the following information from your SAP BTP Cockpit:

| **Setting** | **Where to Find It** | **Example** |
|-------------|---------------------|-------------|
| **USERNAME** | Your choice (lowercase, alphanumeric) | `jsmith` or `tac007581u01` |
| **REGION** | BTP Cockpit → Cloud Foundry → Environment → API Endpoint | `eu10-004` |
| **DATABASE_ID** | BTP Cockpit → HANA Cloud → Instance → Manage Configuration | `1159f744-6592-4c54-a96e-a6a924da3fbb` |
| **AICORE_SERVICE_NAME** | BTP Cockpit → Services → Instances → AI Core instance name | `ch-sbb-aicore` |

#### How to find your Database ID:
1. Go to **BTP Cockpit** → **SAP HANA Cloud**
2. Click on your HANA Cloud instance
3. Click **"Manage Configuration"** or **"Actions"** → **"Manage Configuration"**
4. Copy the **Instance ID** (UUID format)

#### How to find your Region:
1. Go to **BTP Cockpit** → **Cloud Foundry** → **Subaccounts**
2. Click on your subaccount
3. Look at the **API Endpoint** (e.g., `https://api.cf.eu10-004.hana.ondemand.com`)
4. Your region is the part between `cf.` and `.hana` → `eu10-004`

#### How to find your AI Core Service Name:
1. Go to **BTP Cockpit** → **Services** → **Instances and Subscriptions**
2. Look for your **SAP AI Core** service instance
3. Copy the **Instance Name**

### Step 5: Edit the Configuration File

1. In SAP BAS, open the file `user-config.json` from the project root

2. Replace the placeholder values with YOUR information:

```json
{
  "// INSTRUCTIONS": "Edit values below, then run: ./setup-deployment.sh --deploy",

  "USERNAME": "your-username-here",
  "REGION": "your-region-here",
  "DATABASE_ID": "your-database-id-here",
  "AICORE_SERVICE_NAME": "your-aicore-service-name-here",

  "// NAMING_CONVENTION": {
    "CF_APP_SERVICE": "{USERNAME}-genai-hana-rag-srv",
    "CF_APP_WEBAPP": "{USERNAME}-genai-hana-rag-app",
    "CF_APP_DEPLOYER": "{USERNAME}-genai-hana-rag-db-deployer",
    "HDI_CONTAINER": "{USERNAME}-hana-hdi-rag",
    "AI_CORE": "ch-sbb-aicore (shared)",
    "SERVICE_URL": "https://{USERNAME}-genai-hana-rag-srv.cfapps.{REGION}.hana.ondemand.com",
    "APP_URL": "https://{USERNAME}-genai-hana-rag-app.cfapps.{REGION}.hana.ondemand.com"
  }
}
```

**Example configuration:**
```json
{
  "// INSTRUCTIONS": "Edit values below, then run: ./setup-deployment.sh --deploy",

  "USERNAME": "jsmith",
  "REGION": "eu10-004",
  "DATABASE_ID": "1159f744-6592-4c54-a96e-a6a924da3fbb",
  "AICORE_SERVICE_NAME": "ch-sbb-aicore"
}
```

3. **Save the file** (Ctrl+S or Cmd+S)

### Step 6: Run the Configuration Script

In the terminal, run:

```bash
chmod +x setup-deployment.sh
./setup-deployment.sh --config
```

This script will:
- ✅ Validate your configuration
- ✅ Generate `my-deployment.mtaext` with your settings
- ✅ Update `app/webapp/config.js` with your service URL
- ✅ Show a summary of your configuration

**Expected output:**
```
╔══════════════════════════════════════════════════════════════════╗
║          SAP CAP RAG - Multi-User Deployment                     ║
╚══════════════════════════════════════════════════════════════════╝

Reading from user-config.json...
Validating configuration...
Generating MTA extension file...
Created: /home/user/projects/MB_genAI_hana_cap/my-deployment.mtaext
Updating webapp configuration...
Updated: /home/user/projects/MB_genAI_hana_cap/app/webapp/config.js

╔══════════════════════════════════════════════════════════════════╗
║                    Configuration Complete!                       ║
╚══════════════════════════════════════════════════════════════════╝

Your Configuration:
  Username/Namespace:  jsmith
  Region:              eu10-004
  Database ID:         1159f744-6592-4c54-a96e-a6a924da3fbb
  AI Core Service:     ch-sbb-aicore (shared)
```

---

## ☁️ Part 3: Deploy to SAP BTP Cloud Foundry

### Step 7: Login to Cloud Foundry

In the SAP BAS terminal, login to Cloud Foundry:

```bash
cf login -a https://api.cf.<YOUR_REGION>.hana.ondemand.com
```

**Example:**
```bash
cf login -a https://api.cf.eu10-004.hana.ondemand.com
```

You will be prompted to:
1. **Enter your email** (your SAP email address)
2. **Enter your password** (your SAP password)
3. **Select your organization** (if you have multiple)
4. **Select your space** (the CF space where you want to deploy)

**Tip:** If you're already logged in through BAS, you can skip the password prompt.

### Step 8: Verify HANA Cloud is Running

Before deploying, ensure your HANA Cloud instance is running:

```bash
cf services
```

Look for your HANA Cloud instance in the list. If it shows "stopped", start it from the BTP Cockpit.

### Step 9: Build the Application

Build the MTA archive:

```bash
mbt build
```

**Expected output:**
```
[INFO] Building module 'genai-hana-rag-srv'...
[INFO] Building module 'genai-hana-rag-db-deployer'...
[INFO] Building module 'genai-hana-rag-app'...
[INFO] Generating metadata...
[INFO] Creating MTA archive...
[INFO] Build completed: mta_archives/genai-hana-rag_1.0.0.mtar
```

This creates the deployment artifact: `mta_archives/genai-hana-rag_1.0.0.mtar`

### Step 10: Deploy to Cloud Foundry

Deploy using the generated extension file and namespace:

```bash
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace <YOUR_USERNAME>
```

**Replace `<YOUR_USERNAME>` with the username you configured in Step 5.**

**Example:**
```bash
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace jsmith
```

⏰ **This will take 5-10 minutes.** The deployment process will:
- Create your HDI container (`<USERNAME>-hana-hdi-rag`)
- Deploy database artifacts
- Deploy the CAP service app (`<USERNAME>-genai-hana-rag-srv`)
- Deploy the web application (`<USERNAME>-genai-hana-rag-app`)

**Expected output:**
```
Deploying multi-target app archive mta_archives/genai-hana-rag_1.0.0.mtar...
Creating service "jsmith-hana-hdi-rag"...
Uploading application "jsmith-genai-hana-rag-db-deployer"...
Starting application "jsmith-genai-hana-rag-db-deployer"...
Uploading application "jsmith-genai-hana-rag-srv"...
Starting application "jsmith-genai-hana-rag-srv"...
Uploading application "jsmith-genai-hana-rag-app"...
Starting application "jsmith-genai-hana-rag-app"...
Process finished successfully.
```

### Step 11: Bind AI Core Service

After deployment, bind the shared AI Core service:

```bash
cf bind-service <YOUR_USERNAME>-genai-hana-rag-srv <YOUR_AICORE_SERVICE_NAME>
cf restage <YOUR_USERNAME>-genai-hana-rag-srv
```

**Example:**
```bash
cf bind-service jsmith-genai-hana-rag-srv ch-sbb-aicore
cf restage jsmith-genai-hana-rag-srv
```

⏰ **Restaging takes 2-3 minutes.**

---

## ✅ Part 4: Verify Deployment

### Step 12: Check Running Applications

```bash
cf apps
```

**Expected output:**
```
name                              requested state   processes           routes
jsmith-genai-hana-rag-srv         started           web:1/1             jsmith-genai-hana-rag-srv.cfapps.eu10-004.hana.ondemand.com
jsmith-genai-hana-rag-app         started           web:1/1             jsmith-genai-hana-rag-app.cfapps.eu10-004.hana.ondemand.com
jsmith-genai-hana-rag-db-deployer stopped           web:0/1             (no route)
```

### Step 13: Check Service Instances

```bash
cf services
```

**Expected output:**
```
name                       service         plan        bound apps
jsmith-hana-hdi-rag        hana           hdi-shared  jsmith-genai-hana-rag-srv, jsmith-genai-hana-rag-db-deployer
ch-sbb-aicore              aicore         standard    jsmith-genai-hana-rag-srv
```

### Step 14: Access Your Application

Open your web browser and navigate to:

```
https://<YOUR_USERNAME>-genai-hana-rag-app.cfapps.<YOUR_REGION>.hana.ondemand.com
```

**Example:**
```
https://jsmith-genai-hana-rag-app.cfapps.eu10-004.hana.ondemand.com
```

🎉 **Congratulations!** Your RAG application is now deployed and running!

---

## 🔄 Part 5: Alternative - Automated Deployment

Instead of Steps 7-11, you can use the automated deployment script:

```bash
./setup-deployment.sh --deploy
```

This single command will:
- ✅ Build the application
- ✅ Deploy to Cloud Foundry with namespace
- ✅ Bind AI Core service
- ✅ Restage the application

**Before running this, make sure:**
1. You've edited `user-config.json` (Step 5)
2. You're logged into Cloud Foundry (Step 7)

---

## 🛠️ Troubleshooting

### Issue 1: "Route already exists"
**Error:** `The route <username>-genai-hana-rag-app.cfapps.eu10-004.hana.ondemand.com is already in use.`

**Solution:** Choose a different username in `user-config.json` and re-run the configuration script.

### Issue 2: "Database not found"
**Error:** `Service hana-hdi-rag could not be created. Database with ID ... not found`

**Solution:**
1. Verify your Database ID in BTP Cockpit → HANA Cloud
2. Ensure HANA Cloud instance is running (not stopped)
3. Check you have the correct Database ID in `user-config.json`

### Issue 3: "AI Core service not found"
**Error:** `Service instance ch-sbb-aicore not found`

**Solution:**
1. Check the AI Core service name: `cf services`
2. Update `AICORE_SERVICE_NAME` in `user-config.json` with the correct name
3. Re-run `./setup-deployment.sh --config`

### Issue 4: Build fails
**Error:** `npm install failed` or `mbt build failed`

**Solution:**
```bash
# Clean and reinstall
rm -rf node_modules
rm -rf mta_archives
npm install
mbt build
```

### Issue 5: Application not starting
**Error:** App shows "crashed" status

**Solution:**
```bash
# Check logs for errors
cf logs <YOUR_USERNAME>-genai-hana-rag-srv --recent

# Common fixes:
# 1. Verify AI Core is bound
cf services | grep aicore

# 2. Restage the app
cf restage <YOUR_USERNAME>-genai-hana-rag-srv
```

### Viewing Logs

To view real-time logs:
```bash
# Service app logs
cf logs <YOUR_USERNAME>-genai-hana-rag-srv

# Recent logs
cf logs <YOUR_USERNAME>-genai-hana-rag-srv --recent

# DB deployer logs
cf logs <YOUR_USERNAME>-genai-hana-rag-db-deployer --recent
```

---

## 🔄 Redeployment (After Code Changes)

If you make changes to the code and want to redeploy:

```bash
# 1. Build the new version
mbt build

# 2. Deploy with your namespace
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace <YOUR_USERNAME>
```

**Quick command:**
```bash
mbt build && cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace jsmith
```

---

## 🗑️ Undeployment (Clean Up)

To completely remove your deployment:

```bash
cf undeploy genai-hana-rag --namespace <YOUR_USERNAME> --delete-services --delete-service-keys
```

**Example:**
```bash
cf undeploy genai-hana-rag --namespace jsmith --delete-services --delete-service-keys
```

⚠️ **Warning:** This will:
- Delete all your apps (`<username>-genai-hana-rag-*`)
- Delete your HDI container and database content
- Remove service bindings

**It will NOT affect:**
- Other users' deployments
- The shared AI Core service
- The HANA Cloud instance itself

---

## 📝 Quick Reference

### File Locations in Project
```
MB_genAI_hana_cap/
├── user-config.json              ← Edit this first
├── setup-deployment.sh           ← Run this to configure
├── my-deployment.mtaext          ← Generated by script
├── app/webapp/config.js          ← Updated by script
├── mta.yaml                      ← Don't edit (base config)
└── mta_archives/
    └── genai-hana-rag_1.0.0.mtar ← Created by build
```

### Essential Commands

```bash
# === CONFIGURATION ===
./setup-deployment.sh --config       # Generate config files

# === LOGIN ===
cf login -a https://api.cf.<REGION>.hana.ondemand.com

# === BUILD & DEPLOY ===
mbt build                            # Build the MTAR
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace <USERNAME>

# === BIND AI CORE ===
cf bind-service <USERNAME>-genai-hana-rag-srv <AICORE_SERVICE_NAME>
cf restage <USERNAME>-genai-hana-rag-srv

# === VERIFY ===
cf apps                              # Check running apps
cf services                          # Check services
cf logs <USERNAME>-genai-hana-rag-srv --recent

# === ACCESS ===
# Open: https://<USERNAME>-genai-hana-rag-app.cfapps.<REGION>.hana.ondemand.com

# === CLEANUP ===
cf undeploy genai-hana-rag --namespace <USERNAME> --delete-services --delete-service-keys
```

### Complete Deployment Script (Copy & Paste)

Replace the values and run all at once:

```bash
# === CONFIGURATION ===
USERNAME="jsmith"                    # ← CHANGE THIS
REGION="eu10-004"                    # ← CHANGE THIS
DATABASE_ID="your-db-id-here"        # ← CHANGE THIS
AICORE_SERVICE="ch-sbb-aicore"       # ← CHANGE THIS

# === DEPLOY ===
./setup-deployment.sh --config
cf login -a https://api.cf.${REGION}.hana.ondemand.com
mbt build
cf deploy mta_archives/genai-hana-rag_1.0.0.mtar -e my-deployment.mtaext --namespace ${USERNAME}
cf bind-service ${USERNAME}-genai-hana-rag-srv ${AICORE_SERVICE}
cf restage ${USERNAME}-genai-hana-rag-srv

# === VERIFY ===
cf apps
echo "App URL: https://${USERNAME}-genai-hana-rag-app.cfapps.${REGION}.hana.ondemand.com"
```

---

## 🎯 Summary Checklist

- [ ] SAP BAS Dev Space created and running
- [ ] Repository cloned from GitHub Enterprise
- [ ] Dependencies installed (`npm install`)
- [ ] `user-config.json` edited with your settings
- [ ] Configuration script run (`./setup-deployment.sh --config`)
- [ ] Logged into Cloud Foundry (`cf login`)
- [ ] Application built (`mbt build`)
- [ ] Deployed with namespace (`cf deploy ... --namespace <username>`)
- [ ] AI Core service bound (`cf bind-service` + `cf restage`)
- [ ] Verified deployment (`cf apps`, `cf services`)
- [ ] Accessed application URL in browser

---

## 💡 Tips

1. **Choose a unique username** - Use your initials or ID (e.g., `jsmith`, `tac007581u01`)
2. **Save your configuration** - Keep a copy of your `user-config.json` values
3. **Watch the logs** - Use `cf logs` to monitor application startup
4. **Check CF quotas** - Ensure your CF space has enough memory and service quotas
5. **HANA Cloud must be running** - Always verify HANA is started before deploying
6. **Use namespaces** - Always deploy with `--namespace <username>` for isolation

---

## 🆘 Getting Help

If you encounter issues not covered here:

1. Check the logs: `cf logs <USERNAME>-genai-hana-rag-srv --recent`
2. Verify services: `cf services`
3. Check app status: `cf apps`
4. Review the detailed deployment guide: `DEPLOYMENT_GUIDE.md`
5. Contact your SAP BTP administrator

---

**Last Updated:** 2026-02-11
**Version:** 1.0.0
