# Azure Deploy Skill — User Guide

## Overview

The `/azure-deploy` skill prepares and deploys any web application to an Azure Web App. It handles everything from detecting your framework and fixing compatibility issues, to creating the web app resource and deploying your code.

---

## Prerequisites

Before running the skill, ensure:

1. **Azure CLI installed** — [Install guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
2. **Azure account** — The skill will prompt you to log in if needed
3. **Existing App Service Plan** — The skill creates the Web App but expects an App Service Plan to already exist in the target resource group
4. **`zip` available** — Used to package the deployment artifact (`apt install zip` / `brew install zip`)

---

## Usage

```
/azure-deploy [app-name] [resource-group]
```

Both arguments are optional. If omitted, the skill derives them from your project:

| Argument | Default |
|---|---|
| `app-name` | Directory name of the project (sanitized) |
| `resource-group` | `<app-name>-rg` |

### Examples

```
/azure-deploy
/azure-deploy my-api
/azure-deploy my-api my-resource-group
```

---

## What the Skill Does

The skill runs through 7 phases automatically:

### Phase 1 — Pre-flight Checks
- Verifies the Azure CLI is installed
- Checks login status; prompts `az login` if needed
- If multiple subscriptions exist, asks you to confirm which one to target
- Resolves and sanitizes the app name and resource group

### Phase 2 — Framework Detection & Compatibility Analysis
Detects your framework from signal files (`package.json`, `requirements.txt`, `*.csproj`, `pom.xml`, `go.mod`, etc.) and audits the code for Azure compatibility issues.

**Fixes applied automatically:**

| Issue | Fix |
|---|---|
| Hardcoded port number | Patched to use `process.env.PORT` (Node.js) |
| Missing `start` script | Added to `package.json` |
| Missing WSGI server | `gunicorn` / `uvicorn` added to `requirements.txt` |
| Missing startup command | `startup.sh` created |
| Next.js static export | Switched to `standalone` output mode |
| Missing `engines.node` | Added matching local Node version |

### Phase 3 — Azure Configuration Files
Generates files needed by Azure's Oryx build system and Kudu:
- `startup.sh` — for Python and other interpreted runtimes
- `.deployignore` — excludes `node_modules/`, `.git/`, `.env` files, test directories
- `web.config` — only for Node.js on Windows App Service (rare)

### Phase 4 — Build
Installs dependencies and runs the build for your stack:
- **Node.js**: `npm ci` → `npm run build`
- **.NET**: `dotnet restore` → `dotnet publish -c Release`
- **Java**: `mvn package -DskipTests` / `gradle build -x test`
- **Python / Go / Ruby / PHP**: installs dependencies; no build step typically needed

Validates the output before proceeding.

### Phase 5 — Create Web App
Checks whether the Azure Web App already exists. If not, creates it:
```
az webapp create --name <app> --resource-group <rg> --plan <plan> --runtime <runtime>
```
The runtime string is selected automatically based on the detected framework and version. If the app name is already taken globally, a short random suffix is appended.

> The resource group and App Service Plan must already exist. Only the Web App itself is created here.

### Phase 6 — Deploy
Packages the app as a zip and deploys via `az webapp deploy`. Streams logs for 60 seconds after deployment to catch any startup errors.

Node.js `node_modules/` is excluded from the zip — Azure's Oryx build system installs them server-side when `SCM_DO_BUILD_DURING_DEPLOYMENT=true`.

### Phase 7 — Smoke Test & Summary
Polls the live URL (up to 5 retries, 10s apart) until it returns HTTP `200` or `302`. Prints a full summary:

```
App:            my-api
URL:            https://my-api.azurewebsites.net
Resource Group: my-resource-group
Framework:      Node.js 20 (Express)
Runtime:        NODE|20-lts

Changes made to the codebase:
  - package.json: added "start" script
  - server.js: patched port to use process.env.PORT

Next steps:
  - Set up a custom domain: az webapp config hostname add ...
  - Enable HTTPS-only:      az webapp update --https-only true ...
  - Set up CI/CD:           az webapp deployment source config ...
```

---

## Supported Frameworks

| Framework | Detected by | Azure Runtime |
|---|---|---|
| Node.js | `package.json` | `NODE\|20-lts` |
| Next.js | `package.json` + `next.config.*` | `NODE\|20-lts` |
| Vite SPA | `package.json` + `vite.config.*` | `NODE\|20-lts` (with Express server) |
| Python / Flask | `requirements.txt` + Flask import | `PYTHON\|3.12` |
| Python / Django | `requirements.txt` + `manage.py` | `PYTHON\|3.12` |
| Python / FastAPI | `requirements.txt` + FastAPI import | `PYTHON\|3.12` |
| .NET 6/8/9 | `*.csproj` | `DOTNETCORE\|8.0` |
| Java / Spring Boot | `pom.xml` (fat JAR) | `JAVA\|21-java21` |
| Java / Tomcat | `pom.xml` (WAR) | `TOMCAT\|10.1-java21` |
| Go | `go.mod` | `GO\|1.22` |
| Ruby | `Gemfile` | `RUBY\|3.3` |
| PHP | `composer.json` | `PHP\|8.3` |
| Docker | `Dockerfile` | Container deploy |

---

## Environment Variables & Secrets

The skill will ask if you have any environment variables (API keys, database connection strings, etc.) to configure. These are set securely via:

```bash
az webapp config appsettings set --name <app> --resource-group <rg> --settings KEY=VALUE
```

**Never store secrets in `.env` files in your repository.** The `.deployignore` generated by the skill explicitly excludes `.env*` files from deployment.

---

## Troubleshooting

| Symptom | Likely cause | Resolution |
|---|---|---|
| App returns `503` after deploy | App failed to start | Check logs: `az webapp log tail --name <app> --resource-group <rg>` |
| App returns `500` (Python) | Wrong WSGI path in startup command | Re-run the skill; it will re-detect and fix the startup command |
| Deployment zip too large | `node_modules` included | Ensure `SCM_DO_BUILD_DURING_DEPLOYMENT=true` is set and re-deploy |
| App name already taken | Azure names are globally unique | The skill auto-appends a random suffix |
| Cold starts are slow | Free/Basic tier has no Always On | Upgrade to P1v3 or enable Always On (B1+) |
| `az login` hangs | Non-interactive environment | Use `az login --use-device-code` or set up a service principal |

For a full troubleshooting reference see [framework-reference.md](framework-reference.md#troubleshooting).
