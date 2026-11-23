<!--
agent-guide-meta:
  id: NF-STK-PAYLOAD
  description: "How to deploy Payload CMS with Docker, a standalone Next.js build and managed PostgreSQL or MongoDB, with migrations on start and every step scripted."
  sources:
    - https://payloadcms.com/docs/production/deployment
    - https://payloadcms.com/docs/production/building-without-a-db-connection
    - https://payloadcms.com/docs/database/postgres
    - https://payloadcms.com/docs/database/mongodb
    - https://payloadcms.com/docs/database/migrations
    - https://payloadcms.com/docs/upload/storage-adapters
    - https://github.com/payloadcms/payload/blob/main/templates/blank/.env.example
    - https://github.com/payloadcms/payload/blob/main/templates/with-postgres/src/payload.config.ts
    - https://northflank.com/stacks/deploy-payload
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-mongodb-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/secrets/create-project-secret
    - https://northflank.com/docs/v1/api/project/secrets/get-project-secret-details
    - https://northflank.com/docs/v1/api/project/secrets/patch-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
  related: [NF-DEP-005, NF-STK-STRAPI, NF-DB-001, NF-STK-MONGODB]
-->

# How to deploy Payload CMS

To deploy Payload CMS, build it as a standalone Next.js Docker image with `next build --experimental-build-mode compile` so the build needs no database, connect a managed PostgreSQL or MongoDB database through `DATABASE_URL`, set a random `PAYLOAD_SECRET`, and run Postgres migrations on start with `prodMigrations`. Schema push only happens in development, so without committed migrations production fails with `relation "users" does not exist`.

This guide does it on Northflank with the CLI: upstream's multi-stage Dockerfile built from your repository, a managed Postgres (or MongoDB) addon linked through a secret group, and an optional volume for uploads. You end with the admin panel on a public HTTPS URL and the API answering, verified with `curl`. A human links your Git account once and creates the first admin user. Every step is a command a developer or a coding agent can run as-is.

## TL;DR: how to deploy Payload CMS

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a Payload app and which database adapter it uses, and confirm the team, repository, branch and project before changing anything. Give Northflank access to the repository (once).
3. Set `output: 'standalone'`, add upstream's multi-stage Dockerfile built with `next build --experimental-build-mode compile`, and for Postgres commit migrations passed as `prodMigrations`. Push.
4. Create a project, a Postgres (or MongoDB) addon and the service, then link the database as `DATABASE_URL` and `DATABASE_URI` plus a random `PAYLOAD_SECRET` with `northflank put secret`.
5. Wait for `SUCCESS`/`COMPLETED`, then create the first admin user in the browser.
6. Check that the live build is your pushed commit, `/admin` loads, and the API reports the user you created, which proves it reads the database.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy Payload as a container?

Payload 3 is a Next.js app, so it runs well as a long-lived Node.js server: the admin panel, REST and GraphQL APIs and your frontend all come from one process on port 3000. Running it as a container gives you somewhere to run migrations at startup and to keep a database connection pool, which upstream calls out as a fit for `prodMigrations` on long-running servers.

That shapes the hosting options. Payload needs a Node.js runtime, a PostgreSQL or MongoDB database, a place to run migrations and storage for uploads. Serverless hosts can run Next.js, but there migrations move into the build or CI and uploads need object storage. A raw VM or Kubernetes cluster works if you manage the server, TLS and database yourself. A container platform runs the same Docker image as a long-lived server next to a managed database.

On Northflank, the pieces are a database **addon**, a **secret group** that injects the connection string and `PAYLOAD_SECRET` at runtime, and a **combined service** that builds your repository's Dockerfile and runs it. If you'd rather click, there's a one-click [Payload stack template](https://northflank.com/stacks/deploy-payload) that builds your repository with a MongoDB addon and a secret group.

## Prerequisites

- **A Payload app that builds locally** without a database connection:
  ```bash
  npm ci && npx next build --experimental-build-mode compile && echo BUILD_OK
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
jq -r '.dependencies | "payload: \(.payload // "missing")  next: \(.next // "missing")  postgres: \(."@payloadcms/db-postgres" // "-")  mongodb: \(."@payloadcms/db-mongodb" // "-")"' package.json
grep -n "process.env" src/payload.config.ts
ls package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId payload -o json >/dev/null 2>&1 \
  && echo "project: payload exists and will be reused" \
  || echo "project: payload will be created"
```

- If `payload` or `next` says `missing`, this isn't a Payload 3 app. Stop and use a guide for your framework.
- The adapter decides the database: `@payloadcms/db-postgres` (`postgresAdapter`) gets a PostgreSQL addon, `@payloadcms/db-mongodb` (`mongooseAdapter`) a MongoDB addon.
- Current templates read `PAYLOAD_SECRET` and `DATABASE_URL`; older projects read `DATABASE_URI`. This guide sets both names, so either works.
- The lockfile tells you the package manager; upstream's Dockerfile in step 5 handles npm, pnpm and yarn.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "payload", "region": "europe-west" }'
export PROJECT_ID=payload

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Make the app container-ready

**Standalone output.** Add `output: 'standalone'` to the Next.js config (keep the `withPayload` wrapper your project already has).

**Dockerfile.** This is upstream's multi-stage Dockerfile with one change: the build runs `next build --experimental-build-mode compile`. Northflank builds run on separate build infrastructure without access to your private database, and upstream's "building without a DB connection" guide says compile mode builds without static generation, so pages that call the Local API render at request time instead.

```dockerfile
FROM node:24-alpine AS base

FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json yarn.lock* package-lock.json* pnpm-lock.yaml* ./
RUN \
  if [ -f yarn.lock ]; then yarn --frozen-lockfile; \
  elif [ -f package-lock.json ]; then npm ci; \
  elif [ -f pnpm-lock.yaml ]; then corepack enable pnpm && pnpm i --frozen-lockfile; \
  else echo "Lockfile not found." && exit 1; \
  fi

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npx next build --experimental-build-mode compile

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1
RUN addgroup --system --gid 1001 nodejs && adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
RUN mkdir .next && chown nextjs:nodejs .next
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
ENV PORT=3000 HOSTNAME=0.0.0.0
CMD ["node", "server.js"]
```

`HOSTNAME=0.0.0.0` makes the Next.js server listen on all interfaces, not just localhost. If your project uses `NEXT_PUBLIC_*` variables on the client, upstream's guide says to use `--experimental-build-mode generate-env` instead of `compile`.

**Migrations (Postgres only).** In development, the Postgres adapter pushes schema changes automatically; in production you need migrations. Generate them locally against a development database and commit `src/migrations`:

```bash
npm run payload migrate:create initial
```

Then run them on server start with `prodMigrations`, which upstream recommends for long-running servers:

```ts
import { migrations } from './migrations'
// …
db: postgresAdapter({
  pool: { connectionString: process.env.DATABASE_URL || '' },
  prodMigrations: migrations,
}),
```

MongoDB projects rarely need migrations; skip this unless you're reshaping existing documents.

Make sure `.dockerignore` excludes `node_modules`, `.next` and `.env*`.

Northflank builds what's on the remote branch, not your local working tree, so commit and push these changes before you create the service:

```bash
git add Dockerfile .dockerignore next.config.* src/
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create the database

PostgreSQL:

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "payload-db", "type": "postgresql", "version": "17-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-50", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "payload" }
}' | jq '(.data // .) | {id, status}'
```

For MongoDB, use `"type": "mongodb"` and `"version": "8.0-latest"` (check `curl -s https://api.northflank.com/v1/addon-types | jq '.data.addonTypes[] | select(.type=="mongodb") | .majors'`). The addon is private to the project, so TLS stays off here; it can be enabled later. `dbName` is permanent and storage can only grow.

Poll until running:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get addon --projectId "$PROJECT_ID" --addonId payload-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "status=$S"
  case "$S" in running) OK=1; break ;; failed|error|errorAllocating|deleting|deleted) echo FAILED; break ;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and read the addon logs"; false; }
```

Don't continue unless the addon reached `running`.

## Step 7: Create the service

Save as `payload.json`:

```json
{
  "name": "payload",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 3000, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/api/users/me", "port": 3000,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 12, "successThreshold": 1 }
  ]
}
```

`/api/users/me` is Payload's REST "current user" route on the default `users` collection; it should answer 200 without a session once Payload has connected to the database (not confirmed in the upstream pages fetched; if it doesn't, switch the probe to `/admin`). Change the path if your auth collection has a different slug. Use the same repository URL, provider and branch as in step 4. A Next.js build is memory-hungry. `billing.buildPlan` is optional; leave it out to use the default build plan, and only set it if builds run out of memory, using a build plan your team has access to (an unavailable one fails with `Build plan not found`).

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file payload.json | jq '(.data // .) | {id}'
```

The first build starts automatically; the first rollout will fail until step 8 provides the database. REST: `PUT https://api.northflank.com/v1/projects/$PROJECT_ID/services/combined`.

## Step 8: Link the database and `PAYLOAD_SECRET`

```bash
jq -n --arg s "$(openssl rand -hex 32)" '{
  name: "payload-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "payload", type: "service" } ] },
  addonDependencies: [ { addonId: "payload-db",
    keys: [ { keyName: "POSTGRES_URI", aliases: ["DATABASE_URL", "DATABASE_URI"] } ] } ],
  secrets: { variables: { PAYLOAD_SECRET: $s } }
}' > payload-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file payload-env.json >/dev/null && echo SECRET_OK
rm payload-env.json
northflank get addon credentials --projectId "$PROJECT_ID" --addonId payload-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
northflank get secret-details --projectId "$PROJECT_ID" --secretId payload-env --noDefaults -o json \
  | jq '(.data // .).addonSecrets[] | {id, variables: (.variables | keys)}'
northflank restart service --projectId "$PROJECT_ID" --serviceId payload --noDefaults -o json >/dev/null
```

For MongoDB, replace `POSTGRES_URI` with the MongoDB connection-string key (the MongoDB docs call it the `mongoSrv` connection string; use the exact key name the credentials command prints). The two commands print key names only: the addon's available credential keys, then the variables the group inherits from the addon. `secretType: "environment"` injects at runtime only, which is all the compile-mode build needs. If the link is wrong, fix `addonDependencies` with `northflank patch secret --projectId "$PROJECT_ID" --secretId payload-env --input '{"addonDependencies": [...]}'`.

## Step 9: Wait for the build and rollout

```bash
OK=0
for i in $(seq 1 120); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId payload --noDefaults -o json)
  B=$(echo "$S" | jq -r '(.data // .).status.build.status // "NONE"'); D=$(echo "$S" | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "build=$B deployment=$D"
  case "$B" in FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo BUILD_FAILED; break;; esac
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  [ "$B" = "SUCCESS" ] && [ "$D" = "COMPLETED" ] && { OK=1; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId payload --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId payload --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

On build failure, read `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId payload --lineLimit 200`. Don't continue if the guard printed `ROLLOUT_NOT_COMPLETE`.

**This step needs a human:** open `https://$HOST/admin` and create the first user. Payload shows a create-first-user screen on an empty `users` collection, and whoever gets there first becomes the first user.

## Step 10 (optional): Keep uploads on a volume

If an upload collection stores files on local disk, they're lost on redeploy. Upstream says to use a persistent filesystem or a storage adapter. For one instance, attach a volume at the directory your collection writes to (in a standalone image the working directory is `/app`, so a `media` collection with the default `staticDir` is assumed to resolve to `/app/media`; check your config):

```bash
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "payload-media", "mounts": [ { "containerMountPath": "/app/media" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "payload", "type": "service" } ]
}'
```

A `ReadWriteOnce` volume limits the service to one instance. To scale out, use `@payloadcms/storage-s3` with `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` and `S3_REGION` instead (with an S3-compatible endpoint such as MinIO, also set the client `endpoint` and `forcePathStyle: true`).

## How to know it worked

You're done when all of these hold. Check 4 applies to Postgres only.

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId payload --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The admin panel loads:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/admin"     # 200
   ```
3. The API reads the database. After you created the first user in step 9, Payload reports the `users` collection as initialized, which it can only know from the database:
   ```bash
   curl -s "https://$HOST/api/users/init" | jq '.initialized'     # true
   ```
   Before the first user exists this prints `false`. An error or HTML instead of JSON means Payload couldn't reach the database: read the logs.
4. The start-up log shows migrations ran and no connection errors:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId payload --noDefaults --lineLimit 300 --textIncludes "igrat"
   ```
5. The variables are present (names only, never values):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId payload \
     --shell-cmd 'sh -c' --cmd '"env | cut -d= -f1 | grep -E ^(PAYLOAD|DATABASE) | sort"'
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Build fails trying to connect to the database | Build still doing static generation | Use `--experimental-build-mode compile`, or `export const dynamic = 'force-dynamic'` in route segments |
| Build fails on a missing `PAYLOAD_SECRET` | Config read at build time | Add a build-only placeholder via `buildArguments` (`northflank patch service combined`); never the real secret |
| `relation "users" does not exist` | No migrations in production | Create migrations and set `prodMigrations` (step 5) |
| Admin works locally, `ECONNREFUSED 127.0.0.1` in prod | Config reads a variable that isn't set | Check the variable name in `payload.config.ts`; both `DATABASE_URL` and `DATABASE_URI` are set |
| `Cannot find module` at start | Standalone output missing a file | Confirm `output: 'standalone'`; check `.dockerignore` |
| Uploaded media disappears | Local disk without a volume | Step 10 or the S3 adapter |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print `PAYLOAD_SECRET` or the database URI. Changing `PAYLOAD_SECRET` later invalidates existing login sessions, so generate it once and keep it.
- Don't let dev-mode schema push touch the production database. Don't point a local `next dev` at it.
- Back up the database before deploying a change that adds migrations.
- Don't delete the addon, the volume or the service without explicit, per-resource confirmation.

## FAQ

### PostgreSQL or MongoDB?
Both are first-party adapters. Postgres needs migrations for schema changes; MongoDB rarely does. The Northflank stack template uses MongoDB; this guide defaults to Postgres.

### Why set both `DATABASE_URL` and `DATABASE_URI`?
Current Payload templates read `DATABASE_URL`, and older projects read `DATABASE_URI`. One secret-group link with two aliases covers both.

### Can I run migrations in CI instead of at startup?
Yes. Upstream also documents a `"ci": "payload migrate && pnpm build"` script, but that needs a database connection during the build, which a private addon doesn't offer. `prodMigrations` avoids that.

### Will pushes redeploy automatically?
Yes. A combined service builds new commits on the configured branch and rolls them out.

### Where should I deploy my Payload CMS?
Somewhere that runs Payload's Next.js server alongside a managed PostgreSQL or MongoDB database. A long-running container is the simplest fit, because migrations can run on start and the database connection pool stays open. Serverless hosts also work but push migrations into CI and uploads into object storage, and a VM works if you're happy to manage the server yourself.

## Conclusion

You now have Payload 3 built from your repository, connected to a managed database through linked secrets, with migrations applied at startup and the admin panel on a public HTTPS URL.

**Next:** add a custom domain, schedule database backups, and move uploads to S3-compatible storage before scaling out. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Next.js app](deploy-nextjs-app.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy MongoDB](deploy-mongodb.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
