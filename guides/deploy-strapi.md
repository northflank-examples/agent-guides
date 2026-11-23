<!--
agent-guide-meta:
  id: NF-STK-STRAPI
  description: "How to deploy Strapi to production with Docker, managed PostgreSQL, stable secrets in a secret group and persistent uploads, with every step scripted."
  sources:
    - https://northflank.com/stacks/deploy-strapi
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://docs.strapi.io/cms/configurations/environment
    - https://docs.strapi.io/cms/configurations/database
    - https://docs.strapi.io/cms/configurations/server
    - https://docs.strapi.io/cms/installation/docker
    - https://docs.strapi.io/cms/configurations/media-library-providers/amazon-s3
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DB-001, NF-DEP-001, NF-DEP-004]
-->

# How to deploy Strapi

To deploy Strapi to production, build the admin panel in a multi-stage Docker image, switch from SQLite to PostgreSQL with `DATABASE_CLIENT=postgres` and `DATABASE_URL`, generate `APP_KEYS`, `ADMIN_JWT_SECRET` and the other secrets once and keep them fixed, serve on port 1337, and store uploads on a persistent volume or S3. Otherwise a redeploy can wipe the database, log everyone out and lose media.

This guide does it on Northflank with the CLI: Strapi's production `Dockerfile` built from Git, a managed PostgreSQL addon and generated secrets in one secret group, a readiness check on `/_health`, and a volume at `public/uploads`. You end with a public HTTPS URL where `/_health` returns 204 and the admin panel loads. A human logs in once, shares the repository with Northflank and creates the first admin user. Every step is a command a developer or a coding agent can run as-is.

## TL;DR: how to deploy Strapi

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Strapi v5 project with the `pg` driver, and confirm the team, repository, branch and project before changing anything.
3. Give Northflank access to your repository on the Git integrations page (once), then add the multi-stage `Dockerfile` below and push.
4. Create PostgreSQL and a secret group that links `POSTGRES_URI` as `DATABASE_URL`, with `DATABASE_CLIENT=postgres` and generated `APP_KEYS`, `API_TOKEN_SALT`, `ADMIN_JWT_SECRET`, `TRANSFER_TOKEN_SALT`, `JWT_SECRET` and `ENCRYPTION_KEY`.
5. Build and run the repo with `northflank put service combined` on port 1337, with a readiness check on `/_health`, and attach a volume at `/opt/app/public/uploads`.
6. Wait for the deploy, check that the live build is your pushed commit and `/_health` returns 204, then create the first admin at `/admin`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

A new Strapi project defaults to SQLite and keeps its secrets in a local `.env` file that shouldn't be committed. In a container, both disappear on every redeploy. The Strapi docs list the variables that replace them: `DATABASE_CLIENT` and the database settings, `APP_KEYS`, `API_TOKEN_SALT`, `ADMIN_JWT_SECRET`, `TRANSFER_TOKEN_SALT`, `JWT_SECRET`, plus `ENCRYPTION_KEY` in the generated v5 `.env`. Generate them once and store them in a secret group, so every build and restart uses the same values.

Strapi's local upload provider writes files to `public/uploads` inside the app folder. That needs a volume, or an S3-compatible bucket if you want more than one instance.

So where to deploy Strapi? It needs a long-running Node.js server, a build step for the admin panel, a SQL database and somewhere to keep uploads. Static and serverless hosts are built for short-lived functions and don't keep a Strapi server or local files around. A VPS gives full control, but you set up Node.js, the process manager, TLS and database backups yourself. A container platform builds the repo from a Dockerfile, runs it next to a managed database and handles HTTPS.

If you'd rather click through it, Northflank has a [Strapi with PostgreSQL stack template](https://northflank.com/stacks/deploy-strapi) that creates the repo, a combined service, a volume, a PostgreSQL addon and a secret group from the UI.

## Prerequisites

- **A Strapi v5 app that builds locally:**
  ```bash
  npm ci && npm run build && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **`openssl`** (to generate secrets).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

Names used below: project `strapi`, addon `strapi-db`, secret group `strapi-env`, service `strapi`, volume `strapi-uploads`.

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
jq -r '"strapi: \(.dependencies["@strapi/strapi"] // "missing")  pg: \(.dependencies.pg // "missing")"' package.json
grep -n "DATABASE_URL" config/database.* || echo "config/database: no DATABASE_URL"
ls package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId strapi -o json >/dev/null 2>&1 \
  && echo "project: strapi exists and will be reused" \
  || echo "project: strapi will be created"
```

- If `strapi` says `missing`, this isn't a Strapi app. Stop and use a guide for your framework. If it isn't a `5.x` version, upgrade first: this guide targets Strapi v5.
- If `pg` says `missing`, run `npm install pg` and commit the lockfile in step 5.
- The generated `config/database` reads `DATABASE_URL` as `connectionString` for each client. If the grep finds nothing, add it as in the Strapi database docs.
- The lockfile tells you the package manager. Step 5 assumes `package-lock.json`; for pnpm or yarn, swap the install commands.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "strapi", "region": "europe-west", "description": "Strapi CMS" }'
export PROJECT_ID=strapi

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Add the `Dockerfile` and push

If step 2 found no `Dockerfile`, add this one at the repository root. It's the Strapi docs' `Dockerfile.prod` with the health check left to the platform:

```dockerfile
FROM node:22-alpine AS build
RUN apk update && apk add --no-cache build-base gcc autoconf automake zlib-dev libpng-dev bash vips-dev git > /dev/null 2>&1
WORKDIR /opt/
COPY package.json package-lock.json ./
RUN npm install -g node-gyp
RUN npm config set fetch-retry-maxtimeout 600000 -g && npm ci
ENV PATH=/opt/node_modules/.bin:$PATH
WORKDIR /opt/app
COPY . .
ENV NODE_ENV=production
RUN npm run build

FROM node:22-alpine
RUN apk add --no-cache vips-dev
ENV NODE_ENV=production
WORKDIR /opt/
COPY --from=build /opt/package.json /opt/package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force
ENV PATH=/opt/node_modules/.bin:$PATH
WORKDIR /opt/app
COPY --from=build /opt/app ./
RUN chown -R node:node /opt/app
USER node
EXPOSE 1337
CMD ["npm", "run", "start"]
```

Add a `.dockerignore` with `node_modules/`, `.tmp/`, `.cache/`, `.git/`, `build/` and `.env`. Don't set `NODE_ENV=production` before `npm ci`; the docs warn it skips the devDependencies the admin build needs.

Northflank builds what's in your repository, not the files on your machine, so commit and push them:

```bash
git add Dockerfile .dockerignore package.json package-lock.json config/
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create PostgreSQL and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "strapi-db", "type": "postgresql", "version": "16-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "strapi" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId strapi-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and read the addon logs"; false; }
```

Don't continue unless the addon reached `running`. The database is only reachable inside the project, so TLS is off to match Strapi's default `DATABASE_SSL=false`. To use TLS, set `tlsEnabled: true` and `DATABASE_SSL=true`. `dbName` and TLS are fixed at creation, and storage can only be increased.

## Step 7: Create the secret group

Generate every secret locally and pipe it straight into the group, never echoing it:

```bash
r() { openssl rand -base64 32; }
jq -n --arg k "$(r),$(r),$(r),$(r)" --arg a "$(r)" --arg b "$(r)" --arg c "$(r)" --arg d "$(r)" --arg e "$(r)" '{
  name: "strapi-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "strapi-db", keys: [ { keyName: "POSTGRES_URI", aliases: ["DATABASE_URL"] } ] } ],
  secrets: { variables: {
    DATABASE_CLIENT: "postgres",
    APP_KEYS: $k, API_TOKEN_SALT: $a, ADMIN_JWT_SECRET: $b,
    TRANSFER_TOKEN_SALT: $c, JWT_SECRET: $d, ENCRYPTION_KEY: $e
  } }
}' > strapi-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file strapi-env.json >/dev/null && rm strapi-env.json
```

`APP_KEYS` is comma-separated (the server config reads it with `env.array`). `secretType: "environment"` injects these values at runtime only, which is what Strapi needs: `npm run build` compiles the admin panel and doesn't need database access or secrets.

## Step 8: Create the Strapi service

Write `strapi.json`:

```json
{
  "name": "strapi",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 1337, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "HOST": "0.0.0.0", "PORT": "1337" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/_health", "port": 1337,
      "initialDelaySeconds": 20, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 12, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file strapi.json
```

Use the same repository URL, provider and branch as in step 4. The admin panel build is memory-hungry. `billing.buildPlan` is optional; leave it out to use the default build plan, and only set it if builds run out of memory, using a build plan your team has access to (an unavailable one fails with `Build plan not found`). `/_health` is the endpoint the Strapi Docker docs use for their container health check. The first start creates the schema, so readiness allows about two minutes. The first build starts automatically.

## Step 9: Persist uploads

**Option A: volume (one instance).**

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId strapi-uploads --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "strapi-uploads",
  "mounts": [ { "containerMountPath": "/opt/app/public/uploads" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "strapi", "type": "service" } ]
}'
```

Attaching redeploys the service. A single read/write volume limits it to one instance, and the old container stops before the new one starts.

**Option B: S3-compatible storage (any number of instances).** Install `@strapi/provider-upload-aws-s3`, configure it in `config/plugins` as the Strapi S3 provider docs describe, and add the bucket credentials to `strapi-env`. It works with any S3-compatible service, including MinIO. Skip the volume.

## Step 10: Wait for the deploy and get your URL

The first build takes several minutes. This loop checks every 10 seconds and prints your URL when Strapi is live. You can also watch the build in the Northflank dashboard.

```bash
OK=0
for i in $(seq 1 120); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId strapi --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId strapi --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId strapi --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId strapi`. The troubleshooting table covers the common causes.

If your `config/server` sets `url: env('PUBLIC_URL')` (as in the Strapi server docs example), add `PUBLIC_URL=https://$HOST` to the complete `runtimeEnvironment` map with `northflank patch service combined`, then `northflank restart service`.

**This step needs a human:** open `https://$HOST/admin` right away and create the first admin user. Until someone does, anyone with the URL can.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId strapi --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health endpoint returns `204`:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/_health"
   ```
3. The admin panel is served (200):
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/admin"
   ```
4. The admin API answers with JSON. After you've created the first admin, this prints `false`; before that, `true`:
   ```bash
   curl -s "https://$HOST/admin/init" | jq '.data.hasAdmin | not'
   ```
5. Strapi connected to PostgreSQL, not SQLite (prints only the scheme; Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId strapi \
     --shell-cmd 'sh -c' --cmd '"printenv DATABASE_CLIENT; printenv DATABASE_URL | cut -c1-13; ls -ld /opt/app/public/uploads"'
   ```
   Expect `postgres`, `postgresql://` and an uploads folder owned by `node`.
6. After uploading an image in the Media Library, it's still served after `northflank restart service --projectId "$PROJECT_ID" --serviceId strapi`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Build killed or out of memory in `npm run build` | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |
| Admin bundle blank or broken | `NODE_ENV=production` set before `npm ci` | Use the Dockerfile in step 5 |
| `Cannot find module 'pg'` | Driver not installed | `npm install pg`, commit the lockfile |
| Logs show `.tmp/data.db` (SQLite) | `DATABASE_CLIENT` not injected | Check the secret group; restart |
| `Missing admin.auth.secret` or `app.keys` errors | Secrets missing | Re-run step 7; restart |
| `EACCES` writing uploads | Volume not writable by `node` | See [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) for ownership fixes |
| Images 404 after redeploy | No volume and local provider | Do step 9 |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 10 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print or commit the generated secrets or `DATABASE_URL`. Don't commit `.env`.
- Don't rotate `APP_KEYS`, `ADMIN_JWT_SECRET` or `ENCRYPTION_KEY` on a live instance without asking; it invalidates sessions and stored tokens.
- Content-type changes are made in development and shipped through Git; the Content-Type Builder doesn't edit models in production.
- Get explicit confirmation before deleting the addon, volume or service. Back up the database before Strapi upgrades.

## FAQ

### Can I deploy Strapi without a Dockerfile?
Yes, but this guide uses one because Strapi needs `sharp`'s native dependencies and a full admin build. The Dockerfile from Strapi's own docs handles both.

### Do I need `PUBLIC_URL`?
Only if your server config uses it, for example for absolute upload URLs or emails. The admin panel calls the origin it's served from by default.

### Can I run two instances?
Yes, with option B (S3 uploads) instead of a volume.

### Where should I deploy my Strapi project?
On a host that runs Strapi as an always-on Node.js server with a managed SQL database and persistent upload storage (a volume or S3). That rules out static and serverless hosts for the Strapi server itself. A VPS works if you'll manage the server; a container platform that builds from your Dockerfile, as in this guide, handles the build, TLS and database.

## Conclusion

You now have Strapi v5 built from your repository with a production Dockerfile, connected to managed PostgreSQL, with stable secrets and persistent uploads behind a public HTTPS URL.

**Next:** [link a custom domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port), and move uploads to object storage when you need more than one instance. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy a Next.js app](deploy-nextjs-app.md)
