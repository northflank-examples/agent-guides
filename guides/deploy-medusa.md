<!--
agent-guide-meta:
  id: NF-STK-MEDUSA
  description: "How to deploy Medusa v2 in production with Docker: a server and a worker from one repo, managed PostgreSQL and Redis, the admin build and migrations on start."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://docs.medusajs.com/learn/deployment/general
    - https://docs.medusajs.com/learn/production/worker-mode
    - https://docs.medusajs.com/learn/configurations/medusa-config
    - https://docs.medusajs.com/resources/infrastructure-modules/caching/providers/redis
    - https://docs.medusajs.com/resources/infrastructure-modules/event/redis
    - https://docs.medusajs.com/resources/infrastructure-modules/workflow-engine/redis
    - https://docs.medusajs.com/resources/infrastructure-modules/locking/redis
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/start-service-build
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-redis-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DB-001, NF-DB-003, NF-DEP-004, NF-DEP-005, NF-STK-MINIO]
-->

# How to deploy Medusa

To deploy Medusa (v2), run `medusa build` and start the `.medusa/server` output twice: a server instance (`MEDUSA_WORKER_MODE=server`, API and admin) and a worker instance (`MEDUSA_WORKER_MODE=worker`, admin disabled). Both read PostgreSQL from `DATABASE_URL` and Redis from `REDIS_URL`, and `medusa db:migrate` runs before the server starts. The admin bakes in `MEDUSA_BACKEND_URL` at build time, so rebuild once the public URL is known.

This guide does it on Northflank with the CLI: managed PostgreSQL and Redis addons, one secret group with the connection strings and generated `JWT_SECRET` and `COOKIE_SECRET`, and two services built from your repo's `Dockerfile`. You end with a public HTTPS URL where `/health` returns `OK`, the admin loads at `/app`, and an admin user can log in. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy Medusa

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Medusa v2 project and confirm the team, repository, branch and project before changing anything. Give Northflank access to the repository (once).
3. Read `workerMode`, `admin.disable`, `admin.backendUrl`, `redisUrl` and the Redis modules in `medusa-config.ts` from environment variables, add a `predeploy` script and a `Dockerfile`, and push.
4. Create PostgreSQL and Redis with `northflank put addon` and link `POSTGRES_URI` and `REDIS_MASTER_URL` into a secret group with `JWT_SECRET` and `COOKIE_SECRET`.
5. Deploy the server with `northflank put service combined`, set `MEDUSA_BACKEND_URL` and the CORS variables from its public host and rebuild, then deploy the worker with `MEDUSA_WORKER_MODE=worker` and create an admin user.
6. Check that the live build is your pushed commit, `/health` returns `OK`, `/app` returns 200 and `/auth/user/emailpass` returns a token.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why a server, a worker and two databases?

The Medusa deployment guide deploys PostgreSQL, Redis, and the app twice: in **server** mode (API and admin) and **worker** mode (scheduled jobs and subscribers), with at least 2 GB of RAM:

- **PostgreSQL** holds the commerce data.
- **Redis** backs sessions (`redisUrl`) and the production caching, event bus, workflow engine and locking modules.
- **The server** runs `medusa db:migrate` (the `predeploy` script) and then `medusa start`.
- **The worker** runs `medusa start` only, so background work doesn't slow API requests.

So where to deploy Medusa? Somewhere that runs two long-lived Node.js processes next to PostgreSQL and Redis. Serverless platforms don't fit the worker. A VM works if you manage supervision, TLS and databases, and Medusa Cloud is the managed option. A container platform with managed PostgreSQL and Redis runs both modes from one repo.

## Prerequisites

- **A Medusa v2 app that builds locally:**
  ```bash
  npm ci && npx medusa build && test -d .medusa/server && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **`openssl`** (to generate secrets).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

Names used below: project `medusa`, addons `medusa-db` and `medusa-redis`, secret group `medusa-env`, services `medusa` and `medusa-worker`.

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
jq -r '"medusa: \(.dependencies["@medusajs/medusa"] // "missing")  framework: \(.dependencies["@medusajs/framework"] // "missing")"' package.json
ls medusa-config.ts medusa-config.js package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId medusa -o json >/dev/null 2>&1 \
  && echo "project: medusa exists and will be reused" \
  || echo "project: medusa will be created"
```

- If both `medusa` and `framework` say `missing`, this isn't a Medusa app. Stop and use a guide for your framework. If the versions aren't `2.x`, this guide doesn't fit: Medusa v1 has a different config and no worker mode.
- If there's no `medusa-config.ts` (or `.js`), you're not in the Medusa backend folder. In a monorepo, `cd` into it and use that folder in `dockerFilePath` and `dockerWorkDir` later.
- The lockfile tells you the package manager. Step 5 assumes `package-lock.json`; for pnpm or yarn, swap the install commands.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "medusa", "region": "europe-west" }'
export PROJECT_ID=medusa

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Configure Medusa for production and push

**`medusa-config.ts` reads production settings from the environment**, as in the Medusa deployment guide. Install `@medusajs/caching-redis`, then merge these into your existing config:

```ts
module.exports = defineConfig({
  projectConfig: {
    // ...databaseUrl, http.storeCors/adminCors/authCors/jwtSecret/cookieSecret as generated
    redisUrl: process.env.REDIS_URL,
    workerMode: process.env.MEDUSA_WORKER_MODE as "shared" | "worker" | "server",
  },
  admin: {
    disable: process.env.DISABLE_MEDUSA_ADMIN === "true",
    backendUrl: process.env.MEDUSA_BACKEND_URL,
  },
  modules: [
    { resolve: "@medusajs/medusa/caching", options: { providers: [
      { resolve: "@medusajs/caching-redis", id: "caching-redis", is_default: true, options: { redisUrl: process.env.CACHE_REDIS_URL } } ] } },
    { resolve: "@medusajs/medusa/event-bus-redis", options: { redisUrl: process.env.REDIS_URL } },
    { resolve: "@medusajs/medusa/workflow-engine-redis", options: { redis: { redisUrl: process.env.REDIS_URL } } },
    { resolve: "@medusajs/medusa/locking", options: { providers: [
      { resolve: "@medusajs/medusa/locking-redis", id: "locking-redis", is_default: true, options: { redisUrl: process.env.LOCKING_REDIS_URL } } ] } },
  ],
})
```

Before v2.12.2, the workflow engine option was `url` instead of `redisUrl`.

**`package.json` has** `"predeploy": "medusa db:migrate"` in `scripts`.

**Add a `Dockerfile`** if step 2 found none (shown for npm; switch the install commands for yarn or pnpm). It installs dependencies inside `.medusa/server`, as the Medusa docs require, and runs `predeploy` only in server mode:

```dockerfile
FROM node:22-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ARG MEDUSA_BACKEND_URL
ARG DISABLE_MEDUSA_ADMIN=false
ENV MEDUSA_BACKEND_URL=$MEDUSA_BACKEND_URL DISABLE_MEDUSA_ADMIN=$DISABLE_MEDUSA_ADMIN
RUN npx medusa build

FROM node:22-slim
WORKDIR /app/server
ENV NODE_ENV=production
COPY --from=build /app/.medusa/server ./
RUN npm install
EXPOSE 9000
CMD ["sh", "-c", "if [ \"$MEDUSA_WORKER_MODE\" = server ]; then npm run predeploy || exit 1; fi; exec npm run start"]
```

Northflank builds what's in your repository, not the files on your machine, so commit and push them:

```bash
git add Dockerfile medusa-config.ts package.json package-lock.json
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create PostgreSQL and Redis

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "medusa-db", "type": "postgresql", "version": "17-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-50", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "medusa" }
}'
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "medusa-redis", "type": "redis", "version": "8-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 4096, "replicas": 1 },
  "typeSpecificSettings": { "redisMaxMemoryPolicy": "noeviction", "redisSentinelEnabled": false }
}'

ADDONS_OK=1
for a in medusa-db medusa-redis; do
  OK=0
  for i in $(seq 1 90); do
    S=$(northflank get addon --projectId "$PROJECT_ID" --addonId "$a" --noDefaults -o json | jq -r '(.data // .).status')
    echo "$a=$S"
    case "$S" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
    sleep 10
  done
  [ "$OK" = 1 ] || ADDONS_OK=0
done
[ "$ADDONS_OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and read the addon logs"; false; }
```

Don't continue unless both addons reached `running`.

Redis uses `noeviction` because it carries events and workflow state. Both addons accept only in-project traffic, with TLS off so the URLs work without driver SSL options.

## Step 7: Create the secret group

Generate the two secrets locally and pass them straight into the group, without echoing them:

```bash
JWT_SECRET=$(openssl rand -hex 32); COOKIE_SECRET=$(openssl rand -hex 32)
jq -n --arg jwt "$JWT_SECRET" --arg cookie "$COOKIE_SECRET" '{
  name: "medusa-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [
    { addonId: "medusa-db",    keys: [ { keyName: "POSTGRES_URI", aliases: ["DATABASE_URL"] } ] },
    { addonId: "medusa-redis", keys: [ { keyName: "REDIS_MASTER_URL", aliases: ["REDIS_URL", "CACHE_REDIS_URL", "LOCKING_REDIS_URL"] } ] }
  ],
  secrets: { variables: { JWT_SECRET: $jwt, COOKIE_SECRET: $cookie } }
}' > medusa-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file medusa-env.json >/dev/null && rm medusa-env.json
unset JWT_SECRET COOKIE_SECRET
```

Both services share this group. `secretType: "environment"` injects at runtime only, so database URLs never reach the build.

## Step 8: Check the linked keys

```bash
for a in medusa-db medusa-redis; do
  northflank get secret-link --projectId "$PROJECT_ID" --secretId medusa-env --addonId "$a" --noDefaults -o json \
    | jq '(.data // .) | {linked, linkedKeys, availableKeys}'
done
```

If a key is missing from `linkedKeys`, re-run step 7 with the exact name from `availableKeys`.

## Step 9: Deploy the server and get your URL

Write `medusa.json`:

```json
{
  "name": "medusa",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "buildArguments": { "DISABLE_MEDUSA_ADMIN": "false" },
  "ports": [ { "name": "http", "internalPort": 9000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "9000", "MEDUSA_WORKER_MODE": "server", "DISABLE_MEDUSA_ADMIN": "false" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "startupProbe", "path": "/health", "port": 9000,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 30, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/health", "port": 9000,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file medusa.json
```

Use the same repository URL, provider and branch as in step 4. `billing.buildPlan` is optional; leave it out to use the default build plan, and only set it if builds run out of memory, using a build plan your team has access to (an unavailable one fails with `Build plan not found`). The startup probe gives migrations up to five minutes. Keep one server instance while migrations run on start; to scale out, move `predeploy` to a job (the pattern is in [how to deploy a Django project](deploy-django-project.md)). The first build starts automatically.

The first build takes several minutes. This loop checks every 10 seconds and prints your URL when the server is live:

```bash
OK=0
for i in $(seq 1 120); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId medusa --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId medusa --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId medusa --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId medusa`. The troubleshooting table covers the common causes.

## Step 10: Set the backend URL and CORS, then rebuild

Send the **complete** maps. The admin bundle reads `MEDUSA_BACKEND_URL` at build time, so this needs a new build, not just a restart:

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId medusa --noDefaults -o json --input "$(jq -n --arg u "https://$HOST" '{
  buildArguments: { DISABLE_MEDUSA_ADMIN: "false", MEDUSA_BACKEND_URL: $u },
  runtimeEnvironment: { PORT: "9000", MEDUSA_WORKER_MODE: "server", DISABLE_MEDUSA_ADMIN: "false",
    MEDUSA_BACKEND_URL: $u, ADMIN_CORS: $u, AUTH_CORS: $u, STORE_CORS: $u }
}')"
northflank start service build --projectId "$PROJECT_ID" --serviceId medusa --noDefaults -o json --input '{ "branch": "main" }'
```

Then run the step 9 loop again and don't continue until it prints the URL. For a storefront, set `STORE_CORS` to its URL and append it to `AUTH_CORS`, then restart. For a custom domain, repeat this step.

## Step 11: Deploy the worker

The worker builds the same repo without the admin and has no public port. Write `medusa-worker.json`:

```json
{
  "name": "medusa-worker",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "buildArguments": { "DISABLE_MEDUSA_ADMIN": "true" },
  "runtimeEnvironment": { "MEDUSA_WORKER_MODE": "worker", "DISABLE_MEDUSA_ADMIN": "true", "PORT": "9000" }
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file medusa-worker.json
```

Wait for it with the step 9 loop, using `--serviceId medusa-worker` (it has no public port, so the URL line prints nothing). Because `MEDUSA_WORKER_MODE` is `worker`, the `CMD` skips `predeploy`, so only the server runs migrations.

## Step 12: Create an admin user

Ask the user which email to use, then generate a password and create the user. `exec` needs `script` to keep output without a TTY (Linux: `script -qefc '<cmd>' /dev/null`):

```bash
ADMIN_EMAIL=<EMAIL>; ADMIN_PASSWORD=$(openssl rand -base64 18)
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId medusa \
  --shell-cmd 'sh -c' --cmd "\"cd /app/server && npx medusa user -e $ADMIN_EMAIL -p $ADMIN_PASSWORD\""
```

**Hand `ADMIN_PASSWORD` to the user** to store, and ask them to change it after the first login.

## How to know it worked

You're done when all of these hold:

1. The live builds of the server and the worker are the commit you pushed. All three values should match:
   ```bash
   git rev-parse HEAD
   for s in medusa medusa-worker; do
     northflank get service builds --projectId "$PROJECT_ID" --serviceId "$s" --noDefaults -o json \
       | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   done
   ```
2. The server is healthy (prints `OK`):
   ```bash
   curl -s "https://$HOST/health"
   ```
3. The admin dashboard is served:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/app"   # 200
   ```
4. The admin user can authenticate against the API (prints `true`, never the token):
   ```bash
   curl -s -X POST "https://$HOST/auth/user/emailpass" -H 'Content-Type: application/json' \
     -d "$(jq -n --arg e "$ADMIN_EMAIL" --arg p "$ADMIN_PASSWORD" '{email: $e, password: $p}')" | jq 'has("token")'
   ```
5. The worker is running in worker mode with no errors:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId medusa-worker --lineLimit 200 --textIncludes "worker"
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Server exits with a missing `JWT_SECRET`/`COOKIE_SECRET` error | Secret group not injected | Check `medusa-env` exists and is unrestricted; restart |
| `ECONNREFUSED 127.0.0.1:5432` or `:6379` | Keys not linked | Re-check step 8 |
| Admin loads but every request fails or hits `localhost:9000` | Admin built without `MEDUSA_BACKEND_URL` | Redo step 10, including the build |
| CORS errors in the browser | `ADMIN_CORS`/`AUTH_CORS` don't match the URL | Set them to `https://$HOST` exactly; restart |
| Startup probe fails during first deploy | Migrations take longer than five minutes | Raise `failureThreshold`; check `get service logs` |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` in both service files as `"vcsLinkId": "<ID>"` |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create a service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live builds are the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print `JWT_SECRET`, `COOKIE_SECRET`, the admin password or connection URLs. Never regenerate the secrets on a live store, because that logs everyone out.
- Ask the user for the admin email instead of inventing one.
- Back up the database before upgrading Medusa, because `predeploy` runs migrations on start. See [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data).
- Don't delete the addons or services without explicit, per-resource confirmation.

## FAQ

### Where should I deploy Medusa?
On a host that runs two long-lived Node.js processes (server and worker) next to PostgreSQL and Redis. Serverless platforms built for short functions don't fit the worker. A container platform with managed databases runs both modes from one repo, a VM works if you manage the rest yourself, and Medusa Cloud is the hosted option.

### Do I need a separate worker?
For production, yes. Without one, `workerMode: "shared"` runs scheduled jobs and subscribers inside the API process, where they compete with requests.

### How do I deploy the storefront?
Deploy the Next.js starter as its own service (see [how to deploy a Next.js app](deploy-nextjs-app.md)), then add its URL to `STORE_CORS` and `AUTH_CORS` on the server.

## Conclusion

You now have Medusa v2 running as a server and a worker from one repo, with managed PostgreSQL and Redis, migrations applied on start, and an admin dashboard that knows its own URL.

**Next:** switch file storage to the S3 File Module Provider (the default local provider writes to disk that's replaced on every deploy), add a custom domain with [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank) and repeat step 10 with it. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a Next.js app](deploy-nextjs-app.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
- [How to deploy a Node.js application](deploy-nodejs-application.md)
