# Framework Reference for Azure Web App Deployment

## Detection Matrix

| File Present | Framework | Azure Runtime String | Linux? |
|---|---|---|---|
| `package.json` (no framework) | Node.js | `NODE|20-lts` | Yes |
| `package.json` + `next.config.*` | Next.js | `NODE|20-lts` | Yes |
| `package.json` + `vite.config.*` (SPA) | Vite SPA — **see note** | `NODE|20-lts` | Yes |
| `requirements.txt` + Flask | Python/Flask | `PYTHON|3.12` | Yes |
| `requirements.txt` + Django | Python/Django | `PYTHON|3.12` | Yes |
| `requirements.txt` + FastAPI | Python/FastAPI | `PYTHON|3.12` | Yes |
| `*.csproj` targeting `net8.0` | .NET 8 | `DOTNETCORE|8.0` | Yes |
| `*.csproj` targeting `net6.0` | .NET 6 | `DOTNETCORE|6.0` | Yes |
| `pom.xml` (Spring Boot fat JAR) | Java/Spring | `JAVA|21-java21` | Yes |
| `pom.xml` (WAR + Tomcat) | Java/Tomcat | `TOMCAT|10.1-java21` | Yes |
| `build.gradle` | Java/Gradle | `JAVA|21-java21` | Yes |
| `go.mod` | Go | `GO|1.22` | Yes |
| `Gemfile` | Ruby | `RUBY|3.3` | Yes |
| `composer.json` | PHP | `PHP|8.3` | Yes |
| `Dockerfile` | Docker | N/A — use container deploy | Yes |

> **Vite SPA note:** A pure client-side SPA has no server to run. Options:
> 1. Serve via a tiny Express static server (add `server.js` + update `start` script) — recommended for Azure Web App.
> 2. Deploy to Azure Static Web Apps instead (`az staticwebapp create`) — better fit for pure SPAs.
> Always ask the user which they prefer.

---

## Node.js Compatibility Checklist

### Required: PORT binding
Azure sets the `PORT` env var at runtime. The app must listen on it.

**Bad (hardcoded):**
```js
app.listen(3000)
```
**Good:**
```js
app.listen(process.env.PORT || 3000)
```

### Required: `start` script in `package.json`
Azure's Oryx build system runs `npm start` to launch the app.
```json
{
  "scripts": {
    "start": "node server.js"
  }
}
```

### Required for Next.js: standalone output
In `next.config.js` / `next.config.ts`:
```js
module.exports = {
  output: 'standalone',
}
```
Then set startup command to: `node .next/standalone/server.js`

### Recommended: `engines` field
```json
{
  "engines": {
    "node": ">=20.0.0"
  }
}
```

### Environment variables to set in Azure
| Setting | Value |
|---|---|
| `WEBSITE_NODE_DEFAULT_VERSION` | `~20` |
| `NODE_ENV` | `production` |
| `SCM_DO_BUILD_DURING_DEPLOYMENT` | `true` (Oryx runs npm install + build) |

---

## Python Compatibility Checklist

### Required: WSGI/ASGI entry point
Azure needs to know how to start the app.

**Flask** — `application.py` or `app.py`:
```python
app = Flask(__name__)  # must be named 'app' or export 'application'
```

**Django** — `wsgi.py` is auto-detected if `manage.py` is present.

**FastAPI:**
```python
app = FastAPI()
```

### Required: Startup command
Set via `az webapp config set --startup-file` or pass the command directly.

**Flask/Django (sync):**
```
gunicorn --bind=0.0.0.0 --timeout 600 app:app
```

**FastAPI/Starlette (async):**
```
uvicorn main:app --host 0.0.0.0 --port 8000
```

**Django with gunicorn:**
```
gunicorn --bind=0.0.0.0 --timeout 600 <project>.wsgi
```

### Required: `requirements.txt` includes server
Add if missing:
- Flask/Django: `gunicorn`
- FastAPI: `uvicorn[standard]` and `gunicorn`

### Environment variables to set in Azure
| Setting | Value |
|---|---|
| `SCM_DO_BUILD_DURING_DEPLOYMENT` | `true` |
| `PYTHON_ENABLE_GUNICORN_MULTIWORKERS` | `true` |
| `FLASK_ENV` / `DJANGO_SETTINGS_MODULE` | production values |

---

## .NET Compatibility Checklist

### Required: Linux publish settings
In `.csproj`:
```xml
<PropertyGroup>
  <RuntimeIdentifier>linux-x64</RuntimeIdentifier>
  <PublishReadyToRun>true</PublishReadyToRun>
  <SelfContained>false</SelfContained>
</PropertyGroup>
```

### Build & publish command
```bash
dotnet publish -c Release -r linux-x64 --self-contained false -o ./publish
```

### Environment variables to set in Azure
| Setting | Value |
|---|---|
| `ASPNETCORE_ENVIRONMENT` | `Production` |
| `ASPNETCORE_URLS` | `http://+:80` (Azure routes to port 80) |

---

## Java Compatibility Checklist

### Spring Boot fat JAR
- Ensure `spring-boot-maven-plugin` or `spring-boot-gradle-plugin` is configured to produce an executable JAR.
- Build: `mvn package -DskipTests` → produces `target/*.jar`
- Azure auto-detects the JAR if only one exists. If multiple, set startup command:
  ```
  java -jar /home/site/wwwroot/target/myapp.jar
  ```

### Port binding
Spring Boot respects `SERVER_PORT` env var — Azure sets this automatically. No code change needed.

### Environment variables to set in Azure
| Setting | Value |
|---|---|
| `JAVA_OPTS` | `-Xms512m -Xmx1024m` |
| `SPRING_PROFILES_ACTIVE` | `prod` |

---

## Go Compatibility Checklist

- Build for Linux: `GOOS=linux GOARCH=amd64 go build -o app .`
- App must listen on `os.Getenv("PORT")` or `os.Getenv("WEBSITES_PORT")`.
- Set startup command: `./app`

---

## PHP Compatibility Checklist

- Azure runs Apache + PHP by default on Linux.
- Ensure `index.php` or `public/index.php` exists (Laravel: `public/index.php` ✓).
- For Laravel, set startup command: `cp /home/site/wwwroot/.env.production /home/site/wwwroot/.env && php artisan migrate --force`.
- Add `COMPOSER_HOME=/home/site/.composer` to app settings.

---

## Windows App Service — IISNode `web.config` Template

Only generate this when targeting a Windows App Service Plan.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="iisnode" path="server.js" verb="*" modules="iisnode"/>
    </handlers>
    <rewrite>
      <rules>
        <rule name="NodeInspector" patternSyntax="ECMAScript" stopProcessing="true">
          <match url="^server.js\/debug[\/]?" />
        </rule>
        <rule name="StaticContent">
          <action type="Rewrite" url="public{REQUEST_URI}"/>
        </rule>
        <rule name="DynamicContent">
          <conditions>
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="True"/>
          </conditions>
          <action type="Rewrite" url="server.js"/>
        </rule>
      </rules>
    </rewrite>
    <security>
      <requestFiltering>
        <hiddenSegments>
          <remove segment="bin"/>
        </hiddenSegments>
      </requestFiltering>
    </security>
    <httpErrors existingResponse="PassThrough" />
    <iisnode watchedFiles="web.config;*.js"/>
  </system.webServer>
</configuration>
```

---

## Troubleshooting

### Error: `App name already exists`
Azure Web App names must be globally unique across all Azure customers.
```bash
# Append random suffix
APP_NAME="${APP_NAME}-$(cat /dev/urandom | tr -dc 'a-z0-9' | head -c6)"
```

### Error: `The subscription is not registered to use namespace 'Microsoft.Web'`
```bash
az provider register --namespace Microsoft.Web
# Wait ~60 seconds then retry
```

### Error: Python app returns 500 after deploy
1. Check startup command is correct: `az webapp config show --name $APP_NAME --resource-group $RESOURCE_GROUP --query linuxFxVersion`
2. Tail logs: `az webapp log tail --name $APP_NAME --resource-group $RESOURCE_GROUP`
3. Common cause: wrong WSGI module path. Fix the startup command.

### Error: Node.js app returns 503
1. Check `npm start` works locally.
2. Confirm `PORT` binding.
3. Check Oryx build logs: `az webapp log deployment show --name $APP_NAME --resource-group $RESOURCE_GROUP`

### Error: `.NET` app fails with `ANCM` error
1. Confirm `ASPNETCORE_URLS=http://+:80` is set.
2. Ensure publish was done for `linux-x64`.
3. Check that `web.config` (auto-generated by `dotnet publish`) is present in the publish output.

### Error: `az login` hangs in non-interactive environment
Use a service principal:
```bash
az login --service-principal \
  --username $AZURE_CLIENT_ID \
  --password $AZURE_CLIENT_SECRET \
  --tenant $AZURE_TENANT_ID
```
Tell the user to set these env vars or use `az login --use-device-code` for interactive browser auth.

### Deployment zip is too large (>2 GB)
For Node.js: exclude `node_modules/` from the zip and set `SCM_DO_BUILD_DURING_DEPLOYMENT=true` so Oryx installs them server-side.
For .NET: use `dotnet publish` output only (no source files).

### App works locally but fails on Azure (environment variables)
Local `.env` files are not deployed (correctly excluded). All env vars must be set via:
```bash
az webapp config appsettings set --name $APP_NAME --resource-group $RESOURCE_GROUP --settings KEY=VALUE
```

### Cold start is slow (B1/F1 SKU)
- Upgrade to `P1v3` or higher for production.
- Enable Always On: `az webapp config set --always-on true --name $APP_NAME --resource-group $RESOURCE_GROUP`
- Note: Always On is not available on Free (F1) tier.
