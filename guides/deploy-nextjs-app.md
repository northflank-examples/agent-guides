<!--
agent-guide-meta:
  id: NF-DEP-005
  description: "How to deploy a Next.js app with Docker and standalone output, NEXT_PUBLIC_* values as build arguments, secrets at runtime and checks for SSR and the bundle."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/services/start-service-build
    - https://northflank.com/docs/v1/api/project/services/list-service-builds
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/inject-build-arguments
    - https://northflank.com/docs/v1/application/run/inject-runtime-variables
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DB-001, NF-DEP-012, NF-DEP-004, NF-DEP-001]
-->

# How to deploy a Next.js app

To deploy a Next.js app, build it with `output: 'standalone'` in a multi-stage Docker image that runs `node server.js` with `HOSTNAME=0.0.0.0` on port 3000, pass `NEXT_PUBLIC_*` values as build arguments, and inject server-only secrets as runtime variables. `NEXT_PUBLIC_*` values are compiled into the client bundle at build time, so setting them only at run time leaves the browser with `undefined`.

This guide does it on Northflank with the CLI: a combined service that builds from Git, with public values in `buildArguments`, secrets in a runtime secret group and a health check on `/api/health`. You end with a public HTTPS URL, verified by checks that SSR renders per request, the public value is in the client bundle and no secret leaked. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to deploy a Next.js app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Next.js app and confirm the team, repository, branch and project before changing anything. Give Northflank access to the repository (once).
3. Set `output: 'standalone'`, add a health route and a small render-check page, and push the multi-stage `Dockerfile` below (`node server.js`, `HOSTNAME=0.0.0.0`, port 3000).
4. Create a project and a service with `northflank put service combined --file web.json`: `NEXT_PUBLIC_*` values go in `buildArguments`, server-only secrets in a runtime secret group.
5. Wait for the deploy, then check that the live build is your pushed commit, the render-check page shows a new server timestamp on each request and the public value is in the bundle.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your Next.js app as a container

A Next.js app that renders on the server needs a running Node.js process, environment variables split between build and run time, and often a database next to it. A static host only fits a fully static export; serverless hosts run Next.js as functions; a container platform or a VM runs the standalone server as one long-running process you control. A raw VM or your own Kubernetes cluster works too, but you manage the OS, TLS and rollouts yourself.

### Build-time vs runtime variables

`next build` replaces every `process.env.NEXT_PUBLIC_X` in client code with the literal value it sees during the build. If the value isn't there at build time, the browser gets `undefined`, and setting it later as a runtime variable changes nothing. Server code, on the other hand, reads `process.env` when it runs.

Northflank keeps the two separate. Build arguments are passed to the `Dockerfile` with `--build-arg` and don't persist in the image unless you copy them into `ENV`. Runtime variables are injected into the running container. Running Next.js as a container also lets the app sit next to its database, workers and cron jobs in the same project. See [how to deploy a full-stack app as one template](deploy-full-stack-app.md).

## Prerequisites

- **A Next.js app that builds locally:**
  ```bash
  npm ci && npm run build && echo BUILD_OK
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
jq -r '"next: \(.dependencies.next // "missing")  react: \(.dependencies.react // "missing")"' package.json
ls package-lock.json pnpm-lock.yaml yarn.lock bun.lock bun.lockb Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId nextjs-app -o json >/dev/null 2>&1 \
  && echo "project: nextjs-app exists and will be reused" \
  || echo "project: nextjs-app will be created"
```

- If `next` says `missing`, this isn't a Next.js app. Stop and use a guide for your framework.
- The lockfile tells you the package manager. Step 6 assumes `package-lock.json`; for pnpm, yarn or bun, swap the lockfile and install command.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "nextjs-app", "region": "europe-west" }'
export PROJECT_ID=nextjs-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Prepare the app

Three small changes make the app container-ready and verifiable.

**Standalone output.** It makes `next build` produce a self-contained `server.js`:

```js
// next.config.js (or next.config.mjs / next.config.ts)
module.exports = { output: 'standalone' };
```

**A health route** that returns 200, for example App Router `app/api/health/route.ts`:

```ts
export function GET() {
  return Response.json({ ok: true });
}
```

**A render-check page** that shows a value only the server can produce on each request. Check 3 reads it to prove the page is rendered per request, not served from a static build. For the App Router, `app/render-check/page.tsx`:

```tsx
export const dynamic = 'force-dynamic';

export default function RenderCheck() {
  const renderedAt = new Date().toISOString();
  return <p data-rendered-at={renderedAt}>Rendered at {renderedAt}</p>;
}
```

With the Pages Router, put the same `<p>` in `pages/render-check.tsx` and pass `renderedAt` from `getServerSideProps`. You can delete the page once you've verified the deploy.

Then list which variables are public and which are server-only:

```bash
grep -rhoE 'process\.env\.[A-Z0-9_]+' app pages src lib 2>/dev/null | sort -u
```

Anything starting with `NEXT_PUBLIC_` goes in build arguments. Everything else goes in runtime variables.

## Step 6: Write the `Dockerfile` and push

This follows the standard Next.js standalone Docker setup (the official `with-docker` example uses Node 24; Node 20 reached end of life in April 2026). Add one `ARG`/`ENV` pair per `NEXT_PUBLIC_*` variable in the **builder** stage:

```dockerfile
FROM node:24-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:24-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:24-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1 PORT=3000 HOSTNAME=0.0.0.0
RUN addgroup --system --gid 1001 nodejs && adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

A few details:

- `HOSTNAME=0.0.0.0` makes the standalone server listen on all interfaces. Without it, the container may only answer on localhost and fail its health check.
- The standalone output doesn't include `public/` or `.next/static`, which is why they're copied separately. If the repo has no `public/` folder, create an empty one or remove that line.
- For pnpm, yarn or bun, swap the lockfile and install command in the `deps` stage.
- Build arguments are only in scope in the stage that declares them, so declare them in `builder`. If the app reads no `NEXT_PUBLIC_*` variables, leave out the `ARG`/`ENV` pair.

Northflank builds what's in your repository, not the files on your machine, so commit and push them:

```bash
git add Dockerfile next.config.* app/api/health app/render-check   # plus any other changes from step 5
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Create the service with build arguments and runtime variables

Write `web.json`. `buildArguments` holds public build-time values. `runtimeEnvironment` holds non-secret runtime values. If the app reads no `NEXT_PUBLIC_*` variables, leave out `buildArguments`.

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-50" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "buildArguments": { "NEXT_PUBLIC_API_URL": "https://api.example.com" },
  "runtimeEnvironment": { "PORT": "3000" },
  "ports": [ { "name": "http", "internalPort": 3000, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/api/health", "port": 3000,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/api/health", "port": 3000,
      "initialDelaySeconds": 20, "periodSeconds": 20, "timeoutSeconds": 2, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file web.json
```

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined` with the same body. The first build starts automatically.

## Step 8: Add server-only secrets (if the app has any)

Skip this step if the app reads no secrets. Server-only secrets (database URLs, API keys, `NEXTAUTH_SECRET`/`AUTH_SECRET`) go in a secret group with `secretType: "environment"`, so they're injected at runtime and never baked into an image layer. Export the value in your shell first; the command reads it without printing it:

```bash
jq -n --arg s "$AUTH_SECRET" '{
  name: "web-runtime", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "web", type: "service" } ] },
  secrets: { variables: { AUTH_SECRET: $s } }
}' > web-runtime.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file web-runtime.json >/dev/null && rm web-runtime.json
northflank restart service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json
```

For a database, link `POSTGRES_URI` as `DATABASE_URL` in that same group. See [connect database secrets to workloads](https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads).

## Step 9: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds and prints your URL when the app is live. You can also watch the build in the Northflank dashboard.

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

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web`. Most Next.js build failures are type or lint errors that also fail locally with `npm run build`. The troubleshooting table covers the other common causes.

## Step 10 (optional): Change a public value (rebuild required)

To change `NEXT_PUBLIC_API_URL`, update the build arguments and start a new build. Send the **complete** map, because existing keys may be replaced:

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json --input '{
  "buildArguments": { "NEXT_PUBLIC_API_URL": "https://api.new-domain.com" }
}'
northflank start service build --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
  --input '{ "branch": "main" }'
```

Then run the step 9 loop again. To try a value for one build only, without saving it, pass it as `overrides.buildArguments` in the `start service build` body. Values there take precedence over the saved ones for that build.

Runtime-only changes need just a restart: patch `runtimeEnvironment` (complete map) or edit the secret group, then run `northflank restart service`.

## How to know it worked

You're done when all of these hold. Skip check 4 if the app reads no `NEXT_PUBLIC_*` variables, and check 5 if it has no server-only secrets.

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health route returns 200:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/api/health"
   ```
3. The render-check page from step 5 is rendered on the server for each request. Both lines should show a timestamp, the timestamps should differ, and `SSR_OK` should print:
   ```bash
   a=$(curl -s "https://$HOST/render-check" | grep -oE 'data-rendered-at="[^"]+"'); sleep 2
   b=$(curl -s "https://$HOST/render-check" | grep -oE 'data-rendered-at="[^"]+"')
   echo "$a"; echo "$b"
   [ -n "$a" ] && [ -n "$b" ] && [ "$a" != "$b" ] && echo SSR_OK
   ```
   If both lines are empty, the page isn't deployed: check that `app/render-check` was pushed. If they're identical, the page was rendered at build time: keep `export const dynamic = 'force-dynamic'`.
4. The public value is compiled into the client bundle:
   ```bash
   EXPECTED='https://api.example.com'
   curl -s "https://$HOST/" | grep -oE '/_next/static/[^"]+\.js' | sort -u | while read -r p; do
     curl -s "https://$HOST$p" | grep -qF "$EXPECTED" && echo "FOUND in $p"
   done
   ```
   At least one `FOUND` line should print. If you changed the value in step 10, search for the new value. If the value is only used on one page, it may sit in a chunk the homepage doesn't reference; run the loop against that page's URL instead.
5. No server-only secret leaked into the bundle. Run the same loop with a distinctive fragment of a secret (not the whole value) as `EXPECTED` and confirm that nothing prints.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `NEXT_PUBLIC_*` is `undefined` in the browser | Set as a runtime variable, or `ARG` declared in the wrong stage | Add `ARG`/`ENV` in the `builder` stage, set it in `buildArguments`, rebuild |
| `Cannot find module '/app/server.js'` | `output: 'standalone'` missing | Add it to `next.config.*`; rebuild |
| CSS/JS 404s, page unstyled | `.next/static` not copied | Keep the `COPY … /app/.next/static ./.next/static` line |
| Health check fails, logs show `- Local: http://localhost:3000` only | Server bound to localhost | Set `HOSTNAME=0.0.0.0` in the runner stage |
| Build fails with `COPY failed: … /app/public` | No `public/` dir | Create it or drop the line |
| Render-check timestamps are identical | The page was prerendered at build time | Keep `export const dynamic = 'force-dynamic'` in the page, push, rebuild |
| Image optimisation errors at runtime | Missing image processing dependency for your Next.js version | Follow the Next.js docs for `sharp` in standalone mode |
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
- Never put a secret in `buildArguments` or in a `NEXT_PUBLIC_*` variable. Anything there can end up in the image or the browser bundle. If the user asks for this, explain the risk and ask them to confirm.
- Don't change `next.config.*` beyond adding `output: 'standalone'` without telling the user, and tell them about the health route and render-check page you add.
- After changing a public value, confirm it with check 4. A green deploy doesn't prove the bundle has the new value.
- Don't delete the service to "start fresh". Patch it and rebuild.

## FAQ

### Where should I deploy my Next.js app?
If the app is fully static, any static host will do. If it uses SSR, API routes or server actions, you need a host that runs Node.js: a serverless platform, a container platform or a VM. Self-hosting with Docker and `output: 'standalone'` on a container platform gives you one portable image that runs the same way anywhere and can sit next to its database and workers.

### Can I set `NEXT_PUBLIC_*` at runtime instead?
Not with the standard build: the values are inlined at build time. To vary them per environment without rebuilding, read them on the server and pass them to the client as props or from an API route.

### Do API routes and server actions work?
Yes. They run inside `node server.js` in the same container. Middleware runs there too.

### How many instances should I run?
Start with one, then scale with `deployment.instances` or autoscaling. If you use ISR or the data cache across instances, configure a shared cache handler. See [autoscale deployments](https://northflank.com/docs/v1/application/scale/autoscale-deployments).

### Can I use a secret group for build arguments?
Yes. A group with `secretType: "arguments"` (or `"environment-arguments"` for both) is passed to builds. Use it for build-time values shared by several services, and remember that public values still end up in the bundle.

## Conclusion

You now have a Next.js app running as a standalone Node container on Northflank, with public values compiled in at build time, secrets injected only at runtime, and checks that prove both.

**Next:** add a database with [how to deploy PostgreSQL](deploy-postgresql-database.md), put the app on your own domain with [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank), or give every pull request its own URL with [set up a preview environment](https://northflank.com/docs/v1/application/release/pipeline/set-up-a-preview-environment). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a full stack app](deploy-full-stack-app.md)
- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
