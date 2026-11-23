<!--
agent-guide-meta:
  id: NF-STK-DENO
  description: "How to deploy a Deno app (Deno.serve, Fresh or Oak) in a denoland/deno Docker image with explicit permission flags, optional PostgreSQL and a health check."
  sources:
    - https://northflank.com/stacks/deploy-deno
    - https://docs.deno.com/runtime/reference/docker/
    - https://docs.deno.com/runtime/fundamentals/http_server/
    - https://docs.deno.com/runtime/fundamentals/security/
    - https://usefresh.dev/docs/deployment/docker
    - https://northflank.com/docs/v1/api/use-the-cli
    - https://northflank.com/docs/v1/api/team/projects/create-or-update-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DB-001, NF-DEP-001, NF-STK-BUN, NF-DB-003]
-->

# How to deploy a Deno app

To deploy a Deno app, make the server (`Deno.serve`, Fresh or Oak) listen on `PORT` and `0.0.0.0` with a `/health` route, build a Docker image from the official `denoland/deno` image that caches dependencies with `deno install`, and write explicit permission flags such as `--allow-net` and `--allow-env` into the `CMD`. A container has no terminal to answer Deno's permission prompt, so a missing flag crashes the app.

This guide does it on Northflank with the CLI: a project, an optional managed Postgres database linked as `DATABASE_URL`, and the app deployed from Git with a readiness check on `/health`. You end with a public HTTPS URL whose `/health` returns 200, checked with `curl`, and no permission errors in the logs. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a Deno app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Deno app and confirm the team, repository, branch and project before changing anything, then give Northflank access to the repository (once).
3. Listen on `Deno.env.get("PORT")` and `0.0.0.0`, add `GET /health`, and add a `Dockerfile` from `denoland/deno` whose `CMD` grants only what the app uses (`--allow-net --allow-env --allow-read=.`; Fresh uses `deno serve -A _fresh/server.js`). Push.
4. Create a project, optionally Postgres with a secret group linking `DATABASE_URL`, and a service built from your repo with `northflank put service combined --file deno-app.json`.
5. Wait for the deploy, then check that the live build is your pushed commit and `https://$HOST/health` returns 200 with no permission errors in the logs.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your Deno app on Northflank

A Deno server needs a host that runs the Deno runtime, keeps a long-running process up on a port, and grants it the permissions it needs. Edge and serverless platforms built for Deno run it as short-lived handlers; a container platform or a VM runs the full runtime from an image you control, next to a database if you need one. A VM or your own Kubernetes cluster leaves the OS, TLS and restarts to you, while a container platform builds your Dockerfile and handles those.

### Why a container for Deno

A container lets you pin the Deno version, pre-download every dependency at build time, and write the permission flags into the `CMD` where they're reviewed with the code. Nothing is fetched or granted at runtime that you didn't decide on.

On Northflank, one **combined service** builds the `Dockerfile` from Git on every push and runs it. A PostgreSQL **addon** and a **secret group** supply `DATABASE_URL`. If you'd rather click through the UI, there's a one-click [Deno stack template](https://northflank.com/stacks/deploy-deno).

## Prerequisites

- **A Deno app that type-checks and installs its dependencies locally.** This guide uses `main.ts` as the entry file:
  ```bash
  deno install --entrypoint main.ts && deno check main.ts && echo CHECK_OK
  ```
  For Fresh, run `deno task build` instead.
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
ls deno.json deno.jsonc deno.lock main.ts Dockerfile 2>/dev/null
grep -qsE '"fresh|@fresh/' deno.json deno.jsonc && echo "framework: Fresh"
deno --version | head -n1

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId deno-app -o json >/dev/null 2>&1 \
  && echo "project: deno-app exists and will be reused" \
  || echo "project: deno-app will be created"
```

- If there's no `deno.json`, `deno.jsonc` or `deno.lock`, check that this is a Deno app; if it has a `package.json` and runs on Node.js or Bun, stop and use a guide for that runtime.
- If your entry file isn't `main.ts`, use its name wherever `main.ts` appears below. If the `grep` printed `framework: Fresh`, use the Fresh variants in steps 5 and 6.
- Note the Deno version: step 6 pins the image to it.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "deno-app", "region": "europe-west" }'
export PROJECT_ID=deno-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Make the server container-ready

`Deno.serve` listens on port 8000 by default ([Deno HTTP server docs](https://docs.deno.com/runtime/fundamentals/http_server/)). Read `PORT` and bind to all interfaces explicitly:

```ts
// Deno.serve
Deno.serve(
  { port: Number(Deno.env.get("PORT") ?? 8000), hostname: "0.0.0.0" },
  (req) => {
    const { pathname } = new URL(req.url);
    if (pathname === "/health") return new Response("ok");
    return new Response("Hello from Deno");
  },
);
```

```ts
// Oak
const app = new Application();
// ...routes, including GET /health
await app.listen({ port: Number(Deno.env.get("PORT") ?? 8000), hostname: "0.0.0.0" });
```

**Fresh** builds to `_fresh/` and is started with `deno serve`, which listens on port 8000 by default. Keep port 8000 so no extra flags are needed, and add a `/health` route.

If you use Postgres, read `Deno.env.get("DATABASE_URL")`, for example with `import postgres from "npm:postgres"; const sql = postgres(Deno.env.get("DATABASE_URL")!);`, and make `/health` run `SELECT 1`.

## Step 6: Write the Dockerfile with permission flags

For `Deno.serve` or Oak:

```dockerfile
FROM denoland/deno:latest
RUN mkdir /app && chown deno:deno /app
WORKDIR /app
USER deno
COPY --chown=deno:deno . .
RUN deno install --entrypoint main.ts
EXPOSE 8000
CMD ["deno", "run", "--allow-net", "--allow-env", "--allow-read=.", "main.ts"]
```

For Fresh, following the [Fresh Docker guide](https://usefresh.dev/docs/deployment/docker):

```dockerfile
FROM denoland/deno:latest
ARG GIT_REVISION
ENV DENO_DEPLOYMENT_ID=${GIT_REVISION}
WORKDIR /app
COPY . .
RUN deno install --allow-scripts
RUN deno task build
EXPOSE 8000
CMD ["deno", "serve", "-A", "_fresh/server.js"]
```

Fresh uses `DENO_DEPLOYMENT_ID` for asset caching, and the Fresh docs say it must change whenever the project's files change; pass the commit SHA as the `GIT_REVISION` build argument (in the service's `buildArguments`). Replace `latest` with the version tag that matches `deno --version`, so a new Deno release can't change your build. About the flags ([Deno security docs](https://docs.deno.com/runtime/fundamentals/security/)):

- `--allow-net` is needed to listen on the port and to reach the database. Scope it, as in `--allow-net=0.0.0.0:8000,<db-host>:5432`, only if you know every host the app calls.
- `--allow-env` lets the app read `PORT` and `DATABASE_URL`. You can scope it (`--allow-env=PORT,DATABASE_URL`), but npm drivers often read extra variables such as `PGHOST`, so start broad and narrow it once the app runs.
- `--allow-read=.` covers static files and templates in `/app`. Drop it if the app reads no files.
- `-A` grants everything. Fresh's docs use it; you can replace it with explicit flags once you know what the app needs.

When a permission is missing, Deno doesn't prompt inside a container. It throws an error naming the flag to add, and the container exits. Some npm packages also need `--allow-sys` or `--allow-read`; add the flag the error names. You can also declare permissions in `deno.json` instead of on the command line.

Northflank builds what's on the remote branch, not your local working tree, so commit and push the Dockerfile and app changes before you create the service. Add any other app files you changed, and use `deno.jsonc` if that's your config file:

```bash
git add Dockerfile deno.json deno.lock main.ts
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Create PostgreSQL and link `DATABASE_URL` (optional)

Skip this step if the app has no database.

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "deno-db",
  "type": "postgresql",
  "version": "17-latest",
  "tlsEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId deno-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and read the addon logs"; false; }

northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "deno-env",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": false },
  "addonDependencies": [
    { "addonId": "deno-db", "keys": [ { "keyName": "POSTGRES_URI", "aliases": ["DATABASE_URL"] } ] }
  ]
}'
```

Don't create the secret group unless the addon reached `running`. `POSTGRES_URI` has the form `postgresql://user:password@host:port/database`, and it's injected at runtime only. With `tlsEnabled: false` (the API default), drivers connect without TLS on the private network, which matches their defaults. If you enable TLS, turn on the driver's `ssl` option too. Storage is in MB and can only be increased later.

## Step 8: Create the service

Write `deno-app.json`:

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "8000" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/health", "port": 8000,
      "initialDelaySeconds": 3, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file deno-app.json
```

Use the same repository URL, provider and branch as in step 4. The first build starts automatically. If none appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json --input '{}'`.

## Step 9: Wait for the deploy and get your URL

`COMPLETED` means rolled out and serving. Poll every 10 seconds to stay under the API limit of 1,000 requests per hour. The loop prints your URL when the app is live:

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

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web`.

If the deployment fails or restarts, read the runtime log first. A missing permission shows up there as an error naming the flag:

```bash
northflank get service logs --projectId "$PROJECT_ID" --serviceId web --lineLimit 100 --textIncludes "allow-"
```

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health endpoint returns 200. With a database, it also proves the connection works:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/health"
   ```
3. The permission-error search above prints nothing, and the log shows the server's startup line (`Listening on …` for `Deno.serve` and `deno serve`).

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Container exits with "Requires net access" or "env access" | Permission flag missing from `CMD` | Add the flag the error names; rebuild |
| Build fails at `deno install` | Lockfile out of date, or a dependency that needs scripts | Update `deno.lock` locally; for npm post-install scripts use `--allow-scripts` |
| 502/503 while `COMPLETED` | Server on another port, or bound to `localhost` | Read `PORT`, set `hostname: "0.0.0.0"`, keep `internalPort` at 8000 |
| Readiness probe fails | No `/health` route | Add it; Fresh needs a route file for it |
| `EACCES` or `PermissionDenied` writing files | Image runs as `deno`, or `--allow-write` missing | Write to `/tmp` with `--allow-write=/tmp`, or use a volume |
| `ECONNREFUSED 127.0.0.1:5432` | `DATABASE_URL` not set | Re-run step 7, then `northflank restart service` |
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
- Don't reach for `-A` to silence a permission error in a non-Fresh app. Add the specific flag the error names and tell the user which one.
- Never print `DATABASE_URL` or other secret values.
- Storage can only be increased. Take a backup before migrations that change data.
- Don't delete the addon or service without explicit, per-resource confirmation. Prefer pausing.

## FAQ

### Where should I deploy my Deno app?
If the app is a set of request handlers, a serverless or edge platform that supports Deno can run it. If you want to self-host, with a pinned Deno version, explicit permission flags and a database next to it, run it as a container on a container platform, a VM or Kubernetes. The `Dockerfile` in this guide runs the same way on any of them.

### Does this work for Hono or other frameworks on Deno?
Yes. Any framework works if it listens on `PORT` and `0.0.0.0`, answers `GET /health`, and the `CMD` has the permissions it needs.

### Why `deno install` in the Dockerfile?
It downloads and caches every dependency at build time, so the container doesn't fetch code when it starts and doesn't need network access to package registries at runtime.

### How do I run database migrations?
Run your migration script as a one-off manual job built from the same repo, with a `customCommand` such as `deno run --allow-net --allow-env --allow-read=. migrate.ts`, rather than at server start.

### Is there a one-click option?
Yes, the [Deno stack template](https://northflank.com/stacks/deploy-deno) deploys a sample app from the UI. This guide does the same for your own repo, scripted.

## Conclusion

You now have a Deno server built from Git into a pinned `denoland/deno` image, with its permissions written into the `Dockerfile`, serving on port 8000 behind a public HTTPS URL, and an optional managed Postgres database wired in as `DATABASE_URL`.

**Next:** narrow the permission flags to exactly what the app uses, add Redis if you need caching, and set up database backups before real data arrives. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Bun app](deploy-bun-app.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
