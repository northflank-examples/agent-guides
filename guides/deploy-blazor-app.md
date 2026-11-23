<!--
agent-guide-meta:
  id: NF-STK-BLAZOR
  description: "How to deploy a Blazor app in Docker: interactive server rendering over SignalR WebSockets on port 8080, persisted data protection keys or WebAssembly on nginx."
  sources:
    - https://learn.microsoft.com/en-us/aspnet/core/blazor/host-and-deploy/server/
    - https://learn.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/overview
    - https://learn.microsoft.com/en-us/aspnet/core/blazor/host-and-deploy/webassembly/
    - https://learn.microsoft.com/en-us/dotnet/core/compatibility/containers/8.0/aspnet-port
    - https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks
    - https://hub.docker.com/_/nginx
    - https://github.com/nginx/nginx/blob/master/conf/mime.types
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/load-balancing
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-STK-DOTNET, NF-DB-001, NF-STK-REACT, NF-DEP-001]
-->

# How to deploy a Blazor app

To deploy a Blazor app with interactive server rendering, publish it into the ASP.NET Core runtime image, listen on port 8080 with `ASPNETCORE_HTTP_PORTS`, serve it behind a proxy that passes WebSockets through for the SignalR circuit, and persist the data protection keys outside the container. Without persisted keys, every redeploy invalidates antiforgery tokens and login cookies. A WebAssembly-only app is static files and needs only a web server.

This guide does it on Northflank with the CLI: a Blazor Web App built from Git into `mcr.microsoft.com/dotnet/aspnet`, keys on a small persistent volume, one instance so circuits stay put, and a health endpoint. You end with a public HTTPS URL where `/healthz` returns `Healthy` and SignalR offers WebSockets; an nginx variant covers standalone WebAssembly. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human links your Git account once.

## TL;DR: how to deploy a Blazor app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a Blazor app and its render mode, and confirm the team, repository, branch and project before changing anything. Give Northflank access to the repository (once).
3. In `Program.cs`, persist data protection keys to `/keys`, add `/healthz` and remove `UseHttpsRedirection`. Add a multi-stage `Dockerfile` (`dotnet/sdk:10.0` builds, `dotnet/aspnet:10.0` runs on port 8080) and push.
4. Create the project and the service with `northflank put service combined`: HTTP port 8080, one instance, probes on `/healthz`.
5. Attach a 1 GB volume at `/keys` with `northflank create volume`, and wait for the rollout.
6. Check that the live build is your pushed commit, `/healthz` answers, SignalR offers WebSockets and an interactive page works in a browser.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why WebSockets, one instance and persisted keys?

Microsoft's server-side Blazor hosting docs say UI updates, events and JavaScript calls travel over a SignalR connection, and that Blazor "works best when using WebSockets" as the transport, falling back to Long Polling otherwise. Each connected user holds a **circuit** in server memory (about 250 KB for a minimal app, per the same docs), so every request for that circuit must reach the same server. On every platform Microsoft documents (App Service, Container Apps, IIS, Kubernetes, nginx), the requirements are the same: WebSockets on, session affinity on with more than one instance, and data protection keys stored where every instance and every restart can read them.

On Northflank, a **combined service** builds your `Dockerfile` from Git and serves it on HTTPS; public HTTP ports pass WebSockets through. A small **volume** holds the keys. A volume is `ReadWriteOnce`, which also keeps the service at one instance, the simplest way to keep circuits on one server.

Where to host a Blazor app depends on the render mode. Interactive Server (and Auto) needs a long-running ASP.NET Core process: Azure App Service, IIS on Windows, a Linux VM behind nginx, or a container platform. Standalone Blazor WebAssembly is static files and runs on any static host or CDN, or in a tiny nginx container (step 10).

## Prerequisites

- **A Blazor app that builds locally** (run from the folder with the server `.csproj`):
  ```bash
  dotnet publish -c Release -o /tmp/blazor-publish && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Check the app and confirm where it will deploy

Before changing anything, confirm that this guide fits the app and print the deployment target. Run this from the repository root:

```bash
grep -lE 'Sdk="Microsoft\.NET\.Sdk\.(Web|BlazorWebAssembly)"' $(find . -name '*.csproj' -not -path '*/bin/*' -not -path '*/obj/*')
grep -hoE '<TargetFramework>[^<]+' $(find . -name '*.csproj' -not -path '*/bin/*' -not -path '*/obj/*') | sort -u
grep -rn --include=Program.cs -E 'AddInteractive(Server|WebAssembly)Components|WebAssemblyHostBuilder' .
ls Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId blazor-app -o json >/dev/null 2>&1 \
  && echo "project: blazor-app exists and will be reused" \
  || echo "project: blazor-app will be created"
```

- If no `.csproj` uses `Microsoft.NET.Sdk.Web` or `Microsoft.NET.Sdk.BlazorWebAssembly`, or the last grep prints nothing, this isn't a Blazor app. Stop and use a guide for your framework.
- The render mode decides the path. `AddInteractiveServerComponents` (Server or Auto): steps 5–9. Only `WebAssemblyHostBuilder` in a standalone `blazorwasm` project with no server: step 10 instead of steps 5, 6 and 8.
- Match the image tags in step 6 to the target framework (`net10.0` → `10.0`, `net8.0` → `8.0`), and note the server project's assembly name (usually the `.csproj` file name) for the `ENTRYPOINT`.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Region matters more than usual here: Microsoft recommends a sustained UI latency of 250 ms or less, and every interaction in Interactive Server is a network round trip. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "blazor-app", "region": "europe-west" }'
export PROJECT_ID=blazor-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

Names used below: project `blazor-app`, service `blazor`, volume `dp-keys`.

## Step 5: Configure `Program.cs`

Add these lines to the template's `Program.cs`:

```csharp
using Microsoft.AspNetCore.DataProtection;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents();

// Keys survive restarts and redeploys when /keys is a persistent volume
builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo(builder.Configuration["DP_KEYS_PATH"] ?? "/keys"))
    .SetApplicationName("blazor-app");

builder.Services.AddHealthChecks();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error", createScopeForErrors: true);
}
// app.UseHttpsRedirection();   // removed: TLS ends at the load balancer

app.UseAntiforgery();
app.MapStaticAssets();
app.MapHealthChecks("/healthz");
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode();

app.Run();
```

- **`PersistKeysToFileSystem`** is the Microsoft-documented way to store keys in a folder. Their Docker guidance says keys should live in "a Docker volume that persists beyond the container's lifetime". Without it, the image keeps keys in the container's home directory and logs a warning that they "may not be persisted outside of the container".
- **`SetApplicationName`** keeps the key isolation name stable across deployments; the docs say the value "should match across deployments of the app".
- **`UseHttpsRedirection`** can't find an HTTPS port inside the container. Remove it. If `Request.Scheme` must read `https`, add `app.UseForwardedHeaders()` for `XForwardedProto`; it hasn't been confirmed that the load balancer sends that header.
- Keep `AddInteractiveWebAssemblyComponents()` if the app uses Auto or WebAssembly components; the rest stays the same.
- `MapStaticAssets()` is the .NET 9+ template's static file call; a .NET 8 app keeps `app.UseStaticFiles()` instead.

## Step 6: Write a multi-stage `Dockerfile`

Put it next to the server `.csproj`:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app .
ENV ASPNETCORE_HTTP_PORTS=8080
EXPOSE 8080
USER $APP_UID
ENTRYPOINT ["dotnet", "<ASSEMBLY>.dll"]
```

Since .NET 8, the official images listen on 8080; setting `ASPNETCORE_HTTP_PORTS` makes it explicit. `APP_UID` selects the image's non-root `app` user. For a solution with a separate `.Client` project (Auto/WebAssembly), copy both `.csproj` files before `dotnet restore`, or restore after `COPY . .`. Add a `.dockerignore` with `bin/`, `obj/` and `.git`.

Northflank builds what's on the remote branch, not your local working tree, so commit and push these files before creating the service:

```bash
git add Dockerfile .dockerignore Program.cs
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Create the service

Write `blazor.json`. Circuits live in server memory, so it starts at `nf-compute-50` (1 GB):

```json
{
  "name": "blazor",
  "billing": { "deploymentPlan": "nf-compute-50" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "ASPNETCORE_HTTP_PORTS": "8080", "ASPNETCORE_ENVIRONMENT": "Production", "DP_KEYS_PATH": "/keys" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/healthz", "port": 8080,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/healthz", "port": 8080,
      "initialDelaySeconds": 20, "periodSeconds": 20, "timeoutSeconds": 3, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file blazor.json
```

Use the same repository URL, provider and branch as in step 4. `billing.buildPlan` is optional; leave it out to use the default build plan. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined`. The first build starts automatically; if none appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId blazor --noDefaults -o json --input '{}'`.

## Step 8: Attach the key volume

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId dp-keys --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "dp-keys",
  "mounts": [ { "containerMountPath": "/keys" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 1024 },
  "attachedObjects": [ { "id": "blazor", "type": "service" } ]
}'
```

`storageSize` is in MB and can only be increased; keys are a few KB each. Attaching redeploys the service. With a `ReadWriteOnce` volume the service runs one instance, and on each deploy the old container stops before the new one starts, so connected users see Blazor's reconnect UI for a few seconds.

**Scaling out later.** To run several instances, store keys somewhere shared instead of a volume, for example `PersistKeysToDbContext` (package `Microsoft.AspNetCore.DataProtection.EntityFrameworkCore`) with the PostgreSQL setup from [how to deploy an ASP.NET Core web application](deploy-aspnet-core-web-application.md), and add `"loadBalancing": { "mode": "consistentHash", "consistentHash": { "mode": "ip" } }` to the service. Whether IP-based routing keeps circuits intact across reconnects hasn't been confirmed, so test it first.

## Step 9: Wait for the rollout and get the URL

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId blazor --noDefaults -o json)
  B=$(echo "$S" | jq -r '(.data // .).status.build.status // "NONE"')
  D=$(echo "$S" | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "build=$B deployment=$D"
  case "$B" in FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo BUILD_FAILED; break;; esac
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  [ "$B" = "SUCCESS" ] && [ "$D" = "COMPLETED" ] && { OK=1; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId blazor --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId blazor --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. Run the loop again after step 8's redeploy. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId blazor`. The troubleshooting table covers the common causes.

## Step 10 (WebAssembly only): Serve static files with nginx

A standalone Blazor WebAssembly app publishes to `wwwroot` and runs entirely in the browser. Skip steps 5, 6 and 8 and use this `Dockerfile`:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /out

FROM nginx:1.29-alpine
COPY --from=build /out/wwwroot /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 8080
```

And `nginx.conf`:

```nginx
server {
  listen 8080;
  root /usr/share/nginx/html;
  location = /healthz { return 200 "ok\n"; }
  location / { try_files $uri $uri/ /index.html; }
}
```

The image's `mime.types` (loaded in the `http` block) already maps `.wasm` to `application/wasm`. `try_files` sends deep links such as `/counter` to `index.html` so the Blazor router handles them. Commit and push `Dockerfile` and `nginx.conf` (`git add Dockerfile nginx.conf .dockerignore`, then commit and `git push`), then create the service as in step 7 with `nf-compute-10`, the same port 8080, probes on `/healthz`, no `runtimeEnvironment` and no volume, and wait for it with the step 9 loop. The [Vite React guide](deploy-vite-react-app.md) uses the same nginx pattern.

## How to know it worked

You're done when these hold (Interactive Server):

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId blazor --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health endpoint answers:
   ```bash
   curl -s "https://$HOST/healthz"     # Healthy
   ```
3. SignalR offers WebSockets on the Blazor hub:
   ```bash
   curl -s -X POST "https://$HOST/_blazor/negotiate?negotiateVersion=1" | jq -r '.availableTransports[].transport'
   ```
   The list includes `WebSockets`.
4. Keys are on the volume, not in the container. The log warning about keys that "may not be persisted outside of the container" is absent, and the folder has a key file. `exec` drops output without a TTY, so wrap it in `script` (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId blazor --lineLimit 300 --textIncludes "persisted outside"
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId blazor \
     --shell-cmd 'sh -c' --cmd '"ls /keys | grep -c key-"'
   ```
   The first command prints nothing; the second prints `1` or more.
5. In a browser, the Counter page increments and the console shows no "Failed to connect via WebSockets" warning. Ask the user or use a browser tool.

For the WebAssembly variant: check 1 holds, `/healthz` returns `ok`, and a deep link such as `curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/counter"` returns 200. Every script and stylesheet the homepage references must load too; each line should start with `200`:

```bash
curl -s "https://$HOST/" | grep -oE '(src|href)="[^"]+\.(js|css)"' | cut -d'"' -f2 | sort -u | while read -r p; do
  echo "$(curl -s -o /dev/null -w '%{http_code}' "https://$HOST/${p#/}") $p"
done
```

The list should include the `_framework/blazor.webassembly` script.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| Console: "using the Long Polling fallback transport" | WebSockets blocked between browser and app | Check browser devtools and network proxies; the platform's HTTP port passes WebSockets |
| "Attempting to reconnect to the server" after every deploy | Old container stopped, circuit lost | Expected with one instance; the page reloads and starts a new circuit |
| `UnauthorizedAccessException` on `/keys` | Volume not writable by the non-root `app` user | Remove `USER $APP_UID` to run as root, or store keys in a database |
| Forms fail with antiforgery errors after a deploy | Keys not persisted | Check step 8 and check 4 |
| Logs show `Now listening on: http://[::]:80`, probes fail | `ASPNETCORE_URLS=http://+:80` set somewhere | Remove it or change it to 8080 |
| WebAssembly app 404s on refresh of a deep link | No `try_files` fallback | Use the `nginx.conf` above |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Don't delete the `dp-keys` volume. Losing the keys signs every user out and breaks pending forms. Ask before any delete.
- Don't raise `instances` above 1 with the volume attached; move keys to shared storage first.
- Never print connection strings or secrets; check that variables are set by name only.

## FAQ

### Where should I host my Blazor app?
For Interactive Server or Auto, on a host that runs ASP.NET Core as a long-lived process with WebSockets: Azure App Service, IIS, a Linux VM with nginx, or a container platform like the one in this guide. Standalone WebAssembly is static files, so any static host or a small nginx container works.

### Can I deploy a Blazor app to IIS instead?
Yes. Microsoft's docs say to enable WebSockets on IIS and session affinity with Application Request Routing when you run more than one server.

### Do I need Azure SignalR Service?
No. The docs call it optional and useful for very large numbers of connections or global distribution. A single instance handles circuits on its own.

### What's the difference between Blazor Server and a Blazor Web App?
Blazor Web App is the .NET 8+ template that lets you pick a render mode per component. With Interactive Server it hosts the same way as the older Blazor Server template; this guide covers both.

## Conclusion

You now have a Blazor Web App built from Git on the ASP.NET Core runtime image, serving interactive components over SignalR WebSockets on HTTPS, with data protection keys on a persistent volume so deploys don't break forms or sign users out. Standalone WebAssembly apps run from a small nginx container instead.

**Next:** add a database with [how to deploy a PostgreSQL database](deploy-postgresql-database.md), and your own domain with [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy an ASP.NET Core web application](deploy-aspnet-core-web-application.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Vite React app](deploy-vite-react-app.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy a WebSocket server](deploy-websocket-server.md)
