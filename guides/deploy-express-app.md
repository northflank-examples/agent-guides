<!--
agent-guide-meta:
  id: NF-STK-EXPRESS
  description: "How to deploy an Express app (Express 5) with Docker on a public HTTPS URL: listen on PORT and 0.0.0.0, set trust proxy, add a /health route, shut down cleanly."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://expressjs.com/en/guide/migrating-5.html
    - https://expressjs.com/en/guide/behind-proxies.html
    - https://expressjs.com/en/advanced/healthcheck-graceful-shutdown.html
    - https://expressjs.com/en/advanced/best-practice-performance.html
    - https://www.npmjs.com/package/express
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/scale/scale-instances
  related: [NF-DEP-004, NF-STK-HONO, NF-STK-NEST, NF-DEP-001, NF-DB-001]
-->

# How to deploy an Express app

To deploy an Express app, package it in a Docker image that runs `node` directly, listen on `process.env.PORT` and `0.0.0.0`, set `app.set("trust proxy", 1)` so `req.ip` and `req.protocol` reflect the real client behind the load balancer, add a `GET /health` route for health checks, and close the server cleanly on `SIGTERM`. Binding to `localhost` is the usual reason an Express deploy builds fine but never becomes healthy.

This guide does it on Northflank with the CLI for Express 5: a production `Dockerfile`, a service built from your repo with readiness and liveness checks on `/health`, and a public HTTPS URL. You end with the app serving on a `code.run` address, checked with `curl` for the health route and for `req.secure` behind the proxy. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy an Express app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app uses Express and confirm the team, repository, branch and project before changing anything, then give Northflank access to the repository (once).
3. Make the app listen on `PORT` and `0.0.0.0`, set `trust proxy` to `1`, add `GET /health`, close the server on `SIGTERM`, and add a `Dockerfile` that runs `node` directly. Push.
4. Create a project and a service built from your repo with `northflank put service combined --file express-app.json`, on port 8080 with HTTP health checks on `/health`.
5. Wait for the deploy, then check that the live build is your pushed commit and that `https://$HOST/health` returns 200 with `"secure": true`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why these Express settings?

Express runs fine on a laptop with defaults, but four things change when it sits behind a platform's load balancer:

- **Port and host.** The platform routes to one port inside the container. Read it from `PORT` and bind `0.0.0.0`, or health checks can't reach the app.
- **`trust proxy`.** TLS ends at the load balancer, so without `trust proxy`, `req.ip` is the proxy's address and `req.protocol` is `http`. That breaks rate limiting by IP, `secure` cookies and HTTPS redirects. The Express docs say a number means "this many proxy hops", and warn that `true` lets clients spoof the headers.
- **A health route.** A cheap `GET /health` tells the platform when to send traffic and when to restart a stuck process.
- **Graceful shutdown.** Every deploy stops the old container with `SIGTERM`. Closing the server lets in-flight requests finish.

This guide covers an Express app with no database. To add PostgreSQL through `DATABASE_URL` and run migrations, see [how to deploy a Node.js application](deploy-nodejs-application.md) (optional reading); the service setup is the same.

Where should you deploy an Express app? Anywhere that runs a long-lived Node.js process with a port. Serverless platforms can run Express through an adapter, but each request becomes a function call, so in-memory state, WebSockets and long requests don't behave like a normal server. A VPS works if you manage Node, a reverse proxy, TLS and restarts. A container platform builds your `Dockerfile` and runs it with HTTPS, health checks and restarts.

## Prerequisites

- **An Express app that runs locally:**
  ```bash
  npm ci && npm start
  ```
  Stop it with Ctrl+C once it says it's listening. The app should use Express 5 with a committed `package-lock.json`; if you're still on Express 4, see the FAQ.
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
jq -r '"express: \(.dependencies.express // "missing")"' package.json
npm ls express
ls package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId express-app -o json >/dev/null 2>&1 \
  && echo "project: express-app exists and will be reused" \
  || echo "project: express-app will be created"
```

- If `express` says `missing`, this isn't an Express app. Stop and use a guide for your framework. `npm ls express` shows the installed major version; this guide is written for `5.x`.
- The lockfile tells you the package manager. Step 6 assumes `package-lock.json`; for pnpm or yarn, swap the install command.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "express-app", "region": "europe-west" }'
export PROJECT_ID=express-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Make the app production-ready

A minimal `server.js` with all four settings:

```js
// server.js
const express = require("express");

const app = express();
app.set("trust proxy", 1);           // one load balancer in front of the app
app.disable("x-powered-by");
app.use(express.json());

app.get("/health", (req, res) => {
  res.json({ status: "ok", secure: req.secure });
});

app.get("/", async (req, res) => {
  res.send(`Hello from Express, ${req.ip}`);
});

app.use((req, res) => res.status(404).json({ error: "not found" }));
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({ error: "internal error" });
});

const port = Number(process.env.PORT ?? 8080);
const server = app.listen(port, "0.0.0.0", (error) => {
  if (error) throw error;              // Express 5 passes listen errors (e.g. EADDRINUSE) here
  console.log(`Listening on ${port}`);
});

process.on("SIGTERM", () => {
  console.log("SIGTERM received, closing server");
  server.close(() => process.exit(0));
  setTimeout(() => process.exit(1), 10_000).unref();
});
```

Notes for Express 5:

- Async handlers that throw or reject go to the error handler automatically; no `.catch(next)` needed.
- `req.body` is `undefined` unless a body parser such as `express.json()` ran.
- Keep `/health` cheap. If the app has a database, make it run `SELECT 1` so a failing probe means a real problem.

## Step 6: Write the Dockerfile

```dockerfile
FROM node:24-slim
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
USER node
EXPOSE 8080
CMD ["node", "server.js"]
```

- **Run `node` directly**, not `npm start`. npm doesn't always forward `SIGTERM` to the app, so graceful shutdown can be skipped.
- **`NODE_ENV=production`** turns on Express's view caching and less verbose errors, as the Express performance guide recommends.
- **TypeScript:** add a build stage that runs `npm ci` and `npm run build`, then copy `dist/` and production `node_modules` into the final stage and run `node dist/server.js`.

Add a `.dockerignore` with `node_modules`, `.git` and `.env`, so local files never reach the image.

Northflank builds what's on the remote branch, not your local working tree, so commit and push these files before you create the service. If your entry file isn't `server.js`, add yours instead:

```bash
git add server.js Dockerfile .dockerignore
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Create the service

Write `express-app.json`:

```json
{
  "name": "express-app",
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
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file express-app.json
```

Use the same repository URL, provider and branch as in step 4. The first build starts automatically. For a monorepo, point `dockerFilePath` and `dockerWorkDir` at the app's folder (for example `/apps/api/Dockerfile` and `/apps/api`). `nf-compute-20` (0.2 vCPU, 512 MB) suits a small API; list other plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[].id'`.

## Step 8: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds and prints your URL when the app is live:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId express-app --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId express-app --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId express-app --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId express-app`. Every push to the branch rebuilds and redeploys.

## Step 9: Check the proxy settings

```bash
curl -s "https://$HOST/health"     # {"status":"ok","secure":true}
```

`"secure": true` means Express saw `X-Forwarded-Proto: https` from the load balancer and trusted it. If it says `false`, the proxy headers aren't arriving as expected: check the `trust proxy` line, and if it's set, read `req.headers` once in the logs to see which `X-Forwarded-*` headers come through. Don't switch to `trust proxy: true` to make it pass; the Express docs warn that lets clients fake their IP.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId express-app --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health route returns 200 over HTTPS:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/health"   # 200
   ```
3. Express recognises HTTPS behind the proxy: `curl -s "https://$HOST/health"` includes `"secure":true`.
4. Unknown routes return your JSON 404, not an HTML stack trace:
   ```bash
   curl -s "https://$HOST/does-not-exist"   # {"error":"not found"}
   ```
5. The app logged its startup line:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId express-app --lineLimit 100 --textIncludes "Listening on"
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Build passes, deployment never `COMPLETED`, probes fail | App listens on `localhost` or a hard-coded port | Bind `0.0.0.0` and read `process.env.PORT` |
| `npm ci` fails in the build | No `package-lock.json`, or it's out of sync | Commit an up-to-date lockfile |
| `TypeError: Missing parameter name` at startup | Express 4 wildcard syntax (`/*`) on Express 5 | Use named wildcards: `/*splat` or `/{*splat}` |
| `req.body` is `undefined` | No body parser | Add `app.use(express.json())` |
| `"secure": false`, or `req.ip` is always the same internal address | `trust proxy` not set | Add `app.set("trust proxy", 1)`; see step 9 |
| Requests cut off during deploys | No `SIGTERM` handling, or started with `npm start` | Handle `SIGTERM` and run `node` directly |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

For build output, run `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId express-app`; see [view logs](https://northflank.com/docs/v1/application/observe/view-logs).

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Don't set `trust proxy` to `true` to fix a check; ask the user instead.
- Keep secrets out of the image and the repo. Put them in a secret group restricted to the service, then restart it so it picks them up: `northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{"name":"express-env","type":"secret","secretType":"environment","priority":10,"restrictions":{"restricted":true,"nfObjects":[{"id":"express-app","type":"service"}]},"secrets":{"variables":{"<KEY>":"<VALUE>"}}}'`, then `northflank restart service --projectId "$PROJECT_ID" --serviceId express-app`. See [secret groups](https://northflank.com/docs/v1/application/secure/manage-secret-groups).
- If the app keeps state in memory (sessions, rate-limit counters), tell the user before scaling above one instance; move that state to Redis first.
- Don't delete the service without explicit confirmation.

## FAQ

### Where should I deploy my Express app?
Somewhere that runs a long-lived Node.js process with a port and HTTPS in front of it: a container platform, a VPS or Kubernetes. Serverless hosts can run Express through an adapter, but WebSockets, in-memory state and long requests don't fit. A container platform that builds your `Dockerfile`, runs health checks and handles TLS, as in this guide, is the shortest path.

### How is this different from the Node.js guide?
This guide covers what's specific to Express 5: `trust proxy`, the error handler, the Express 5 changes and graceful shutdown. [How to deploy a Node.js application](deploy-nodejs-application.md) adds a managed PostgreSQL database and migrations, and works for Fastify and NestJS too.

### I'm on Express 4. Do I need to upgrade first?
No, the deployment steps are the same. Upgrade when you can: Express 5 forwards rejected promises to the error handler and requires named wildcards (`/*splat`). See the Express migration guide.

### Can I run more than one instance?
Yes, if the app keeps no state in memory. Raise `deployment.instances` with `northflank patch service combined`; traffic is load-balanced across instances. See [scale instances](https://northflank.com/docs/v1/application/scale/scale-instances).

## Conclusion

You now have an Express 5 app running from a production `Dockerfile`, bound to `PORT` on `0.0.0.0`, trusting exactly one proxy hop, with health checks on `/health` and a clean shutdown on every deploy.

**Next:** add a database by following [how to deploy a Node.js application](deploy-nodejs-application.md), or link a custom domain with [link a domain to a port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a Hono app](deploy-hono-app.md)
- [How to deploy a NestJS app](deploy-nestjs-app.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
