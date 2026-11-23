<!--
agent-guide-meta:
  id: NF-DEP-004
  description: "How to deploy a Node.js application in production with Docker, managed PostgreSQL linked as DATABASE_URL and a /health check that runs a real database query."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/create-addon
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/secrets/create-project-secret
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/create-combined-service
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/api/forwarding
  related: [NF-DB-001, NF-DB-003, NF-JOB-002, NF-DEP-012, NF-DEP-001]
-->

# How to deploy a Node.js application

To deploy a Node.js application in production, package it in a Docker image that listens on `process.env.PORT` and `0.0.0.0`, connect a managed PostgreSQL database through `DATABASE_URL`, add a `GET /health` route that runs `SELECT 1` for health checks, and run migrations once the new build is live. Binding to `localhost` or ignoring `DATABASE_URL` are the usual reasons a Node.js server fails its health check or hits `127.0.0.1:5432`.

This guide does it on Northflank with the CLI: a PostgreSQL addon, a secret group that links its connection string as `DATABASE_URL`, and a combined service built from your repo's `Dockerfile`. You end with an Express, Fastify or NestJS API on a public HTTPS URL whose `/health` returns 200 after a real database query. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a Node.js application

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Node.js server and confirm the team, repository, branch and project before changing anything, then give Northflank access to the repository (once).
3. Make the app read `DATABASE_URL`, listen on `PORT` and `0.0.0.0`, and serve a `/health` route that runs `SELECT 1`; add a `Dockerfile` and push.
4. Create a PostgreSQL addon, link its `POSTGRES_URI` as `DATABASE_URL` with a secret group, and create the service with `northflank put service combined --file api.json`.
5. Wait for the deploy, then run migrations with `northflank exec service`.
6. Check that the live build is your pushed commit and that `https://$HOST/health` returns 200.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your Node.js application on Northflank

A Node.js server needs a host that keeps a long-running process up, gives it a port and a public URL, and usually a database plus a way to run migrations. Static and serverless hosts suit frontends and short request handlers, but a persistent Node.js server with a connection pool fits better on a container platform or a VM. A raw VM or your own Kubernetes cluster gives full control, at the cost of managing the OS, process manager, TLS and database yourself; a container platform builds your Dockerfile and runs it with health checks and managed databases.

Hard-coding a connection string means copying a password into config files and updating it by hand. On Northflank, you use three pieces instead: a PostgreSQL **addon**, a **secret group** that links the addon's `POSTGRES_URI` as `DATABASE_URL`, and a **combined service** that builds the repo's Dockerfile. The app reads `DATABASE_URL` like it would anywhere else, and the secret never passes through your terminal.

## Prerequisites

- **A Node.js server that runs locally:**
  ```bash
  npm ci && npm start
  ```
  Stop it with Ctrl+C once it says it's listening. This guide works for Express, Fastify and NestJS.
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
jq -r '"start: \(.scripts.start // "missing")  main: \(.main // "-")  framework: \([(.dependencies // {}) | keys[] | select(. == "express" or . == "fastify" or . == "@nestjs/core")] | join(",") | if . == "" then "none" else . end)"' package.json
ls package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId node-api -o json >/dev/null 2>&1 \
  && echo "project: node-api exists and will be reused" \
  || echo "project: node-api will be created"
```

- If `package.json` is missing, this isn't a Node.js app. Stop and use a guide for your language. If `start` says `missing`, find the command that starts the server; the `Dockerfile` in step 5 runs it directly.
- The lockfile tells you the package manager. Step 5 assumes `package-lock.json`; for pnpm or yarn, swap the install command.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "node-api", "region": "europe-west" }'
export PROJECT_ID=node-api

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Prepare the app and push

Fix the app before deploying if any item fails:

- Reads the database URL from `process.env.DATABASE_URL`.
- Listens on `process.env.PORT` and host `0.0.0.0` (not `localhost`). NestJS: `app.listen(process.env.PORT ?? 8080, '0.0.0.0')`. Fastify: `fastify.listen({ port, host: '0.0.0.0' })`.
- Has `GET /health` that runs `SELECT 1` against the pool and returns 200 only on success. This makes the platform health check and your verification a real DB check.
- Has a `Dockerfile` (e.g. `FROM node:22-slim`, `npm ci --omit=dev`, `CMD ["node","dist/main.js"]`). If there's none, write one; this guide builds with Dockerfile.

Northflank builds what's in your repository, not the files on your machine. Commit and push the `Dockerfile` and any app changes you made (the `/health` route, `PORT` and `0.0.0.0` binding):

```bash
git add Dockerfile <CHANGED_APP_FILES>
git commit -m "Add Northflank deployment files"
git push
```

Names used below: addon `api-db`, secret group `api-db-env`, service `api`. IDs are slugified names.

## Step 6: Create the PostgreSQL addon

```bash
northflank get addon types --noDefaults -o json | jq '.'   # confirm type "postgresql" and available versions
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "api-db",
  "type": "postgresql",
  "version": "<MAJOR>-latest",
  "tlsEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'
```

`version` accepts `latest` or `<major>-latest` (e.g. `16-latest`); pin a major for production. `tlsEnabled: false` is the API default, so drivers such as `pg` connect without TLS on the private network, which matches their defaults; if you enable TLS, turn on the driver's `ssl` option too. Storage can only be increased later, never reduced, so start small. The 6144 MB storage value comes from the API example; it hasn't been confirmed as the documented minimum.

REST: `PUT https://api.northflank.com/v1/projects/{projectId}/addons` (or `POST` to fail on duplicates).

## Step 7: Wait for the addon to be running

```bash
OK=0
for i in $(seq 1 60); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId api-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"; [ "$A" = "running" ] && { OK=1; break; }
  case "$A" in failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and read the addon logs"; false; }
```

Don't continue to step 8 unless the addon is `running`.

## Step 8: Link the connection string as `DATABASE_URL`

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "api-db-env",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": true, "nfObjects": [ { "id": "api", "type": "service" } ] },
  "addonDependencies": [
    { "addonId": "api-db", "keys": [ { "keyName": "POSTGRES_URI", "aliases": ["DATABASE_URL"] } ] }
  ],
  "secrets": { "variables": { "NODE_ENV": "production" } }
}'
```

`secretType: "environment"` injects at runtime only (not as build arguments). REST: `PUT https://api.northflank.com/v1/projects/{projectId}/secrets`.

Restricting the group to service `api` before that service exists hasn't been confirmed to work. If this call is rejected, re-run it with `"restrictions": { "restricted": false }` (all resources in the project can read it), or run it again after step 9 and then restart the service.

For details and alternatives (individual `HOST`/`PORT`/`USERNAME`/`PASSWORD` keys), see [connect database secrets to workloads](https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads).

## Step 9: Create the API service with a health check

Write `api.json`:

```json
{
  "name": "api",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "8080" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/health", "port": 8080,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/health", "port": 8080,
      "initialDelaySeconds": 30, "periodSeconds": 20, "timeoutSeconds": 2, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file api.json
```

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined`. The first build starts automatically. For a monorepo, set `dockerFilePath`/`dockerWorkDir` to the API folder (e.g. `/apps/api/Dockerfile`, `/apps/api`). `nf-compute-20` suits a small API; list other plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[].id'`.

## Step 10: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds and prints your URL when the app is live. `COMPLETED` means rolled out and serving.

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

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId api`.

## Step 11: Run migrations (if the project has them)

`exec` without a TTY drops output in agent harnesses, so wrap it with `script` (macOS/BSD shown; Linux: `script -qfc '<cmd>' /dev/null`). See the [execute command API docs](https://northflank.com/docs/v1/api/execute-command).

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId api \
  --shell-cmd 'sh -c' --cmd '"npx prisma migrate deploy"'    # or: npm run migrate, npx knex migrate:latest
```

## How to know it worked

You're done when the first four of these hold; the fifth is optional:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health endpoint (which runs `SELECT 1` in the app) returns 200:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/health"
   ```
3. `DATABASE_URL` is injected (prints `postgresql://`, never the full value):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId api \
     --shell-cmd 'sh -c' --cmd '"printenv DATABASE_URL | cut -c1-13"'
   ```
4. The app log shows a DB connection and no connection errors (adjust the text to the app's log line):
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId api --lineLimit 200 --textIncludes "connect"
   ```
5. Optional, from your machine: `SELECT 1` through a forwarded tunnel. Run the forward in the background; `--skipHostnames` avoids `sudo`, and the CLI prints the local address and port:
   ```bash
   northflank forward addon --projectId "$PROJECT_ID" --addonId api-db --skipHostnames &
   CREDS=$(northflank get addon credentials --projectId "$PROJECT_ID" --addonId api-db --noDefaults -o json)
   PGPASSWORD=$(echo "$CREDS" | jq -r '(.data // .).secrets.password') psql -h <LOCAL_ADDR> -p <LOCAL_PORT> \
     -U "$(echo "$CREDS" | jq -r '(.data // .).secrets.username')" -d <DATABASE> -c 'SELECT 1'
   kill %1
   ```
   The credential key names and the database name key for PostgreSQL haven't been confirmed, so inspect `$CREDS` with `jq '(.data // .) | keys, (.secrets|keys)'` first.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Addon stuck in `allocating` > 10 min, or `failed` | Capacity / plan issue | `northflank get addon logs --help`; report to user, do not delete |
| `DATABASE_URL` empty in container | Secret group restricted to another ID, or created after the service started | Fix `restrictions`; restart: `northflank restart service --help` |
| `ECONNREFUSED 127.0.0.1:5432` in logs | App ignores `DATABASE_URL` and uses a local default | Read `process.env.DATABASE_URL` in the DB config |
| TLS / SSL errors connecting | Driver SSL options don't match the addon's TLS setting | Align driver `ssl` config with the addon's TLS setting (see PostgreSQL docs link in sources) |
| Health check failing, pod restarting | `/health` missing, wrong port, or listening on `localhost` | Bind `0.0.0.0:$PORT`; ensure route exists; see [configure health checks](https://northflank.com/docs/v1/application/observe/configure-health-checks) |
| Build `FAILURE` | `npm ci` without lockfile, wrong Dockerfile path | `northflank get service build-logs …`; see [build with a Dockerfile](https://northflank.com/docs/v1/application/build/build-with-a-dockerfile) |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 10 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print full connection strings or passwords; truncate as in check 3 of "How to know it worked".
- Addon storage can only be increased, never reduced; don't oversize "just in case".
- Before risky migrations on a database with data, take a backup (see [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data)).
- Don't delete the addon or service without explicit, per-resource user confirmation. Prefer pausing.

## FAQ

### Where should I deploy my Node.js application?
Anywhere that runs a long-running process with a port: a container platform, a VM, or Kubernetes. Static and serverless hosts are a better fit for frontends and short-lived functions than for a persistent server with a database pool. A container platform that builds your `Dockerfile`, runs health checks and provides a managed Postgres gives you a production setup without managing servers.

### Does this work for NestJS and Fastify as well as Express?
Yes. The deployment only needs the app to read `DATABASE_URL`, listen on `process.env.PORT` and `0.0.0.0`, serve `GET /health`, and have a `Dockerfile`. Step 5 shows the `listen` call for NestJS and Fastify.

### Is `DATABASE_URL` available at build time?
No. `secretType: "environment"` injects it at runtime only, not as build arguments.

### Why does my app connect to `127.0.0.1:5432`?
The app is ignoring `DATABASE_URL` and falling back to a local default. Read `process.env.DATABASE_URL` in the DB config.

### How do I run migrations without an interactive shell?
Wrap `northflank exec service` with `script`, as in step 11. Without a TTY, `exec` drops output in agent harnesses.

## Conclusion

You now have a Node.js API built from Git, connected to a managed PostgreSQL database through `DATABASE_URL`, with a health check that tests the database on every probe.

**Next:** for database details and sizing, see [how to deploy PostgreSQL](deploy-postgresql-database.md). To deploy a frontend, API and database as one reproducible run, see [how to deploy a full-stack app as one template](deploy-full-stack-app.md). For logs, see [view logs](https://northflank.com/docs/v1/application/observe/view-logs). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy a full stack app](deploy-full-stack-app.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
