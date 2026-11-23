<!--
agent-guide-meta:
  id: NF-STK-DOTNET
  description: "How to deploy an ASP.NET Core web application with Docker on Linux, port 8080, managed PostgreSQL via an Npgsql connection string and database health checks."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://learn.microsoft.com/en-us/dotnet/core/compatibility/containers/8.0/aspnet-port
    - https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks
    - https://www.npgsql.org/doc/connection-string-parameters.html
    - https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/whatsnew#protection-against-concurrent-migrations
  related: [NF-DEP-001, NF-DB-001, NF-STK-SPRING, NF-JOB-002]
-->

# How to deploy an ASP.NET Core web application

To deploy an ASP.NET Core web application, publish it with the .NET SDK image in a multi-stage Dockerfile, run it on the ASP.NET runtime image with `ASPNETCORE_HTTP_PORTS=8080`, build the Npgsql connection string from separate host, port, database and credential variables, and add a database health check. ASP.NET Core ignores `PORT`, and Npgsql rejects `postgresql://` URLs, which cause most failed first deploys.

This guide does it on Northflank with the CLI: managed PostgreSQL linked through a secret group, EF Core migrations on startup, and separate `/healthz` and `/livez` probes. You end with a public HTTPS URL whose `/healthz` returns `Healthy`, with Kestrel logging port 8080. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end; a human links your Git account once if needed.

## TL;DR: how to deploy an ASP.NET Core web application

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo has an ASP.NET Core web project, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Build the connection string with `NpgsqlConnectionStringBuilder`, add `/healthz` and `/livez`, and push a `Dockerfile` that publishes with `dotnet/sdk:10.0` and runs on `dotnet/aspnet:10.0` with `ASPNETCORE_HTTP_PORTS=8080`.
4. Create Postgres with `northflank put addon` and link `HOST`, `PORT`, `DATABASE`, `USERNAME` and `PASSWORD` with `northflank put secret`.
5. Deploy with `northflank put service combined`, probing `/healthz` (database) and `/livez`.
6. Check that the live build is your pushed commit and that `curl https://$HOST/healthz` returns `Healthy`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why set the port and connection string explicitly?

Since .NET 8, the official ASP.NET Core images listen on port 8080 and set `ASPNETCORE_HTTP_PORTS`, according to Microsoft's breaking-change notice. Older guides that use `ASPNETCORE_URLS=http://+:80` or `EXPOSE 80` now point at a port nothing listens on. Setting the port in one place, and declaring the same number on the service, removes the guesswork.

On Northflank, the database is a PostgreSQL **addon**. A **secret group** links its connection details into the service as environment variables, so the password never passes through your terminal. A **combined service** builds the `Dockerfile` from Git on every push and serves it over HTTPS.

On where to host an ASP.NET Core application: it runs on Kestrel as a long-lived process, so it needs a host that keeps it up, plus a database. The traditional choices are IIS on Windows Server or a Linux VM behind a reverse proxy, where you manage the OS, runtime and TLS. Serverless function hosts suit small handlers more than a full web app. A container platform runs the Linux image from your `Dockerfile` with HTTPS and a managed database.

## Prerequisites

- **An ASP.NET Core app that builds locally:**
  ```bash
  dotnet publish -c Release -o /tmp/publish && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The project references** `Npgsql.EntityFrameworkCore.PostgreSQL` and `Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore`.

Names used below: project `dotnet-app`, addon `dotnet-db`, secret group `dotnet-db-env`, service `api`, DbContext `AppDb`.

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
grep -l 'Sdk="Microsoft.NET.Sdk.Web"' $(git ls-files '*.csproj') || echo "web project: missing"
grep -hoE '<(TargetFramework|AssemblyName)>[^<]+' $(git ls-files '*.csproj')
ls *.sln *.slnx Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId dotnet-app -o json >/dev/null 2>&1 \
  && echo "project: dotnet-app exists and will be reused" \
  || echo "project: dotnet-app will be created"
```

- If no `.csproj` uses `Microsoft.NET.Sdk.Web`, this isn't an ASP.NET Core web app. Stop and use a guide for your framework.
- The web `.csproj` is the one step 6 builds. Its `AssemblyName` (or the file name, if none prints) replaces `<ASSEMBLY>`, and its target framework sets the image tags (`net10.0` → `10.0`, `net8.0` → `8.0`).
- A `.sln` file means a solution layout: adjust the `COPY` paths in step 6.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "dotnet-app", "region": "europe-west" }'
export PROJECT_ID=dotnet-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Build the connection string and health checks in `Program.cs`

The secret group in step 8 injects `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER` and `DB_PASSWORD`. Configuration reads environment variables, so compose them like this:

```csharp
using Microsoft.EntityFrameworkCore;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);

var cs = builder.Configuration.GetConnectionString("Default");   // local dev: appsettings or ConnectionStrings__Default
if (string.IsNullOrEmpty(cs))
{
    cs = new NpgsqlConnectionStringBuilder
    {
        Host = builder.Configuration["DB_HOST"],
        Port = int.Parse(builder.Configuration["DB_PORT"] ?? "5432"),
        Database = builder.Configuration["DB_NAME"],
        Username = builder.Configuration["DB_USER"],
        Password = builder.Configuration["DB_PASSWORD"],
        SslMode = SslMode.Prefer,
    }.ConnectionString;
}

builder.Services.AddDbContext<AppDb>(o => o.UseNpgsql(cs));
builder.Services.AddHealthChecks().AddDbContextCheck<AppDb>(tags: new[] { "ready" });

var app = builder.Build();

if (app.Configuration["RUN_MIGRATIONS"] == "true")
{
    using var scope = app.Services.CreateScope();
    scope.ServiceProvider.GetRequiredService<AppDb>().Database.Migrate();
}

app.MapHealthChecks("/healthz", new() { Predicate = c => c.Tags.Contains("ready") });
app.MapGet("/livez", () => "ok");
app.MapControllers();   // or your minimal API endpoints
app.Run();
```

`NpgsqlConnectionStringBuilder` escapes special characters in the password, which string concatenation doesn't. `/healthz` runs the EF Core database check. `/livez` stays independent, so a database blip doesn't restart the app.

TLS ends at the platform's load balancer, so the app receives plain HTTP. `app.UseHttpsRedirection()` can't find an HTTPS port inside the container and only logs a warning; remove it. If you need `Request.Scheme` to read `https`, add `app.UseForwardedHeaders()` configured for `XForwardedProto`. It hasn't been confirmed that the load balancer sends that header, so test it before you depend on it.

## Step 6: Write a multi-stage Dockerfile

Put this next to the `.csproj`, or adjust the paths for a solution layout:

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

Restoring before copying the source keeps NuGet packages in a cached layer. `APP_UID` is defined by the official images and selects the non-root `app` user. Add a `.dockerignore` with `bin/`, `obj/` and `.git`; a copied `obj/` from a Windows or macOS machine can break `dotnet publish`.

`ASPNETCORE_URLS=http://+:8080` also works. If both are set, `ASPNETCORE_URLS` wins and the app logs that it's overriding `HTTP_PORTS`. Set only one.

Northflank builds what's on the remote branch, not your local working tree, so commit and push these files before creating the service:

```bash
git add Dockerfile .dockerignore Program.cs
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Create the PostgreSQL database

Storage is in MB and can only be increased later, so start small.

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "dotnet-db",
  "type": "postgresql",
  "version": "17-latest",
  "tlsEnabled": true,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId dotnet-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

## Step 8: Link the connection details

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "dotnet-db-env",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": false },
  "addonDependencies": [
    { "addonId": "dotnet-db", "keys": [
      { "keyName": "HOST",     "aliases": ["DB_HOST"] },
      { "keyName": "PORT",     "aliases": ["DB_PORT"] },
      { "keyName": "DATABASE", "aliases": ["DB_NAME"] },
      { "keyName": "USERNAME", "aliases": ["DB_USER"] },
      { "keyName": "PASSWORD", "aliases": ["DB_PASSWORD"] }
    ] }
  ],
  "secrets": { "variables": { "ASPNETCORE_ENVIRONMENT": "Production", "RUN_MIGRATIONS": "true" } }
}'
```

The variables are injected at runtime only. The addon's `POSTGRES_URI` is a `postgresql://` URL, which Npgsql doesn't parse, so link the individual keys. To see which keys the addon provides, print names only:

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId dotnet-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

## Step 9: Create the service

Write `dotnet-api.json`. The port matches `ASPNETCORE_HTTP_PORTS`. A small API runs on `nf-compute-20` (512 MB); `nf-compute-50` (1024 MB), used here, gives EF Core and the JIT more headroom.

```json
{
  "name": "api",
  "billing": { "deploymentPlan": "nf-compute-50" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "ASPNETCORE_HTTP_PORTS": "8080" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/healthz", "port": 8080,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/livez", "port": 8080,
      "initialDelaySeconds": 20, "periodSeconds": 20, "timeoutSeconds": 3, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file dotnet-api.json
```

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined`. The first build starts automatically; if none appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json --input '{}'`. Keep `instances` at 1 for the first deploy, while `RUN_MIGRATIONS` creates the schema.

## Step 10: Wait for the build and rollout

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. Poll every 10 seconds to stay under the API limit of 1,000 requests per hour. If the loop prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId api --lineLimit 200`. The troubleshooting table covers the common causes.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The database health check passes:
   ```bash
   curl -s "https://$HOST/healthz"   # Healthy
   ```
3. Kestrel is listening on 8080 and nothing failed at startup:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId api --lineLimit 300 --textIncludes "Now listening on"
   ```
   This prints `Now listening on: http://[::]:8080`.
4. The connection variables are injected, without printing their values. `exec` drops output without a TTY, so wrap it in `script` (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId api \
     --shell-cmd 'sh -c' --cmd '"for v in DB_HOST DB_PORT DB_NAME DB_USER DB_PASSWORD; do printenv \$v >/dev/null && echo \$v=set; done"'
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| Health checks fail, logs show `Now listening on: http://localhost:5000` | App built with `WebHost.CreateDefaultBuilder` ignores `ASPNETCORE_HTTP_PORTS` | Set `ASPNETCORE_URLS=http://+:8080` instead |
| Logs show port 80, probes fail | `ASPNETCORE_URLS=http://+:80` left in the image or settings | Remove it, or change it to 8080 |
| `Format of the initialization string does not conform to specification` | A `postgresql://` URL passed to Npgsql | Use the builder from step 5 |
| `/healthz` returns `Unhealthy` | `DB_*` missing or addon not `running` | Re-check steps 7–8, then `northflank restart service --projectId "$PROJECT_ID" --serviceId api` |
| `Permission denied` writing files | Running as non-root `app` | Write only to `/tmp`, or use a volume |
| `dotnet publish` fails with assets file errors | Local `obj/` copied into the build | Add `obj/` and `bin/` to `.dockerignore` |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 10 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print `DB_PASSWORD` or a full connection string. Check that variables are set, as in check 4.
- Back up the database before deploying a release whose migrations drop or rewrite data. Set `RUN_MIGRATIONS` to `false` if the user prefers to apply migrations by hand.
- Addon storage can only be increased. Don't delete the addon, service or project without explicit, per-resource confirmation.

## FAQ

### Where should I deploy my ASP.NET Core application?
If you're already on Windows servers, IIS is a reasonable choice. Otherwise, a Linux container is the simplest path, because the official runtime images already include everything Kestrel needs. A container platform runs that image with HTTPS and a managed database, with no server to patch.

### Should I set `ASPNETCORE_URLS` or `ASPNETCORE_HTTP_PORTS`?
Either one. `ASPNETCORE_HTTP_PORTS` takes a list of port numbers. `ASPNETCORE_URLS` takes full URLs and is the only one older `WebHost`-based apps read. Set one, and keep it equal to `internalPort`.

### Can I pass the connection string as `ConnectionStrings__Default`?
Yes. .NET maps `ConnectionStrings__Default` to `GetConnectionString("Default")`. The platform gives the parts as separate keys, though, so composing them in code avoids storing a second copy of the password.

### Does this work for Blazor Server or MVC apps?
Yes. The port, Dockerfile and database steps are the same. Blazor Server uses long-lived connections, so test it with more than one instance before you scale out.

## Conclusion

You now have an ASP.NET Core app built from Git with the .NET SDK, running on the ASP.NET runtime image on port 8080, connected to managed PostgreSQL through a properly escaped Npgsql connection string, with separate readiness and liveness checks.

**Next:** add Redis for caching or a background worker for queued jobs. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy a Spring Boot application](deploy-spring-boot-application.md)
