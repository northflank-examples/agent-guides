<!--
agent-guide-meta:
  id: NF-STK-HONO
  description: "How to deploy a Hono app in Docker on Node.js with @hono/node-server or on Bun, listening on PORT and 0.0.0.0 with a /health route, on a public HTTPS URL."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://hono.dev/docs/getting-started/nodejs
    - https://hono.dev/docs/getting-started/bun
    - https://www.npmjs.com/package/@hono/node-server
    - https://www.npmjs.com/package/hono
    - https://bun.com/docs/runtime/http/server
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/network/configure-ports
  related: [NF-STK-EXPRESS, NF-STK-BUN, NF-DEP-004, NF-STK-DENO, NF-DEP-001]
-->

# How to deploy a Hono app

To deploy a Hono app as a long-running server, run it on Node.js with `@hono/node-server` (or on Bun, which serves Hono's `fetch` handler natively), listen on `process.env.PORT` and `0.0.0.0`, add a `GET /health` route, compile TypeScript at build time, and package it all in a multi-stage Docker image. Hono itself is runtime-agnostic, so the adapter and the start command decide where it runs.

This guide does it on Northflank with the CLI: a multi-stage `Dockerfile` for Node.js (with a Bun variant), a service built from your repo with health checks on `/health`, and a public HTTPS URL. You end with the Hono app serving on a `code.run` address, checked with `curl` against the health route and a 404. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a Hono app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app uses Hono and confirm the team, repository, branch and project before changing anything, then give Northflank access to the repository (once).
3. Start Hono with `serve({ fetch: app.fetch, port, hostname: "0.0.0.0" })` from `@hono/node-server` (Bun: `export default { port, hostname, fetch: app.fetch }`), add `GET /health`, and add a multi-stage `Dockerfile` that starts `node dist/index.js`. Push.
4. Create a project and a service built from your repo with `northflank put service combined --file hono-app.json`, on port 8080 with HTTP health checks on `/health`.
5. Wait for the deploy, then check that the live build is your pushed commit, `https://$HOST/health` returns 200 and an unknown path returns your JSON 404.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why run Hono in a container?

Hono is built on Web Standards (`Request`, `Response`, `fetch`), so the same app runs on Cloudflare Workers, Deno, Bun, Node.js and serverless functions. The runtime you pick decides what the app can do:

- **Edge and serverless** suit short, stateless handlers close to users. They limit CPU time per request, Node.js APIs and long-lived connections.
- **A long-running server** (Node.js or Bun in a container) gives you the full Node.js API, in-memory caches, WebSockets, background timers, and a private network to your database.

If you're choosing a server, where should you deploy a Hono app? Anywhere that runs a long-lived process with a port. A VPS works if you manage the runtime, reverse proxy, TLS and restarts. A container platform builds your `Dockerfile` and runs it with HTTPS, health checks and restarts.

Three details matter when Hono runs as a server:

- **`@hono/node-server`** turns Hono's `fetch` handler into a Node.js HTTP server. It defaults to port 3000, so pass `PORT` explicitly. Version 2.x needs Node.js 20 or later.
- **Bind `0.0.0.0`**, not `localhost`, so the platform's health checks and load balancer can reach the app.
- **Compile TypeScript at build time.** Running `tsx` or `ts-node` in production slows startup and ships dev dependencies.

## Prerequisites

- **A Hono app on Node.js that builds locally**, with `hono` and `@hono/node-server` in `dependencies` and a `build` script that compiles `src/index.ts` to `dist/index.js` (the `create-hono` Node.js template's `tsc` setup):
  ```bash
  npm ci && npm run build && test -f dist/index.js && echo BUILD_OK
  ```
  For Bun, run `bun install && bun run src/index.ts` instead and see the Bun variant in steps 5 and 6.
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
jq -r '"hono: \(.dependencies.hono // "missing")  @hono/node-server: \(.dependencies["@hono/node-server"] // "missing")"' package.json
ls package-lock.json pnpm-lock.yaml yarn.lock bun.lock bun.lockb Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId hono-app -o json >/dev/null 2>&1 \
  && echo "project: hono-app exists and will be reused" \
  || echo "project: hono-app will be created"
```

- If `hono` says `missing`, this isn't a Hono app. Stop and use a guide for your framework. If only `@hono/node-server` is missing, the app targets another runtime: use the Bun variant if it runs on Bun, or add `@hono/node-server`.
- The lockfile tells you the package manager. Step 6 assumes `package-lock.json` (or `bun.lock` for the Bun variant); for pnpm or yarn, swap the install command.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "hono-app", "region": "europe-west" }'
export PROJECT_ID=hono-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Start Hono on PORT and 0.0.0.0

```ts
// src/index.ts
import { serve } from "@hono/node-server";
import { Hono } from "hono";
import { logger } from "hono/logger";

const app = new Hono();
app.use(logger());

app.get("/health", (c) => c.json({ status: "ok" }));
app.get("/", (c) => c.text("Hello from Hono"));

app.notFound((c) => c.json({ error: "not found" }, 404));
app.onError((err, c) => {
  console.error(err);
  return c.json({ error: "internal error" }, 500);
});

const server = serve(
  { fetch: app.fetch, port: Number(process.env.PORT ?? 8080), hostname: "0.0.0.0" },
  (info) => console.log(`Listening on ${info.port}`),
);

process.on("SIGTERM", () => {
  server.close((err) => process.exit(err ? 1 : 0));
});
```

The `SIGTERM` handler follows the graceful shutdown example in the Hono Node.js docs: every deploy stops the old container with `SIGTERM`, and closing the server lets in-flight requests finish. If the app uses a database, make `/health` run a cheap query such as `SELECT 1`.

**Bun variant.** Bun serves the default export's `fetch` handler, so `@hono/node-server` isn't needed:

```ts
// src/index.ts (Bun)
import { Hono } from "hono";
const app = new Hono();
app.get("/health", (c) => c.json({ status: "ok" }));
app.get("/", (c) => c.text("Hello from Hono on Bun"));

export default { port: Number(process.env.PORT ?? 8080), hostname: "0.0.0.0", fetch: app.fetch };
```

## Step 6: Write the Dockerfile

A multi-stage build compiles TypeScript with dev dependencies, then ships only `dist/` and production dependencies. This is a condensed version of the Dockerfile in the Hono Node.js docs:

```dockerfile
FROM node:24-slim AS build
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm ci
COPY src ./src
RUN npm run build && npm prune --omit=dev

FROM node:24-slim
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/package.json ./
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
USER node
EXPOSE 8080
CMD ["node", "dist/index.js"]
```

Run `node` directly rather than `npm start`, so `SIGTERM` reaches the app. If `package.json` has `"type": "module"`, keep it in the final stage (it's copied above) so Node loads `dist/index.js` as ESM.

**Bun variant.** Bun runs TypeScript directly, so there's no build stage:

```dockerfile
FROM oven/bun:1
WORKDIR /usr/src/app
COPY package.json bun.lock* ./
RUN bun install --frozen-lockfile --production
COPY . .
USER bun
EXPOSE 8080
CMD ["bun", "run", "src/index.ts"]
```

Add a `.dockerignore` with `node_modules`, `dist`, `.git` and `.env`.

Northflank builds what's on the remote branch, not your local working tree, so commit and push these files (plus your lockfile) before you create the service:

```bash
git add Dockerfile .dockerignore src/ package.json
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Create the service

Write `hono-app.json`:

```json
{
  "name": "hono-app",
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
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file hono-app.json
```

Use the same repository URL, provider and branch as in step 4. The first build starts automatically. The same file works for the Bun variant; only the `Dockerfile` differs. For a monorepo, point `dockerFilePath` and `dockerWorkDir` at the app's folder. `nf-compute-20` (0.2 vCPU, 512 MB) suits a small API.

## Step 8: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds and prints your URL when the app is live:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId hono-app --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId hono-app --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId hono-app --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId hono-app`. Every push to the branch rebuilds and redeploys.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId hono-app --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health route returns 200 over HTTPS:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/health"   # 200
   ```
3. Routing and the not-found handler work:
   ```bash
   curl -s "https://$HOST/"                  # Hello from Hono
   curl -s "https://$HOST/does-not-exist"    # {"error":"not found"}
   ```
4. The server logged its startup line and requests (the `logger()` middleware prints `<-- GET /health` lines):
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId hono-app --lineLimit 100 --textIncludes "Listening on"
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Probes fail, deployment never `COMPLETED` | Server on port 3000 (the default) or `localhost` | Pass `port` from `PORT` and `hostname: "0.0.0.0"` |
| `Cannot find module '/app/dist/index.js'` | `tsc` writes somewhere else | Match `outDir` in `tsconfig.json` to the `COPY` and `CMD` paths |
| `SyntaxError: Cannot use import statement outside a module` | ESM output without `"type": "module"` | Keep `package.json` in the final image, or compile to CommonJS |
| `npm ci` fails | Missing or stale `package-lock.json` | Commit an up-to-date lockfile |
| Unsupported engine warning for `@hono/node-server` | Node.js older than 20 | Use `node:24-slim` (or another supported LTS) |
| Bun image: `error: lockfile had changes` | `bun.lock` out of date | Run `bun install` locally and commit the lockfile |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

For build output, run `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId hono-app`; see [view logs](https://northflank.com/docs/v1/application/observe/view-logs).

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Don't swap the runtime (Node.js, Bun, Workers) without asking; it changes which APIs the code can use.
- Keep secrets out of the image and the repo. Put them in a secret group restricted to the service, then restart it so it picks them up: `northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{"name":"hono-env","type":"secret","secretType":"environment","priority":10,"restrictions":{"restricted":true,"nfObjects":[{"id":"hono-app","type":"service"}]},"secrets":{"variables":{"<KEY>":"<VALUE>"}}}'`, then `northflank restart service --projectId "$PROJECT_ID" --serviceId hono-app`. See [secret groups](https://northflank.com/docs/v1/application/secure/manage-secret-groups).
- Don't delete the service without explicit confirmation.

## FAQ

### Where should I deploy my Hono app?
If it's a small stateless API, an edge or serverless runtime works. If it needs Node.js APIs, WebSockets, long requests or a database on a private network, run it as a server on a container platform, a VPS or Kubernetes. A container platform that builds your `Dockerfile` and handles TLS and health checks, as in this guide, is the shortest path.

### Node.js or Bun?
Both run Hono well. Node.js with `@hono/node-server` has the widest library compatibility; Bun skips the build step and starts faster. The Northflank steps are the same for both. See [how to deploy a Bun app](deploy-bun-app.md) for more on Bun images.

### How do I get the client IP behind the load balancer?
`getConnInfo` from `@hono/node-server/conninfo` returns the socket address, which is the proxy. Read the `X-Forwarded-For` header with `c.req.header("x-forwarded-for")` for the client, and treat it as a hint rather than proof, since clients can send it too.

### How does this compare to Express?
The deployment is the same: a port, a health route and a `Dockerfile`. Hono uses Web Standard `Request`/`Response` objects and runs on more runtimes. See [how to deploy an Express app](deploy-express-app.md).

## Conclusion

You now have a Hono app running as a server on Node.js (or Bun), built from a multi-stage `Dockerfile`, bound to `PORT` on `0.0.0.0`, with health checks on `/health` and a clean shutdown on every deploy.

**Next:** add a database by following [how to deploy a Node.js application](deploy-nodejs-application.md), or link a custom domain with [link a domain to a port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy an Express app](deploy-express-app.md)
- [How to deploy a Bun app](deploy-bun-app.md)
- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a Deno app](deploy-deno-app.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
