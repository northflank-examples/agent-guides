<!--
agent-guide-meta:
  id: NF-STK-NEST
  description: "How to deploy a NestJS app with a multi-stage Docker build, managed PostgreSQL as DATABASE_URL, a Terminus health check and migrations run as a separate job."
  sources:
    - https://docs.nestjs.com/deployment
    - https://docs.nestjs.com/recipes/terminus
    - https://northflank.com/docs/v1/api/use-the-cli
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/project/jobs/check-repository-access-for-jobs
    - https://northflank.com/docs/v1/api/project/jobs/list-job-builds
    - https://northflank.com/docs/v1/api/project/jobs/start-job-build
    - https://northflank.com/docs/v1/api/project/jobs/run-job
    - https://northflank.com/docs/v1/api/project/jobs/get-run-details
    - https://northflank.com/docs/v1/application/release/run-migrations
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DEP-004, NF-DB-001, NF-JOB-002, NF-DB-003, NF-DEP-012]
-->

# How to deploy a NestJS app

To deploy a NestJS app, build it with a multi-stage Docker image that runs `nest build` and starts `node dist/main.js`, listen on `process.env.PORT` and `0.0.0.0`, connect a managed PostgreSQL database through `DATABASE_URL`, add a `/health` endpoint with `@nestjs/terminus`, and run TypeORM or Prisma migrations as a separate one-off step. Migrating at startup or with `synchronize: true` causes races once several instances run.

This guide does it on Northflank with the CLI: the API built from Git as a combined service with a readiness probe on `/health`, PostgreSQL linked as `DATABASE_URL`, and migrations as a manual job with a clear pass or fail. You end with a public HTTPS URL whose health check reports the database up. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a NestJS app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a NestJS app and confirm the team, repository, branch and project before changing anything, then give Northflank access to the repository (once).
3. Listen on `process.env.PORT` and `0.0.0.0`, add `/health` with `@nestjs/terminus` and a database ping, and add a multi-stage `Dockerfile` that runs `node dist/main.js`. Push.
4. Create a project and PostgreSQL addon, link `POSTGRES_URI` as `DATABASE_URL`, and create the API with `northflank put service combined --file api.json`.
5. Run migrations as a manual job (`typeorm migration:run` or `prisma migrate deploy`).
6. Check that the live build is your pushed commit, the migrate run concluded `SUCCESS`, and `https://$HOST/health` reports the database `up`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your NestJS app on Northflank

A NestJS backend is a long-running Node.js server: it needs a host that keeps the process up on a port, builds TypeScript to `dist/`, connects to a database and runs migrations once per release. Serverless hosts can run Nest behind an adapter, but a persistent server with a connection pool fits more naturally on a container platform or a VM. A VM or your own Kubernetes cluster gives full control but leaves the OS, TLS, database and rollouts to you; a container platform builds your Dockerfile and runs it next to a managed database and migration jobs.

The TypeScript compiler, the Nest CLI and type packages are only needed to produce `dist/`. A multi-stage build leaves them out of the runtime image. Migrations change shared state once per release, so running them from every API instance at startup causes races when you scale.

On Northflank, the API runs as a **combined service** that builds from Git on every push, the database is a PostgreSQL **addon**, and migrations run as a **manual job** built from the same repository.

## Prerequisites

- **A NestJS app that builds locally:**
  ```bash
  npm ci && npm run build && ls dist/main.js dist/src/main.js 2>/dev/null
  ```
  The path it prints is the build output entry the `Dockerfile` runs (`dist/main.js`, or `dist/src/main.js` if root `.ts` files are compiled too). Know which ORM the app uses (TypeORM or Prisma).
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
jq -r '"nestjs: \(.dependencies["@nestjs/core"] // "missing")  typeorm: \(.dependencies.typeorm // "-")  prisma: \(.dependencies.prisma // .devDependencies.prisma // "-")"' package.json
ls package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId nest-app -o json >/dev/null 2>&1 \
  && echo "project: nest-app exists and will be reused" \
  || echo "project: nest-app will be created"
```

- If `nestjs` says `missing`, this isn't a NestJS app. Stop and use a guide for your framework.
- The lockfile tells you the package manager. Step 6 assumes `package-lock.json`; for pnpm or yarn, swap the install command.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "nest-app", "region": "europe-west" }'
export PROJECT_ID=nest-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Prepare the app

**Port and shutdown.** In `src/main.ts`:

```ts
const app = await NestFactory.create(AppModule);
app.enableShutdownHooks();
await app.listen(process.env.PORT ?? 3000, '0.0.0.0');
```

**Database config.** TypeORM in `AppModule`, with schema sync off:

```ts
TypeOrmModule.forRoot({
  type: 'postgres',
  url: process.env.DATABASE_URL,
  autoLoadEntities: true,
  synchronize: false,
}),
```

For TypeORM's CLI, add `src/data-source.ts` exporting `new DataSource({ type: 'postgres', url: process.env.DATABASE_URL, entities: ['dist/**/*.entity.js'], migrations: ['dist/migrations/*.js'] })`. With Prisma, point the datasource at `DATABASE_URL` (in `schema.prisma` or `prisma.config.ts`, depending on your Prisma version), and move `prisma` from `devDependencies` to `dependencies` so the migration job can run its CLI. If `prisma.config.ts` reads the URL with `env('DATABASE_URL')`, every Prisma command, including `prisma generate` in the Docker build, fails when the variable is unset; read it with `process.env.DATABASE_URL` instead, since `DATABASE_URL` isn't available at build time. Prisma 7 doesn't load `.env` files by itself.

**Health check.** Install `@nestjs/terminus`, import `TerminusModule` in a `HealthModule`, and add a controller, as in the [NestJS Terminus recipe](https://docs.nestjs.com/recipes/terminus):

```ts
@Controller('health')
export class HealthController {
  constructor(private health: HealthCheckService, private db: TypeOrmHealthIndicator) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([() => this.db.pingCheck('database')]);
  }
}
```

With Prisma, use `PrismaHealthIndicator` and `pingCheck('database', prismaClient)` instead. The endpoint returns 503 when the database is unreachable, so the platform stops routing traffic to an instance that can't serve.

## Step 6: Write a multi-stage Dockerfile

The NestJS docs show a single-stage image based on `node:24`; this splits it so the runtime stage has only production dependencies:

```dockerfile
FROM node:24-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
# Prisma: RUN npx prisma generate
RUN npm run build && npm prune --omit=dev

FROM node:24-slim
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build --chown=node:node /app/package*.json ./
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
# Prisma: COPY --from=build --chown=node:node /app/prisma ./prisma  (plus prisma.config.ts if you have one)
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

Add a `.dockerignore` with `node_modules`, `dist` and `.git`. Match the Node version to the one you develop with. `DATABASE_URL` isn't available during the build, so nothing in the build may connect to the database.

Northflank builds what's on the remote branch, not your local working tree, so commit and push these files and the step 5 changes before you create the service:

```bash
git add Dockerfile .dockerignore src/ package.json package-lock.json
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Create PostgreSQL and link `DATABASE_URL`

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "nest-db",
  "type": "postgresql",
  "version": "17-latest",
  "tlsEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId nest-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and read the addon logs"; false; }

northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "nest-env",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": false },
  "addonDependencies": [
    { "addonId": "nest-db", "keys": [ { "keyName": "POSTGRES_URI", "aliases": ["DATABASE_URL"] } ] }
  ]
}'
```

Don't continue unless the addon reached `running`. `POSTGRES_URI` has the form `postgresql://user:password@host:port/database`. The group is unrestricted, so both the API and the migration job can read it, and it's injected at runtime only. With `tlsEnabled: false` (the API default), `pg` and Prisma connect without TLS on the private network, which matches their defaults. If you enable TLS, set `ssl` in the TypeORM options or `sslmode` for Prisma. Storage is in MB and can only be increased later.

## Step 8: Create the API service

Write `api.json`:

```json
{
  "name": "api",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 3000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "3000", "NODE_ENV": "production" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/health", "port": 3000,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file api.json
```

Use the same repository URL, provider and branch as in step 4. The first build starts automatically. If none appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json --input '{}'`. Only a readiness probe is set, so a database outage takes instances out of rotation without restarting them.

## Step 9: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds, staying under the API limit of 1,000 requests per hour, and prints your URL when the app is live. `COMPLETED` means rolled out and serving.

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

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId api --lineLimit 200`. Don't continue to step 10 unless the rollout completed.

## Step 10: Run migrations as a job

Write `migrate.json`. It builds the same image and overrides the command. For Prisma, use `npx prisma migrate deploy` as the `customCommand`. `put job` makes a manual job because `settings.cron` is absent (`put job manual` is deprecated in the API). Use the same repository URL, provider and branch as in step 8. If you want to check access for jobs too, `northflank get job repository-access` takes the same body as the step 4 check.

```json
{
  "name": "api-migrate",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "settings": { "backoffLimit": 0, "activeDeadlineSeconds": 900, "runOnSourceChange": "never" },
  "deployment": {
    "vcs": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
    "docker": { "configType": "customCommand", "customCommand": "npx typeorm migration:run -d dist/data-source.js" }
  },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } }
}
```

```bash
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file migrate.json

OK=0
for i in $(seq 1 90); do
  JB=$(northflank get job builds --projectId "$PROJECT_ID" --jobId api-migrate --noDefaults -o json \
       | jq -r '(.data // .).builds | sort_by(.createdAt) | last | .status // "NONE"')
  echo "job build=$JB"
  case "$JB" in
    SUCCESS) OK=1; break ;;
    FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo JOB_BUILD_FAILED; break ;;
    NONE) [ "$i" = 6 ] && northflank start job build --projectId "$PROJECT_ID" --jobId api-migrate --noDefaults -o json --input '{}' ;;
  esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "JOB_BUILD_NOT_COMPLETE: stop here and read the job build logs"; false; }

RUN_ID=""; [ "$OK" = 1 ] && RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId api-migrate --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
RUN_OK=0
[ -n "$RUN_ID" ] && for i in $(seq 1 90); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId api-migrate --runId "$RUN_ID" --noDefaults -o json)
  echo "run=$(echo "$R" | jq -r '(.data // .).status')"
  if [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ]; then
    [ "$(echo "$R" | jq -r '(.data // .).status')" = "SUCCESS" ] && RUN_OK=1; break
  fi
  sleep 10
done
[ "$RUN_OK" = 1 ] || echo "MIGRATION_NOT_SUCCESSFUL: read the job logs below before relying on the new schema"
northflank get job logs --projectId "$PROJECT_ID" --jobId api-migrate --runId "$RUN_ID" --lineLimit 200
```

Run status is `RUNNING`, `SUCCESS` or `FAILED`. Start a new run after each release that adds migrations ([run migrations](https://northflank.com/docs/v1/application/release/run-migrations)).

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The migrate run from step 10 concluded with `status == "SUCCESS"` (the loop prints `MIGRATION_NOT_SUCCESSFUL` otherwise):
   ```bash
   northflank get job run --projectId "$PROJECT_ID" --jobId api-migrate --runId "$RUN_ID" --noDefaults -o json \
     | jq -r '(.data // .) | "\(.status) concluded=\(.concluded)"'
   ```
3. The health endpoint returns 200 and reports the database `up`, which proves the app reached PostgreSQL:
   ```bash
   curl -s -w '\n%{http_code}\n' "https://$HOST/health"   # {"status":"ok","info":{"database":{"status":"up"}},...} then 200
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Cannot find module '/app/dist/main.js'` | Entry is `dist/src/main.js` | Change `CMD` to the path `nest build` produces |
| `Cannot find module '@nestjs/cli'` or `typescript` at runtime | A start script that builds, such as `nest start` | Run `node dist/main.js`, not `npm run start` |
| 502/503 while `COMPLETED` | Listening on another port or `localhost` | `app.listen(process.env.PORT ?? 3000, '0.0.0.0')` |
| `/health` returns 503 | Database ping failing | Check `DATABASE_URL` is set; restart after secret changes |
| Migrate job: `prisma: not found` | Prisma CLI pruned as a dev dependency | Move `prisma` to `dependencies`; rebuild the job |
| Migrate job: `No migrations are pending` but tables missing | `migrations` glob points at `src/*.ts` | Point the data source at `dist/migrations/*.js` |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` (and to the job's `deployment.vcs`) as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Don't rely on the new schema until the step 10 run concluded with `SUCCESS`.
- Never set `synchronize: true` in production to "fix" missing tables. Generate a migration and run the job.
- Never print `DATABASE_URL`. Check that it's set by name only.
- Take a backup before migrations on a database with data, and ask before `migration:revert` or `prisma migrate reset` (which drops data).
- Storage can only be increased. Don't delete the addon, service or job without explicit, per-resource confirmation.

## FAQ

### Where should I deploy my NestJS app?
On a host that runs a long-running Node.js process with a port, such as a container platform, a VM or Kubernetes. Serverless hosts need an adapter and suit short request handlers better than a persistent API with a database pool. A container platform that builds a multi-stage `Dockerfile`, provides managed Postgres and runs migrations as a separate job covers what a NestJS backend needs in production.

### TypeORM or Prisma?
Both work the same way here. The only differences are the database config, the health indicator and the migration command in the job.

### Why not run migrations in `main.ts`?
With several instances, each would try to migrate at startup. A job runs once and gives you a pass or fail status before you depend on the new schema.

### Is `DATABASE_URL` available at build time?
No. The secret group uses `secretType: "environment"`, which injects at runtime only.

### Can I add a queue worker?
Yes. Deploy a second service from the same repo with a custom command that starts the worker entry, for example `node dist/worker.js`.

## Conclusion

You now have a NestJS API built in a multi-stage image, running on port 3000 behind a public HTTPS URL, connected to managed PostgreSQL through `DATABASE_URL`, with a Terminus health check and migrations that run as a tracked job.

**Next:** add Redis for caching or BullMQ queues, move slow work into a background worker, and schedule database backups. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a full stack app](deploy-full-stack-app.md)
