---
name: azure-deploy
description: Analyze, prepare, and deploy a web application to Azure Web App. Detects framework, fixes compatibility issues, configures Azure resources, and deploys end-to-end. Use when deploying any web app to Azure.
argument-hint: "[app-name] [resource-group]"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Azure Web App Deployment Skill

Deploy the web application to Azure Web App following these steps in order. Arguments (all optional — you will derive sensible defaults if not provided):
- `$0`: App name (default: derived from project directory name)
- `$1`: Resource group name (default: `<app-name>-rg`)

---

## PHASE 1 — Pre-flight Checks

### 1.1  Azure CLI availability
Run `az version` to confirm the CLI is installed. If it fails, stop and tell the user to install the Azure CLI from https://learn.microsoft.com/en-us/cli/azure/install-azure-cli and re-run the skill.

### 1.2  Azure login
Run `az account show` to verify the user is logged in.
- If not logged in, run `az login` and wait for confirmation.
- After login, list subscriptions with `az account list --output table` and ask the user to confirm which subscription to target if there is more than one. Then run `az account set --subscription "<id>"`.

### 1.3  Resolve deployment parameters
Determine the following values — prefer explicit `$ARGUMENTS`, fall back to the defaults described above:
- `APP_NAME`
- `RESOURCE_GROUP`

Sanitize `APP_NAME`: lowercase, hyphens only, max 60 chars (Azure naming rules). Tell the user the resolved values before proceeding.

---

## PHASE 2 — Framework Detection & Compatibility Analysis

### 2.1  Detect framework
Read the project root. Inspect files in this priority order to identify the stack. See [framework-reference.md](framework-reference.md) for the full detection matrix.

| Signal file | Framework |
|---|---|
| `package.json` | Node.js (Express, Next.js, Vite, React, etc.) |
| `requirements.txt` / `pyproject.toml` / `Pipfile` | Python (Flask, Django, FastAPI, etc.) |
| `*.csproj` / `*.sln` | .NET |
| `pom.xml` / `build.gradle` | Java (Spring Boot, Quarkus, etc.) |
| `go.mod` | Go |
| `Gemfile` | Ruby |
| `composer.json` | PHP |
| `Dockerfile` (any of the above missing) | Docker/Container |

Read the detected config file fully to understand:
- Entry point / start command
- Build command
- Output directory
- Port binding
- Environment variable expectations

### 2.2  Detect existing Azure config
Check for any existing Azure-related files:
- `azure.yaml`, `.azure/`, `Bicep/*.bicep`, `infra/`, `arm/`
- `web.config`, `.deployment`, `startup.sh`, `startup.cmd`
- `Dockerfile` / `.dockerignore`

If found, read them and incorporate their settings rather than overwriting.

### 2.3  Compatibility audit
Run the framework-specific checks from [framework-reference.md](framework-reference.md). Common issues to detect and fix:

**Node.js**
- Is `"start"` script defined in `package.json`? If missing, add one.
- Does the app bind to `process.env.PORT`? Azure sets PORT dynamically. If hardcoded, patch it.
- Is there a `.nvmrc` or `engines.node` field? If missing, add `engines.node` matching the local node version.
- For Next.js: ensure `output` is not set to `export` (static export is not compatible with Azure Web App's Node runtime — switch to standalone or warn user).

**Python**
- Is there a WSGI/ASGI entry point? Detect `app`, `application`, or `create_app` in the main module.
- Is `gunicorn` (sync) or `uvicorn` (async/FastAPI) listed in requirements? Add if missing.
- Is there a `startup.sh` or `startup.txt`? Create if missing with the correct gunicorn/uvicorn command.

**.NET**
- Ensure the project targets a runtime compatible with Azure (`net6.0`, `net7.0`, `net8.0`, `net9.0`).
- Confirm `<PublishReadyToRun>` and `<RuntimeIdentifier>linux-x64</RuntimeIdentifier>` for Linux App Service.

**Java**
- Confirm the JAR is executable and `JAVA_OPTS` is configurable via environment.
- Add a `startup.sh` if not present.

**Go / Ruby / PHP**
- Check for necessary startup scripts and add as needed.

**All frameworks**
- Check `.gitignore` / `.deployignore` — ensure build artifacts and secrets are not committed but output folders are not excluded from deployment.

### 2.4  Apply fixes
For each issue found, make the minimal required code change and explain what was changed and why. Do not refactor unrelated code.

---

## PHASE 3 — Azure Configuration Files

### 3.1  Generate `.deployment` (if not present)
```ini
[config]
command = deploy.sh
```
Only add if a custom build step is needed. Skip for simple runtimes.

### 3.2  Generate startup command file (if needed)
For Python, create `startup.sh`:
```bash
#!/bin/bash
gunicorn --bind=0.0.0.0 --timeout 600 <module>:<app>
```
For FastAPI/async apps use `uvicorn <module>:<app> --host 0.0.0.0 --port $PORT`.

### 3.3  Generate `web.config` (Node.js on Windows App Service only)
Only generate if the user explicitly targets a Windows plan. Use the IISNode template from [framework-reference.md](framework-reference.md).

### 3.4  Add `.azure-ignore` / `.deployignore`
Create `.deployignore` (Kudu / Oryx respects this) to exclude:
- `node_modules/` (Azure rebuilds on server)
- `.git/`
- Test files and directories
- Local `.env` files (never deploy secrets in files)

Preserve any existing entries.

---

## PHASE 4 — Build

### 4.1  Install dependencies locally (for validation)
Run the appropriate install command:
- Node.js: `npm ci` or `yarn install --frozen-lockfile` or `pnpm install --frozen-lockfile`
- Python: `pip install -r requirements.txt --quiet`
- .NET: `dotnet restore`
- Java: `mvn dependency:resolve -q` or `gradle dependencies -q`

### 4.2  Run the build
- Node.js: run `npm run build` if a `build` script exists
- .NET: `dotnet publish -c Release -o ./publish`
- Java: `mvn package -DskipTests` or `gradle build -x test`
- Python / Go / Ruby / PHP: typically no build step

### 4.3  Validate build output
Confirm the expected output directory / artifact exists and is non-empty. Stop and report if the build failed.

---

## PHASE 5 — Create Web App

### 5.1  Create the Web App (if not exists)
Check whether the web app already exists:
```bash
az webapp show --name "$APP_NAME" --resource-group "$RESOURCE_GROUP" --query name --output tsv 2>/dev/null
```
If it already exists, skip to Phase 6.

Choose the `--runtime` string based on the framework detected in Phase 2. See [framework-reference.md](framework-reference.md) for the full runtime string map. Examples:
- Node.js 20: `NODE|20-lts`
- Python 3.12: `PYTHON|3.12`
- .NET 8: `DOTNETCORE|8.0`
- Java 21 (Tomcat): `TOMCAT|10.1-java21`

```bash
az webapp create \
  --name "$APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --plan "${APP_NAME}-plan" \
  --runtime "<RUNTIME_STRING>"
```

If the app name is already taken globally, append a short random suffix (`$(cat /dev/urandom | tr -dc 'a-z0-9' | head -c6)`) and update `APP_NAME`.

---

## PHASE 6 — Deploy

### 6.1  Choose deployment method
Use **zip deploy** as the default (fast, reliable, works for all frameworks):
```bash
# Create deployment zip (exclude .git, node_modules for Node.js, etc.)
zip -r deploy.zip . \
  --exclude "*.git*" \
  --exclude "*node_modules*" \
  --exclude "*.env*" \
  --exclude "*__pycache__*" \
  --exclude "*.pytest_cache*"

az webapp deploy \
  --name "$APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --src-path deploy.zip \
  --type zip
```

For .NET, deploy the `./publish` folder instead:
```bash
cd publish && zip -r ../deploy.zip . && cd ..
```

### 6.2  Monitor deployment
```bash
az webapp log tail \
  --name "$APP_NAME" \
  --resource-group "$RESOURCE_GROUP"
```
Stream logs for up to 60 seconds. Look for startup errors. If the app crashes on startup, read the error and return to Phase 2 to fix the root cause.

---

## PHASE 7 — Smoke Test & Summary

### 7.1  Get the app URL
```bash
az webapp show \
  --name "$APP_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --query defaultHostName \
  --output tsv
```

### 7.2  Smoke test
Run `curl -s -o /dev/null -w "%{http_code}" https://<defaultHostName>` — retry up to 5 times with 10-second waits (apps take ~30s to warm up). A `200` or `302` is success. Any `5xx` means the app failed to start — tail logs and diagnose.

### 7.3  Summary report
Print a deployment summary:
```
✅ Deployment complete!

App:            <APP_NAME>
URL:            https://<defaultHostName>
Resource Group: <RESOURCE_GROUP>
Framework:      <detected framework and version>
Runtime:        <Azure runtime string>

Changes made to the codebase:
  - <list each file modified and why>

Next steps:
  - Set up a custom domain: az webapp config hostname add ...
  - Enable HTTPS-only:      az webapp update --https-only true ...
  - Set up CI/CD:           az webapp deployment source config ...
```

---

## Error Handling

If any step fails:
1. Print the exact error message and the command that failed.
2. Diagnose the root cause (do not retry blindly).
3. Apply the minimal fix.
4. Resume from the failed step — do not restart from the beginning.

Common errors and fixes are documented in [framework-reference.md](framework-reference.md#troubleshooting).
