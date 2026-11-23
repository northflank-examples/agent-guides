<!--
agent-guide-meta:
  id: NF-STK-NUXT
  description: "How to deploy a Nuxt app with Docker as a Nitro node-server, runtime config from NUXT_* variables, secrets in a secret group and optional PostgreSQL."
  sources:
    - https://nuxt.com/docs/4.x/getting-started/deployment
    - https://nuxt.com/docs/4.x/guide/going-further/runtime-config
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/services/list-service-builds
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/run/inject-runtime-variables
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
  related: [NF-DEP-001, NF-DB-001, NF-DEP-005, NF-STK-SVELTEKIT]
-->

# How to deploy a Nuxt app

To deploy a Nuxt app, build it with the default Nitro `node-server` preset, copy the `.output/` folder into a slim Docker image that runs `node .output/server/index.mjs` with `NITRO_HOST=0.0.0.0`, and set configuration through `NUXT_*` environment variables for keys declared in `runtimeConfig`. A built Nuxt server doesn't read `.env`, and variables for undeclared keys are ignored.

This guide does it on Northflank with the CLI: one combined service built from Git with a `/healthz` check, secrets in a runtime secret group, and an optional PostgreSQL database. You end with a public HTTPS URL serving server-rendered pages, verified by a per-request server timestamp, runtime config in the HTML and a check of secret names. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a Nuxt app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Nuxt app and confirm the team, repository, branch and project before changing anything. Give Northflank access to the repository (once).
3. Declare runtime values in `runtimeConfig`, add a `/healthz` route and a small render-check page, and push a `Dockerfile` that runs `node .output/server/index.mjs` with `NITRO_HOST=0.0.0.0` on port 3000.
4. Create a project, put secrets such as `NUXT_API_SECRET` in a secret group, and create the service with `northflank put service combined --file web.json`.
5. Wait for the deploy, then check that the live build is your pushed commit and that the render-check page shows a new server timestamp on each request.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your Nuxt app on Northflank

A Nuxt app with server rendering or server routes needs a host that runs a Node.js server and passes it runtime configuration as environment variables. A statically generated Nuxt site can go on any static host; serverless platforms run Nuxt through their own Nitro presets; a container platform or a VM runs the default `node-server` output as one long-running process. A raw VM or Kubernetes cluster works too, but you manage the OS, TLS and restarts yourself.

### Why the Nitro node-server output

Nuxt's server engine, Nitro, uses the `node-server` preset by default. `nuxt build` writes a self-contained `.output/` folder whose entry point "launches a ready-to-run Node server", so the runtime image doesn't need your source or your full `node_modules`. That fits a container well. On Northflank, one **combined service** builds that image from Git on every push and serves it behind HTTPS.

Two Nuxt rules shape the setup. First, a built server doesn't read `.env`. Second, only keys declared in `runtimeConfig` can be overridden by environment variables, and only by names in the `NUXT_` form. This guide follows both, and works for Nuxt 3 and Nuxt 4.

## Prerequisites

- **A Nuxt app that builds locally with the Node preset:**
  ```bash
  npm ci && npx nuxt build && test -f .output/server/index.mjs && echo BUILD_OK
  ```
  If `nuxt.config.ts` sets `nitro.preset` to something else (for example a serverless preset) or the app uses `nuxt generate`, set it to `node-server`.
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
jq -r '"nuxt: \(.dependencies.nuxt // .devDependencies.nuxt // "missing")"' package.json
ls package-lock.json pnpm-lock.yaml yarn.lock bun.lock bun.lockb Dockerfile 2>/dev/null
ls -d pages app/pages 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId nuxt-app -o json >/dev/null 2>&1 \
  && echo "project: nuxt-app exists and will be reused" \
  || echo "project: nuxt-app will be created"
```

- If `nuxt` says `missing`, this isn't a Nuxt app. Stop and use a guide for your framework.
- The lockfile tells you the package manager. Step 6 assumes `package-lock.json`; for pnpm, yarn or bun, swap the lockfile and install command.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- The `pages` line tells you where the render-check page goes in step 5: `pages/` (Nuxt 3), `app/pages/` (Nuxt 4), or nothing if the app has a single `app.vue`.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "nuxt-app", "region": "europe-west" }'
export PROJECT_ID=nuxt-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Declare runtime config and add a health route and a render-check page

In `nuxt.config.ts`, declare every value the app reads at runtime, with empty or safe defaults:

```ts
export default defineNuxtConfig({
  runtimeConfig: {
    apiSecret: '',          // NUXT_API_SECRET (server only)
    databaseUrl: '',        // NUXT_DATABASE_URL (server only)
    public: {
      apiBase: '',          // NUXT_PUBLIC_API_BASE (also sent to the browser)
    },
  },
});
```

Read the values with `useRuntimeConfig()`. Keys map to upper snake case with a `NUXT_` prefix: `apiSecret` → `NUXT_API_SECRET`, `public.apiBase` → `NUXT_PUBLIC_API_BASE`. Don't set defaults to a differently named variable like `process.env.OTHER_VAR`: per the Nuxt docs, that only works at build time and breaks at runtime.

Add a health route at `server/routes/healthz.ts`:

```ts
export default defineEventHandler(() => 'ok');
```

Add a render-check page that shows a value only the server can produce on each request. Check 3 reads it to prove pages are rendered per request, not prerendered. Put it at `pages/render-check.vue` (or `app/pages/render-check.vue` in Nuxt 4):

```vue
<script setup lang="ts">
const renderedAt = useState('rendered-at', () => new Date().toISOString());
</script>

<template>
  <p :data-rendered-at="renderedAt">Rendered at {{ renderedAt }}</p>
</template>
```

`useState` runs its initializer on the server during SSR and hands the value to the browser, so the browser doesn't replace it. If step 2 found no `pages` folder, don't create one (that turns on routing): put the same `<script setup>` line and `<p>` in `app.vue` and use `/` instead of `/render-check` in check 3. You can remove the page once you've verified the deploy.

## Step 6: Write the `Dockerfile` and push

```dockerfile
FROM node:22-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npx nuxt build

FROM node:22-slim
WORKDIR /app
ENV NODE_ENV=production NITRO_HOST=0.0.0.0 NITRO_PORT=3000
COPY --from=build --chown=node:node /app/.output ./.output
USER node
EXPOSE 3000
CMD ["node", ".output/server/index.mjs"]
```

The Nuxt docs say to set `NODE_ENV=production` when running the server. `.output` contains the server bundle and the dependencies it needs, so the runtime stage copies nothing else. For pnpm, yarn or bun, swap the lockfile and install command. Add a `.dockerignore` with `node_modules`, `.nuxt`, `.output` and `.env`, so a local `.env` never ends up in the image.

Northflank builds what's in your repository, not the files on your machine, so commit and push them. Use `app/pages/render-check.vue` (Nuxt 4) or `app.vue` if that's where you put the render check:

```bash
git add Dockerfile .dockerignore nuxt.config.ts server/routes/healthz.ts pages/render-check.vue
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Put secrets in a secret group (and optionally link PostgreSQL)

Skip the secret group if the app reads no secrets. Server-only values go in a secret group with `secretType: "environment"`, so they're injected at runtime and never reach the build or the image. Export the value in your shell first; the command reads it without printing it:

```bash
jq -n --arg s "$API_SECRET" '{
  name: "web-runtime", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "web", type: "service" } ] },
  secrets: { variables: { NUXT_API_SECRET: $s } }
}' > web-runtime.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file web-runtime.json >/dev/null && rm web-runtime.json
```

**Optional database.** Create a PostgreSQL addon, wait for `running`, then add an `addonDependencies` entry to the same group that links `POSTGRES_URI` under the runtime config name:

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "web-db", "type": "postgresql", "version": "16-latest", "tlsEnabled": true,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'
ADDON_OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId web-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) ADDON_OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$ADDON_OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon before linking it"; false; }
```

Then re-run the secret command with this field added to the `jq` object:

```json
"addonDependencies": [ { "addonId": "web-db", "keys": [ { "keyName": "POSTGRES_URI", "aliases": ["NUXT_DATABASE_URL", "DATABASE_URL"] } ] } ]
```

`NUXT_DATABASE_URL` fills `runtimeConfig.databaseUrl`. `DATABASE_URL` is there for ORMs that read it directly. The addon is created with `tlsEnabled: true`, so turn on TLS in the driver too (for example `ssl: 'require'` in `postgres`, `ssl: true` in `pg`, or `sslmode=require` for Prisma). Storage is in MB and can only be increased later. Restricting the group to `web` before the service exists hasn't been confirmed to work; if it's rejected, use `"restricted": false`, or run it after step 8 and restart the service.

## Step 8: Create the service

Write `web.json`. Non-secret runtime config goes in `runtimeEnvironment`:

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 3000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "NITRO_PORT": "3000", "NITRO_HOST": "0.0.0.0", "NUXT_PUBLIC_API_BASE": "https://api.example.com" },
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

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web`. The troubleshooting table covers the common causes.

## Step 10 (optional): Change a runtime config value

Patch the **complete** `runtimeEnvironment` map (or edit the secret group), then restart. No rebuild is needed, including for `NUXT_PUBLIC_*`:

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json --input '{
  "runtimeEnvironment": { "NITRO_PORT": "3000", "NITRO_HOST": "0.0.0.0", "NUXT_PUBLIC_API_BASE": "https://api.new.example.com" }
}'
northflank restart service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json
```

## How to know it worked

You're done when all of these hold. Check 4 applies only if a page renders `useRuntimeConfig().public.apiBase`, and check 5 only if you created the secret group in step 7.

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
   If both lines are empty, the page isn't deployed or SSR is off (`ssr: false`). If they're identical, the route is prerendered or cached: check `routeRules` in `nuxt.config.ts`.
4. The runtime config value reaches the page. The page that renders `useRuntimeConfig().public.apiBase` contains the value you set; this should print a number above 0:
   ```bash
   curl -s "https://$HOST/<PAGE>" | grep -c 'https://api.example.com'
   ```
5. Secrets are injected but not leaked. This prints the variable names only:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId web \
     --shell-cmd 'sh -c' --cmd '"env | cut -d= -f1 | grep ^NUXT_"'
   ```
   On Linux use `script -qefc '<command>' /dev/null`; without a TTY wrapper, `exec` may lose its output.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Cannot find module '/app/.output/server/index.mjs'` | Non-Node preset, or `nuxt generate` | Use the default `node-server` preset and `nuxt build` |
| Env variable set but `useRuntimeConfig()` returns the default | Key missing from `runtimeConfig`, or wrong name | Declare the key; use `NUXT_` + upper snake case (`NUXT_PUBLIC_` for `public`) |
| Value correct locally, missing in production | It came from `.env`, which a built server doesn't read | Set it in `runtimeEnvironment` or the secret group |
| Render-check timestamps are identical | The route is prerendered or cached by `routeRules` (`prerender`, `swr`, `isr`) | Exclude `/render-check` from those rules, push, rebuild |
| Health check fails, container restarts | Port mismatch or bound to localhost | Keep `NITRO_PORT=3000`, `NITRO_HOST=0.0.0.0` and `internalPort: 3000` in sync |
| Build `FAILURE` on `npm ci` | Lockfile missing or out of sync | Commit `package-lock.json`; read `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web` |
| Database errors | `NUXT_DATABASE_URL` not linked, or group created after the container started | Fix `addonDependencies`/`restrictions`, then `northflank restart service` |
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
- Never put secrets in `runtimeConfig.public` or a `NUXT_PUBLIC_*` variable: those values are sent to the browser.
- Never print secret values. Print variable names, as in check 5.
- Ask before changing `nitro.preset` in a project that deploys elsewhere, and tell the user about the health route and render-check page you add.
- Addon storage only grows, and the project region is fixed. Confirm both with the user. Don't delete resources without explicit per-resource confirmation.

## FAQ

### Where should I deploy my Nuxt app?
If you generate a fully static site, any static host works. If the app uses SSR or server API routes, you need a host that runs Node.js: a serverless platform with a Nitro preset, a container platform, or a VM. Building the default `node-server` output into a `Dockerfile` gives you one image that runs on any container host, next to its database if it has one.

### Do `NUXT_PUBLIC_*` changes need a rebuild?
No. Public runtime config is resolved when the server runs and passed to the client in the page payload, so a new value plus a restart is enough. Only values hard-coded at build time need a rebuild.

### Should I use `PORT` or `NITRO_PORT`?
Either. Nitro reads `NITRO_PORT` or `PORT` (default 3000) and `NITRO_HOST` or `HOST`. The Nitro docs don't state a host default, so this guide sets `NITRO_HOST=0.0.0.0` explicitly.

### Can I run several instances?
Yes, raise `deployment.instances`. Keep sessions and caches in a shared store such as Redis rather than in process memory.

### Does Nitro need HTTPS certificates?
No. The platform terminates TLS in front of the container. Leave `NITRO_SSL_CERT` and `NITRO_SSL_KEY` unset.

## Conclusion

You now have a Nuxt app running from its Nitro `.output` server in a container, built from Git on every push, with runtime config driven by `NUXT_*` variables, secrets in a secret group and an optional PostgreSQL database.

**Next:** for a client-only Vue app, see [how to deploy a Vue app](deploy-vue-app.md); for a shared cache or sessions, see [how to deploy Redis](deploy-redis.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a Vue app](deploy-vue-app.md)
- [How to deploy a SvelteKit app](deploy-sveltekit-app.md)
- [How to deploy a Next.js app](deploy-nextjs-app.md)
