# PLD Deployment Guide

Complete guide for deploying the Product Landscape Designer (PLD) to SAP BTP Cloud Foundry — covering local setup, MTA build, `cf push` manifest deployment, and post-deployment verification.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Structure](#project-structure)
3. [Environment Setup](#environment-setup)
4. [Deployment Methods](#deployment-methods)
   - [Method 1: CF Push with Manifest (Recommended for individual apps)](#method-1-cf-push-with-manifest)
   - [Method 2: MTA Build and Deploy (Full stack)](#method-2-mta-build-and-deploy)
5. [Service Bindings](#service-bindings)
6. [Post-Deployment Verification](#post-deployment-verification)
7. [Rollback](#rollback)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Tools Required

```bash
# Cloud Foundry CLI
brew install cloudfoundry/tap/cf-cli@8

# MTA plugin for CF CLI
cf install-plugin multiapps

# MTA Build Tool
npm install -g mbt

# Verify installations
cf --version
cf plugins | grep multiapps
mbt --version
```

### CF Login

```bash
# Production
cf login -a https://api.cf.<region>.hana.ondemand.com --sso
# Get passcode at: https://login.cf.<region>.hana.ondemand.com/passcode
# Org:   <your-prod-org>
# Space: Production

# Canary
cf login -a https://api.cf.<region>.hana.ondemand.com --sso
# Get passcode at: https://login.cf.<region>.hana.ondemand.com/passcode
# Org:   <your-canary-org>
# Space: Canary
```

---

## Project Structure

```
pld/
├── mta.yaml                  # MTA descriptor — full stack deployment
├── manifest.yml              # Template for cf push manifest (has userId placeholders)
├── app-manifest-template.yaml # Template used by manifest-update.js
├── app-manifest.yaml         # Generated manifest (gitignored) — actual cf push target
├── Deployment.mk             # Makefile for deployment commands
├── manifest-update.js        # Script to generate app-manifest.yaml from template
├── package-update.js         # Pre-build script run before MTA build
├── db-manifest.yaml          # Manifest for HANA HDI container deployment
├── mtaext/                   # MTA extension files per environment
├── ui/                       # Approuter (pld-ui)
├── ui3/                      # Model UI (pld-model-ui)
├── blueprints/               # Blueprints microservice
├── provisioning/             # Provisioning microservice
├── scenarios/                # Scenarios microservice
├── templates/                # Templates microservice
├── utilities/                # Utilities microservice
├── layouter/                 # Python layouter service
└── db/                       # HANA HDI database module
```

---

## Environment Setup

### 1. Create `.env` file (project root)

```bash
# Copy from a colleague or Passvault — never commit this file
cat > .env << 'EOF'
CF_ORG=<your-prod-org>
CF_SPACE=Production
APP_PREFIX=pld
BACKEND_URL=https://<your-backend-url>
DASHBOARD_URL=https://<your-dashboard-url>
TCO_REPORTING_URL=https://<your-tco-reporting-url>
EOF
```

### 2. Generate app-manifest.yaml

The `manifest.yml` has `userId` placeholders. Run the update script to generate the actual manifest:

```bash
node --env-file=.env manifest-update.js
# Generates: app-manifest.yaml
```

---

## Deployment Methods

### Method 1: CF Push with Manifest

Use this for deploying individual backend apps or the UI without a full MTA rebuild. Fastest option for hotfixes.

#### Deploy all backend apps

```bash
# Build all backend modules first
make -f Deployment.mk deploy_apps
# This runs npm run build for each module, then cf push -f app-manifest.yaml
```

#### Deploy a single app

```bash
# Build the module
cd utilities && npm run build && cd ..

# Push only that app
cf push pld-utilities-green -f app-manifest.yaml --no-start
cf start pld-utilities-green
```

#### Deploy the UI (approuter)

```bash
cf push pld-ui-green -f app-manifest.yaml
```

#### Deploy the database

```bash
make -f Deployment.mk deploy_db
# Runs: cf push -f db-manifest.yaml
```

---

### Method 2: MTA Build and Deploy

Use this for full stack releases — creates a versioned `.mtar` archive and deploys everything atomically.

#### Step 1: Build the MTA archive

```bash
# From the project root
node package-update.js    # Update package versions if needed
mbt build -p cf           # Builds the .mtar file into mta_archives/
```

The build output will be at:
```
mta_archives/PLD_<version>.mtar
```

#### Step 2: Deploy to CF

```bash
# Deploy to Production
cf deploy mta_archives/PLD_<version>.mtar \
  --strategy rolling \
  -e mtaext/production.mtaext

# Deploy to Canary
cf deploy mta_archives/PLD_<version>.mtar \
  --strategy rolling \
  -e mtaext/canary.mtaext
```

**Deployment flags:**

| Flag | Description |
|---|---|
| `--strategy rolling` | Zero-downtime rolling update |
| `--strategy blue-green` | Blue-green with manual switch |
| `-e <file>.mtaext` | Environment-specific extension (overrides mta.yaml values) |
| `--no-start` | Deploy but don't start apps (useful for staged rollouts) |
| `-m <module>` | Deploy only specific MTA module |

#### Step 3: Monitor deployment

```bash
cf dmol   # List recent deployments (requires multiapps plugin)
cf deploy-service-logs <operation-id>  # Stream logs for a specific deploy
```

---

## Service Bindings

Service bindings inject credentials into apps via `VCAP_SERVICES`. PLD uses these services:

| Service Name | CF Service | Plan | Used By |
|---|---|---|---|
| `pld-uaa` | `xsuaa` | `application` | All apps |
| `pld-external-uaa` | `xsuaa` | `application` | Backend apps |
| `pld-internal-uaa` | `xsuaa` | `application` | Backend apps |
| `connectivity-service` | `connectivity` | `lite` | Approuter, backend apps |
| `destination-service` | `destination` | `lite` | Approuter, backend apps |
| `pld-hdi` | `hana` | `hdi-shared` | Backend apps, db |
| `audit-logs-service` | `auditlog` | `standard` | Backend apps |
| `pld-cloud-logs` | `cloud-logging` | `standard` | Backend apps |
| `application-autoscaler` | `autoscaler` | `lite` | Backend apps |

### Creating service instances (first-time setup)

```bash
cf create-service xsuaa application pld-uaa -c xs-security.json
cf create-service xsuaa application pld-external-uaa -c xs-security-external.json
cf create-service xsuaa application pld-internal-uaa -c xs-security-internal.json
cf create-service connectivity lite connectivity-service
cf create-service destination lite destination-service
cf create-service hana hdi-shared pld-hdi
cf create-service auditlog standard audit-logs-service
cf create-service cloud-logging standard pld-cloud-logs
cf create-service autoscaler lite application-autoscaler
```

### Binding an app manually

```bash
# Standard bind (BTP picks credential type from service instance config)
cf bind-service <app-name> <service-name>

# Bind with explicit X.509 and custom validity
cf bind-service <app-name> connectivity-service \
  -c '{"xsuaa":{"credential-type":"x509","x509":{"key-length":2048,"validity":365,"validity-type":"DAYS"}}}'

# After any bind change, restage is required (not just restart)
cf restage <app-name>
```

### Viewing current bindings

```bash
cf services                     # List all service instances and bound apps
cf env <app-name>               # Show VCAP_SERVICES for an app
```

> **Important:** All PLD service bindings use `credential-type: x509` (configured in `mta.yaml`). These certificates expire — see the [X.509 Certificate Renewal runbook](./service-bindings-x509.md).

---

## Post-Deployment Verification

```bash
# Check app is running
cf app pld-ui-green
cf app pld-utilities-green

# Tail live logs
cf logs pld-ui-green

# Check recent logs for errors
cf logs pld-ui-green --recent | grep -i error

# Verify no cert errors (main failure mode)
cf logs pld-ui-green --recent | grep -E "certificate expired|fetchClientCredentialsToken"

# Quick smoke test — check the app URL responds
curl -s -o /dev/null -w "%{http_code}" https://<your-app-url>/
```

---

## Rollback

### Rolling back a cf push

CF keeps the previous droplet. To roll back to it:

```bash
# List recent deployments / app revisions
cf app pld-utilities-green --guid | xargs -I{} cf curl /v3/apps/{}/revisions

# Roll back to previous revision
cf rollback pld-utilities-green --revision <revision-number>
```

### Rolling back an MTA deploy

```bash
# List MTA operations
cf mta-ops

# Roll back the last deploy
cf undeploy PLD --delete-services false
```

---

## Troubleshooting

### App crashes on start

```bash
cf logs <app-name> --recent   # Check startup errors
cf events <app-name>          # Check recent CF events
cf app <app-name>             # Check memory/disk limits
```

### 500 errors on all proxied routes

Almost always an expired X.509 certificate on connectivity or destination service binding.
→ Follow the [X.509 Certificate Renewal runbook](./service-bindings-x509.md).

### App stuck in "starting" state

```bash
cf app <app-name>
# If health-check-type is http, verify the health endpoint returns 200
# If health-check-type is process, the main process must stay running
```

### MTA build fails

```bash
# Check node/npm version matches engines in package.json
node --version
npm --version

# Clear caches and retry
rm -rf node_modules mta_archives
npm install
mbt build -p cf
```

### Service instance creation fails

```bash
cf service <service-name>      # Check if it already exists
cf service-brokers             # Verify the service broker is available in this space
cf marketplace -e <service>    # Check available plans
```
