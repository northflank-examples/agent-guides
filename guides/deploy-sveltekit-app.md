<!--
agent-guide-meta:
  id: NF-STK-SVELTEKIT
  description: "How to deploy a SvelteKit app with adapter-node and Docker: a health-checked Node server, optional PostgreSQL as DATABASE_URL, and ORIGIN set for forms."
  sources:
    - https://svelte.dev/docs/kit/adapter-node
    - https://svelte.dev/docs/kit/environment-variables
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/services/list-service-builds
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
  related: [NF-DEP-001, NF-DB-001, NF-DEP-005, NF-STK-NUXT]
-->

# How to deploy a SvelteKit app

To deploy a SvelteKit app that uses server-side rendering, switch it to `@sveltejs/adapter-node`, build a multi-stage Docker image that runs `node build` on `HOST=0.0.0.0` and `PORT=3000`, add a health route, read secrets as runtime environment variables, and set `ORIGIN` to the public HTTPS URL. A missing `ORIGIN` is the usual reason form submissions fail with `Cross-site POST form submissions are forbidden`.

This guide does it on Northflank with the CLI: one service built from Git with a `/healthz` health check, an optional managed PostgreSQL database linked as `DATABASE_URL`, and `ORIGIN` set from the hostname Northflank assigns. You end with a public HTTPS URL serving server-rendered pages, verified by a per-request server timestamp and `printenv` checks. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a SvelteKit app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a SvelteKit app and confirm the team, repository, branch and project before changing anything. Give Northflank access to the repository (once).
3. Switch to `@sveltejs/adapter-node`, add a `/healthz` endpoint and a small render-check page, and push the multi-stage `Dockerfile` below (`node build` on `HOST=0.0.0.0`, `PORT=3000`).
4. Create a project, optionally a PostgreSQL addon linked as `DATABASE_URL`, and the service with `northflank put service combined --file web.json`.
5. Read the public hostname, set `ORIGIN=https://<host>` in `runtimeEnvironment` and restart the service.
6. Check that the live build is your pushed commit, the render-check page shows a new server timestamp on each request and `ORIGIN` is set.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your SvelteKit app on Northflank

What a SvelteKit app needs from a host depends on its adapter. A fully prerendered app can go on any static host, serverless platforms use their own adapters, and self-hosting means `adapter-node`: a long-running Node server on a port, plus a database if the app has one. A container platform or a VM runs that server; a raw VM or your own Kubernetes cluster leaves the OS, TLS and restarts to you, while a container platform builds your Dockerfile and handles them.

### Why adapter-node in a container

`adapter-node` turns a SvelteKit app into a standalone Node server: `npm run build` writes it to `build/`, and `node build` starts it. That server needs `build/`, `package.json` and the production `node_modules`, which is exactly what a small container image holds. On Northflank, one **combined service** builds the image from Git on every push and runs it behind HTTPS, and a PostgreSQL **addon** can sit next to it in the same project.

## Prerequisites

- **A SvelteKit app that builds locally:**
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
jq -r '"kit: \(.devDependencies["@sveltejs/kit"] // .dependencies["@sveltejs/kit"] // "missing")  adapter-node: \(.devDependencies["@sveltejs/adapter-node"] // .dependencies["@sveltejs/adapter-node"] // "not installed")"' package.json
ls package-lock.json pnpm-lock.yaml yarn.lock bun.lock bun.lockb Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId sveltekit-app -o json >/dev/null 2>&1 \
  && echo "project: sveltekit-app exists and will be reused" \
  || echo "project: sveltekit-app will be created"
```

- If `kit` says `missing`, this isn't a SvelteKit app. Stop and use a guide for your framework.
- If `adapter-node` says `not installed`, step 5 installs it.
- The lockfile tells you the package manager. Step 6 assumes `package-lock.json`; for pnpm, yarn or bun, swap the lockfile and install command.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "sveltekit-app", "region": "europe-west" }'
export PROJECT_ID=sveltekit-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Switch to adapter-node and add a health route and a render-check page

```bash
npm i -D @sveltejs/adapter-node
```

On **SvelteKit 3**, the adapter goes in `vite.config.js` (SvelteKit 3 no longer reads `svelte.config.js`):

```js
import adapter from '@sveltejs/adapter-node';
import { sveltekit } from '@sveltejs/kit/vite';
import { defineConfig } from 'vite';

export default defineConfig({ plugins: [sveltekit({ adapter: adapter() })] });
```

On **SvelteKit 2**, replace `adapter-auto` with `adapter-node` in `svelte.config.js` (`kit: { adapter: adapter() }`).

Add a health endpoint at `src/routes/healthz/+server.js`:

```js
export function GET() {
  return new Response('ok');
}
```

Add a render-check page that shows a value only the server can produce on each request. Check 3 reads it to prove pages are rendered per request, not prerendered. `src/routes/render-check/+page.server.js`:

```js
export const prerender = false;

export function load() {
  return { renderedAt: new Date().toISOString() };
}
```

and `src/routes/render-check/+page.svelte`:

```svelte
<script>
  let { data } = $props();
</script>

<p data-rendered-at={data.renderedAt}>Rendered at {data.renderedAt}</p>
```

On Svelte 4, use `export let data;` instead of `$props()`. You can remove the page once you've verified the deploy.

**Environment variables.** `adapter-node` doesn't load `.env` files in production, so every value must come from the platform. Read secrets at runtime, not at build time: on SvelteKit 3, declare them in `src/env.ts` and import from `$app/env/private` (variables are dynamic, meaning read at runtime, unless you set `static: true`); on SvelteKit 2, use `$env/dynamic/private`. Reading `process.env.DATABASE_URL` inside a server-only module (`src/lib/server/db.js`) also works with `adapter-node` on either version. Don't query the database from prerendered routes, because builds don't receive runtime secrets.

## Step 6: Write the `Dockerfile` and push

```dockerfile
FROM node:22-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --omit=dev

FROM node:22-slim
WORKDIR /app
ENV NODE_ENV=production HOST=0.0.0.0 PORT=3000
COPY --from=build --chown=node:node /app/build ./build
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/package.json ./package.json
USER node
EXPOSE 3000
CMD ["node", "build"]
```

`HOST` and `PORT` are `adapter-node`'s own variables (defaults `0.0.0.0` and `3000`). For pnpm, yarn or bun, swap the lockfile and install command. If you changed the adapter's `out` option, copy that directory instead of `build`. Put the file at the repo root.

Northflank builds what's in your repository, not the files on your machine, so commit and push them:

```bash
git add Dockerfile package.json package-lock.json svelte.config.js vite.config.js src/   # whichever step 5 changed
git commit -m "Add Northflank deployment files"
git push
```

## Step 7 (optional): Add PostgreSQL and link `DATABASE_URL`

Skip this step if the app has no database.

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "web-db", "type": "postgresql", "version": "16-latest", "tlsEnabled": true,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'

ADDON_OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId web-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in
    running) ADDON_OK=1; break ;;
    failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break ;;
  esac
  sleep 10
done
[ "$ADDON_OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon before linking it"; false; }

northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "web-db-env", "type": "secret", "secretType": "environment", "priority": 10,
  "restrictions": { "restricted": true, "nfObjects": [ { "id": "web", "type": "service" } ] },
  "addonDependencies": [
    { "addonId": "web-db", "keys": [ { "keyName": "POSTGRES_URI", "aliases": ["DATABASE_URL"] } ] }
  ]
}'
```

Storage is in MB and can only be increased later, never reduced. `secretType: "environment"` injects the value at runtime only, never into the build. Restricting the group to service `web` before it exists hasn't been confirmed to work. If the call is rejected, use `"restrictions": { "restricted": false }`, or run it after step 8 and restart the service.

## Step 8: Create the service

Write `web.json`. `nf-compute-20` (0.2 vCPU, 512 MB) runs a small SvelteKit app:

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 3000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "HOST": "0.0.0.0", "PORT": "3000" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/healthz", "port": 3000,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/healthz", "port": 3000,
      "initialDelaySeconds": 20, "periodSeconds": 20, "timeoutSeconds": 2, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file web.json
```

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined` with the same body. The first build starts automatically.

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

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs (see Troubleshooting).

## Step 10: Set `ORIGIN`

Give SvelteKit its public origin. Send the **complete** `runtimeEnvironment` map, then restart:

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
  --input "$(jq -n --arg o "https://$HOST" '{runtimeEnvironment: {HOST: "0.0.0.0", PORT: "3000", ORIGIN: $o}}')"
northflank restart service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json
```

Then run the step 9 loop again. `adapter-node` uses `ORIGIN` as the trusted origin for CSRF checks on form submissions. The alternative, `PROTOCOL_HEADER`/`HOST_HEADER`, depends on which forwarded headers the ingress sets, which hasn't been confirmed, so this guide uses `ORIGIN`. When you add a custom domain, update `ORIGIN` to it.

## How to know it worked

You're done when all of these hold. Check 5 applies only if you added the database in step 7.

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health route returns 200:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/healthz"
   ```
3. The render-check page from step 5 is rendered on the server for each request. Both lines should show a timestamp, the timestamps should differ, and `SSR_OK` should print:
   ```bash
   a=$(curl -s "https://$HOST/render-check" | grep -oE 'data-rendered-at="[^"]+"'); sleep 2
   b=$(curl -s "https://$HOST/render-check" | grep -oE 'data-rendered-at="[^"]+"')
   echo "$a"; echo "$b"
   [ -n "$a" ] && [ -n "$b" ] && [ "$a" != "$b" ] && echo SSR_OK
   ```
   If both lines are empty, the page isn't deployed or a layout sets `export const ssr = false`. If they're identical, a layout sets `export const prerender = true`; keep `prerender = false` in the page.
4. `ORIGIN` is set in the running container:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId web \
     --shell-cmd 'sh -c' --cmd '"printenv ORIGIN"'
   ```
   On Linux use `script -qefc '<command>' /dev/null`. Without `script`, `exec` may lose its output when there's no TTY.
5. `DATABASE_URL` is present (prints the scheme only):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId web \
     --shell-cmd 'sh -c' --cmd '"printenv DATABASE_URL | cut -c1-13"'
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Cross-site POST form submissions are forbidden` | `ORIGIN` missing or doesn't match the URL in the browser | Set `ORIGIN` as in step 10; update it after adding a custom domain |
| `Cannot find module '/app/build'` | Still on `adapter-auto`, or custom `out` directory | Install and configure `adapter-node`; copy the right directory |
| Health check fails, container restarts | Server bound to localhost or wrong port | Keep `HOST=0.0.0.0`, `PORT=3000` and `internalPort: 3000` in sync |
| Render-check timestamps are identical | The page was prerendered at build time | Keep `export const prerender = false` in `+page.server.js`, push, rebuild |
| Build fails reading a secret | Secret read at build time (static import or prerender) | Read it at runtime (`$app/env/private` dynamic, `$env/dynamic/private`, or `process.env`) |
| `413` on uploads | `BODY_SIZE_LIMIT` defaults to `512kb` | Add `BODY_SIZE_LIMIT` (for example `10M`) to `runtimeEnvironment`, restart |
| `DATABASE_URL` empty | Secret group restricted to another ID or created after the container started | Fix `restrictions`, then `northflank restart service` |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

Logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web` and `northflank get service logs --projectId "$PROJECT_ID" --serviceId web --lineLimit 200`.

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print `DATABASE_URL` or addon passwords. Print key names or truncated values only.
- Ask before changing the adapter in a project that targets another platform; the switch from `adapter-auto` changes how it builds everywhere. Tell the user about the health route and render-check page you add.
- Addon storage can only grow, and the project region is fixed at creation. Confirm both with the user.
- Don't delete the service or addon to "start fresh" without explicit confirmation for that resource. Patch and rebuild instead.

## FAQ

### Where should I deploy my SvelteKit app?
If the app is fully prerendered, any static host works. If it uses SSR, form actions or API routes, choose a serverless platform with a matching SvelteKit adapter, or self-host with `adapter-node` on a container platform, a VM or Kubernetes. Self-hosting with Docker gives you one portable image that can sit next to its database.

### Do I need `ORIGIN` if I set `PROTOCOL_HEADER` and `HOST_HEADER`?
No, either works according to the `adapter-node` docs, but the header approach only works if the proxy in front sets those headers. `ORIGIN` is explicit and doesn't depend on the ingress.

### Can I run more than one instance?
Yes. `adapter-node` keeps no session state in memory by default, so raise `deployment.instances`. If you add in-memory caches or sessions, move them to a shared store first.

### Does `node build` handle graceful shutdown?
Yes. On `SIGTERM` it stops accepting connections and waits up to `SHUTDOWN_TIMEOUT` seconds (default 30) for open requests.

### Do public variables need a rebuild?
Only if they're inlined at build time (`static: true` on SvelteKit 3, `$env/static/public` on SvelteKit 2). Dynamic ones change with a runtime variable and a restart.

## Conclusion

You now have a SvelteKit app running as an `adapter-node` server in a container, built from Git on every push, with a health check, a correct `ORIGIN` for form actions, and, optionally, a managed PostgreSQL database wired in as `DATABASE_URL`.

**Next:** compare the setup with [the Nuxt guide](deploy-nuxt-app.md) or [the Next.js guide](deploy-nextjs-app.md), or add a cache with [how to deploy Redis](deploy-redis.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy a Nuxt app](deploy-nuxt-app.md)
- [How to deploy a Next.js app](deploy-nextjs-app.md)
