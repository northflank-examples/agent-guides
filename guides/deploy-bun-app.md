<!--
agent-guide-meta:
  id: NF-STK-BUN
  description: "How to deploy a Bun app (Bun.serve, Elysia or Hono) with an oven/bun Docker image, a managed PostgreSQL DATABASE_URL and a health check on a public HTTPS URL."
  sources:
    - https://northflank.com/stacks/deploy-bun
    - https://bun.com/docs/guides/ecosystem/docker
    - https://bun.com/docs/runtime/http/server
    - https://bun.com/docs/runtime/sql
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
    - https://northflank.com/docs/v1/api/project/jobs/run-job
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/release/run-migrations
  related: [NF-DB-001, NF-DEP-001, NF-STK-DENO, NF-JOB-002, NF-DB-003]
-->

# How to deploy a Bun app

To deploy a Bun app, make the server (`Bun.serve`, Elysia or Hono) listen on `PORT` and `0.0.0.0` with a `/health` route, package it in a Docker image built from the official `oven/bun` base with `bun install --frozen-lockfile --production`, and inject a PostgreSQL `DATABASE_URL` at runtime if it needs a database. Bun runs TypeScript directly, so there is usually no build step.

This guide does it on Northflank with the CLI: a project, an optional managed Postgres database linked as `DATABASE_URL`, and the app deployed from Git with a readiness check on `/health`. You end with a public HTTPS URL whose `/health` returns 200, proving the database connection too, checked with `curl`. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a Bun app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Bun app and confirm the team, repository, branch and project before changing anything, then give Northflank access to the repository (once).
3. Listen on `process.env.PORT` and `0.0.0.0`, add `GET /health`, and add a `Dockerfile` built from `oven/bun:1` with `bun install --frozen-lockfile --production`. Push.
4. Create a project and, if needed, a PostgreSQL addon linked as `DATABASE_URL`, then create the service with `northflank put service combined --file bun-app.json`.
5. Wait for the deploy, then check that the live build is your pushed commit and that `https://$HOST/health` returns 200.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your Bun app on Northflank

A Bun server needs a host that runs the Bun runtime itself, keeps a long-running process up on a port, and often provides a database. Static hosts don't run servers, and serverless platforms only run Bun if they support that runtime. A container platform or a VM runs whatever runtime your image contains; a VM or your own Kubernetes cluster leaves the OS, TLS and restarts to you, while a container platform builds your Dockerfile and handles those.

Bun runs TypeScript directly, so there's usually no build step and no `dist` folder. Many hosts assume Node.js, though, and either don't offer Bun or hide the version they run. A container built from the official `oven/bun` image pins the runtime you test with locally.

On Northflank, the image is built and run by one **combined service** that rebuilds on every push. A PostgreSQL **addon** and a **secret group** supply `DATABASE_URL`. If you'd rather click through the UI, there's a one-click [Bun stack template](https://northflank.com/stacks/deploy-bun).

## Prerequisites

- **A Bun app that runs locally:**
  ```bash
  bun install --frozen-lockfile && bun run src/index.ts
  ```
  Stop it with Ctrl+C once it's serving. This guide uses `src/index.ts` as the entry file; use yours if it differs. `--frozen-lockfile` fails without a committed `bun.lock` (or `bun.lockb` in older projects).
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
jq -r '"name: \(.name // "-")  start: \(.scripts.start // "-")  frameworks: \([(.dependencies // {}) | keys[] | select(. == "elysia" or . == "hono")] | join(",") | if . == "" then "Bun.serve" else . end)"' package.json
ls bun.lock bun.lockb package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null
ls src/index.ts 2>/dev/null || echo "entry: src/index.ts not found, find your entry file"

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId bun-app -o json >/dev/null 2>&1 \
  && echo "project: bun-app exists and will be reused" \
  || echo "project: bun-app will be created"
```

- If there's no `bun.lock` or `bun.lockb`, this isn't set up as a Bun app. Run `bun install` and commit the lockfile, or stop and use a guide for your runtime (a `package-lock.json` alone means Node.js).
- If the entry file isn't `src/index.ts`, change the `ENTRYPOINT` in step 6 to match.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "bun-app", "region": "europe-west" }'
export PROJECT_ID=bun-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Make the server container-ready

`Bun.serve` defaults to port 3000 (or `BUN_PORT`, `PORT`, `NODE_PORT` if set) and hostname `0.0.0.0` ([Bun HTTP server docs](https://bun.com/docs/runtime/http/server)). Being explicit avoids surprises with frameworks that change those defaults:

```ts
// Bun.serve
import { sql } from "bun";
Bun.serve({
  port: Number(process.env.PORT ?? 3000),
  hostname: "0.0.0.0",
  routes: {
    "/health": async () => {
      if (process.env.DATABASE_URL) await sql`SELECT 1`;
      return new Response("ok");
    },
    "/": () => new Response("Hello from Bun"),
  },
});
```

```ts
// Elysia
new Elysia().get("/health", () => "ok")
  .listen({ port: Number(process.env.PORT ?? 3000), hostname: "0.0.0.0" });

// Hono: Bun serves the default export
export default { port: Number(process.env.PORT ?? 3000), fetch: app.fetch };
```

The `sql` client in `bun` reads `DATABASE_URL` first, then `POSTGRES_URL` and similar names ([Bun SQL docs](https://bun.com/docs/runtime/sql)). Drizzle, Prisma and `postgres` also take `process.env.DATABASE_URL`. If you use the database, make `/health` run `SELECT 1` so a probe failure means a real problem.

## Step 6: Write the Dockerfile

This is a condensed version of the [Bun Docker guide](https://bun.com/docs/guides/ecosystem/docker). Dependencies install in their own stage, so source changes don't reinstall packages:

```dockerfile
FROM oven/bun:1 AS base
WORKDIR /usr/src/app

FROM base AS install
COPY package.json bun.lock* bun.lockb* ./
RUN bun install --frozen-lockfile --production

FROM base AS release
COPY --from=install /usr/src/app/node_modules node_modules
COPY . .
ENV NODE_ENV=production
USER bun
EXPOSE 3000/tcp
ENTRYPOINT ["bun", "run", "src/index.ts"]
```

Add a `.dockerignore` with `node_modules` and `.git`. If the app has a build step (a bundled frontend, for example), install all dependencies in a separate stage, run `bun run build` there, and copy the output into `release`. Pin `oven/bun:1.x.y` to the version in your `bun --version` for reproducible builds.

Northflank builds what's on the remote branch, not your local working tree, so commit and push the Dockerfile and app changes before you create the service:

```bash
git add Dockerfile .dockerignore src/ package.json bun.lock*
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Create PostgreSQL and link `DATABASE_URL` (optional)

Skip this step if the app has no database.

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "bun-db",
  "type": "postgresql",
  "version": "17-latest",
  "tlsEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId bun-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and read the addon logs"; false; }

northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "bun-env",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": false },
  "addonDependencies": [
    { "addonId": "bun-db", "keys": [ { "keyName": "POSTGRES_URI", "aliases": ["DATABASE_URL"] } ] }
  ]
}'
```

Don't create the secret group unless the addon reached `running`. `POSTGRES_URI` has the form `postgresql://user:password@host:port/database`. It's injected at runtime only, not into the build. `tlsEnabled: false` (the API default) matches Bun's default `PGSSLMODE` of `disable`; the database stays on the private network either way. If you enable TLS, set `PGSSLMODE=require` or pass `tls: true` to your client. Storage is in MB and can only be increased later.

## Step 8: Create the service

Write `bun-app.json`:

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 3000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "3000" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/health", "port": 3000,
      "initialDelaySeconds": 3, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file bun-app.json
```

Use the same repository URL, provider and branch as in step 4. The first build starts automatically. If none appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json --input '{}'`.

## Step 9: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds, staying under the API limit of 1,000 requests per hour, and prints your URL when the app is live. `COMPLETED` means rolled out and serving.

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web --lineLimit 200`.

### Running migrations

If you use a migration tool (for example `bunx drizzle-kit migrate`), run it as a manual job built from the same repo rather than at server start. Create it with `northflank put job` (which makes a manual job when `settings.cron` is absent; `put job manual` is deprecated in the API) using the same `vcsData` values under `deployment.vcs`, `"settings": { "backoffLimit": 0 }` and `"docker": { "configType": "customCommand", "customCommand": "bunx drizzle-kit migrate" }`. Then start it with `northflank start job run --projectId "$PROJECT_ID" --jobId <JOB> --noDefaults -o json --input '{}'` and poll `northflank get job run` until `concluded` is `true`. Migration tools are often dev dependencies, so make sure the one you call survives `--production`. See [run migrations](https://northflank.com/docs/v1/application/release/run-migrations).

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health endpoint returns 200. With a database, `/health` runs `SELECT 1`, so this also proves the connection works:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/health"
   ```
3. The runtime log shows the server started, with no `ECONNREFUSED` or authentication errors:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId web --lineLimit 100
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Build fails at `bun install --frozen-lockfile` | Lockfile missing or out of date | Run `bun install` locally and commit the lockfile |
| `Cannot find module` at startup | Package is a dev dependency, removed by `--production` | Move it to `dependencies` |
| 502/503 while `COMPLETED` | Server on another port, or bound to `127.0.0.1` | Read `PORT`; set `hostname: "0.0.0.0"` |
| Readiness probe fails | `/health` missing, or DB query failing | Add the route; check `DATABASE_URL` is set |
| `ECONNREFUSED 127.0.0.1:5432` | No `DATABASE_URL`, so the client used local defaults | Re-run step 7, then `northflank restart service` |
| `EACCES` writing files | Image runs as `USER bun` | Write to `/tmp`, or use a volume for persistent data |
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
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print `DATABASE_URL` or other secret values. Read env var names, not values.
- Don't put secrets in `runtimeEnvironment` in a committed file; add them to the secret group.
- Storage can only be increased. Take a backup before migrations that change data.
- Don't delete the addon or service without explicit, per-resource confirmation. Prefer pausing.

## FAQ

### Where should I deploy my Bun app?
On a host that runs a long-running process with the Bun runtime, which in practice means a container platform, a VM or Kubernetes. Building from the official `oven/bun` image in a `Dockerfile` pins the Bun version and makes the app portable across any container host. A container platform adds builds from Git, health checks, TLS and a managed database without managing servers.

### Does this work for Elysia and Hono?
Yes. The deployment only needs the app to listen on `PORT` and `0.0.0.0`, answer `GET /health`, and start from the `ENTRYPOINT`. Step 5 shows the listen call for each.

### Should I compile to a single binary with `bun build --compile`?
You can. Build the binary in one stage and copy it into a slim runtime image. The image above is simpler and matches Bun's own Docker guide.

### Is `DATABASE_URL` available at build time?
No. `secretType: "environment"` injects it at runtime only. Avoid database access during `docker build`.

### Is there a one-click option?
Yes, the [Bun stack template](https://northflank.com/stacks/deploy-bun) deploys a sample app from the UI. This guide does the same for your own repo, scripted.

## Conclusion

You now have a Bun server built from Git into a pinned `oven/bun` image, listening on port 3000 behind a public HTTPS URL, with an optional managed Postgres database wired in as `DATABASE_URL` and a health check that tests it.

**Next:** add Redis for caching or queues, move slow work into a background worker, and set up database backups before real data arrives. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy a Deno app](deploy-deno-app.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
