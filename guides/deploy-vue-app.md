<!--
agent-guide-meta:
  id: NF-STK-VUE
  description: "How to deploy a Vue app with Docker and nginx: history-mode fallback for Vue Router, VITE_* values as build arguments, and a public HTTPS URL verified."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/start-service-build
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/build/inject-build-arguments
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/domains/domains-on-northflank
    - https://northflank.com/stacks/deploy-vue
    - https://vite.dev/guide/env-and-mode
    - https://router.vuejs.org/guide/essentials/history-mode.html
    - https://hub.docker.com/_/nginx
  related: [NF-DEP-001, NF-DEP-004, NF-DEP-012, NF-STK-REACT, NF-STK-ANGULAR]
-->

# How to deploy a Vue app

To deploy a Vue app built with Vite, run `npm run build` in a Node stage of a Docker image, serve the `dist/` output with nginx and an `index.html` fallback for Vue Router's history mode, and pass every `VITE_*` variable as a build argument. Without the fallback, refreshing a route like `/products/42` returns 404, and `VITE_*` values set on the running container have no effect.

This guide does it on Northflank with the CLI: one service built from Git, with nginx serving `/healthz` on port 8080, caching hashed assets and returning a real 404 for missing ones. You end with a public HTTPS URL, verified by `curl` checks on deep links, missing assets and the bundle contents. Every step is a command a developer or coding agent (Claude Code, Codex, Cursor) can run as-is; for a private repo, a human links the Git account once.

## TL;DR: how to deploy a Vue app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Vue + Vite app and confirm the team, repository, branch and project before changing anything.
3. Give Northflank access to your repository on the Git integrations page (once).
4. Add a two-stage `Dockerfile` (Node builds `dist/`, nginx serves it) and an `nginx.conf` with `try_files $uri $uri/ /index.html` and a `/healthz` route on port 8080, then push.
5. Create a project and a service built from your repo with `northflank put service combined --file web.json`. Pass any `VITE_*` values as `buildArguments`.
6. Wait for the deploy, then check that the live build is your pushed commit and that the homepage, its assets and a history-mode deep link load.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your Vue app on Northflank

A Vue build is static files, so the host needs to serve them over HTTPS, fall back to `index.html` for client-side routes, and get build-time variables into the build. Static hosting services and CDNs are a natural fit for that alone. A container platform fits when you want the frontend built from Git next to its API and database, with your own nginx config and health check. A raw VM or Kubernetes cluster works too, but you install and maintain the web server and TLS yourself.

### Why a container build and nginx for Vue

A Vue + Vite build needs no Node at runtime, so the final image is just nginx and your `dist/` folder. Building inside the `Dockerfile` keeps it reproducible: the same commit and the same build arguments give the same bundle anywhere.

Two framework rules shape the setup. Vue Router's `createWebHistory()` gives clean URLs, and its docs say plainly that without a server fallback to `index.html`, users get a 404 on direct visits. Vite only exposes `VITE_`-prefixed variables to client code, via `import.meta.env`, and replaces them statically at build time. Variables already in the environment when Vite runs take priority over `.env` files, which is how a Docker build argument sets them. On Northflank, build arguments reach the `Dockerfile` through `--build-arg` and don't persist in the final image.

Prefer the UI? Northflank has a one-click [Vue stack template](https://northflank.com/stacks/deploy-vue). The steps below work for any Vue 3 + Vite repository.

## Prerequisites

- **A Vue app that builds locally** and writes to `dist/`. In TypeScript projects from `create-vue`, `npm run build` also runs a type check, so type errors fail the build:
  ```bash
  npm ci && npm run build && test -f dist/index.html && echo BUILD_OK
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
jq -r '"vue: \(.dependencies.vue // "missing")  vite: \(.devDependencies.vite // .dependencies.vite // "missing")  nuxt: \(.dependencies.nuxt // .devDependencies.nuxt // "no")"' package.json
grep -rhoE 'import\.meta\.env\.VITE_[A-Z0-9_]+' src | sort -u
grep -rn 'createWeb' src/router 2>/dev/null
ls package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId vue-app -o json >/dev/null 2>&1 \
  && echo "project: vue-app exists and will be reused" \
  || echo "project: vue-app will be created"
```

- If `vue` or `vite` says `missing`, or `nuxt` shows a version, this isn't a client-only Vue + Vite app. Stop and use a guide for your framework.
- The `VITE_*` list shows your build-time variables. If it's empty, leave out the `ARG` line in step 5 and `buildArguments` in step 6.
- `createWebHistory` needs the fallback in step 5. `createWebHashHistory` (URLs with `#/`) works without it, but Vue Router's docs note that hash URLs hurt SEO.
- The lockfile tells you the package manager. Step 5 assumes `package-lock.json`; for pnpm or yarn, swap the install command.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "vue-app", "region": "europe-west" }'
export PROJECT_ID=vue-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Add the `Dockerfile`, `nginx.conf` and `.dockerignore`

`nginx.conf` replaces the official image's default server block. It listens on 8080, answers health checks, caches Vite's hashed `/assets/` files for a year, returns a real 404 for missing assets, and serves `index.html` for every other unknown path so Vue Router can resolve it:

```nginx
server {
  listen 8080;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;

  location = /healthz {
    access_log off;
    default_type text/plain;
    return 200 'ok';
  }

  location /assets/ {
    try_files $uri =404;
    add_header Cache-Control "public, max-age=31536000, immutable";
  }

  location / {
    try_files $uri $uri/ /index.html;
    add_header Cache-Control "no-cache";
  }

  gzip on;
  gzip_types text/css application/javascript application/json image/svg+xml;
}
```

The `Dockerfile` builds with Node and ships only `dist/`. Put one `ARG` per `VITE_*` variable in the **build** stage, before `npm run build`. Build arguments are visible as environment variables to the `RUN` steps after them in that stage:

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ARG VITE_API_URL
RUN npm run build

FROM nginx:stable-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 8080
```

The nginx image brings its own start command. Add a `.dockerignore`:

```
node_modules
dist
.git
.env.local
.env.*.local
```

Notes:

- For pnpm or yarn, copy that lockfile and change the install command.
- Keep each `VITE_*` value in one place: `buildArguments` or a committed `.env.production`, not both. An environment value wins over the file.
- To serve under a sub-path, set `base` in `vite.config.*`. The scaffolded router calls `createWebHistory(import.meta.env.BASE_URL)`, so it follows `base` automatically.

Northflank builds what's in your repository, not the files on your machine, so commit and push them:

```bash
git add Dockerfile nginx.conf .dockerignore
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create the service

Write `web.json`. Public build-time values go in `buildArguments`. There's no `runtimeEnvironment`, since nginx wouldn't read it. If the app reads no `VITE_*` variables, leave out `buildArguments`.

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-10" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "buildArguments": { "VITE_API_URL": "https://api.example.com" },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/healthz", "port": 8080,
      "initialDelaySeconds": 2, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/healthz", "port": 8080,
      "initialDelaySeconds": 10, "periodSeconds": 20, "timeoutSeconds": 2, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file web.json
```

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined` with the same body. The first build starts automatically. In a monorepo, set `dockerFilePath` and `dockerWorkDir` to the app folder.

## Step 7: Wait for the deploy and get your URL

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

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web`. A `vue-tsc` error there is a type error in your code, not a platform problem. The troubleshooting table covers the other common causes.

## Step 8 (optional): Change a build-time value

Only needed if the app reads `VITE_*` variables. New `VITE_*` values need a new build. Send the **complete** `buildArguments` map, then build:

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json --input '{
  "buildArguments": { "VITE_API_URL": "https://api.new-domain.com" }
}'
northflank start service build --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
  --input '{ "branch": "main" }'
```

Repeat the step 7 loop. For a one-off value, pass `overrides.buildArguments` in the `start service build` body; it applies to that build only. Avoid the deprecated `build-arguments` endpoint. For values shared by several services, a secret group with `secretType: "arguments"` passes them to builds.

## How to know it worked

You're done when all of these hold. Skip check 6 if the app reads no `VITE_*` variables.

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
3. The homepage and every script and stylesheet it references load. Each line should start with `200`:
   ```bash
   curl -s "https://$HOST/" | grep -oE '/assets/[^"]+\.(js|css)' | sort -u | while read -r p; do
     echo "$(curl -s -o /dev/null -w '%{http_code}' "https://$HOST$p") $p"
   done
   ```
   If nothing prints, the homepage isn't serving the Vite build: open `https://$HOST/` and check what it returns.
4. A history-mode deep link returns the app shell (200, HTML):
   ```bash
   curl -s -o /dev/null -w '%{http_code} %{content_type}\n' "https://$HOST/products/42"
   ```
5. A missing asset returns 404, not `index.html`:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/assets/does-not-exist.js"
   ```
6. The build-time value is in the bundle:
   ```bash
   EXPECTED='https://api.example.com'
   curl -s "https://$HOST/" | grep -oE '/assets/[^"]+\.js' | sort -u | while read -r p; do
     curl -s "https://$HOST$p" | grep -qF "$EXPECTED" && echo "FOUND in $p"
   done
   ```
   At least one `FOUND` line should print. Values used only in lazy-loaded route components live in chunks that `index.html` doesn't list; check those routes in a browser.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| 404 when refreshing a route | No history-mode fallback | Keep `try_files $uri $uri/ /index.html` in `location /` |
| `import.meta.env.VITE_X` is `undefined` | Set at runtime, or `ARG` missing from the build stage | Add `ARG VITE_X` before `RUN npm run build`, set `buildArguments`, rebuild |
| Build fails on `vue-tsc` | Type errors in the code | Fix locally; `npm run build` must pass |
| Blank page, assets served as HTML | Sub-path deploy without `base` | Set `base` in `vite.config.*` |
| Health check fails | nginx still on port 80 | Copy `nginx.conf` to `/etc/nginx/conf.d/default.conf` with `listen 8080` |
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
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never put a secret in a `VITE_*` variable or `buildArguments`. Those values ship to every browser.
- Don't switch the router between history and hash mode without asking; it changes every URL in the app.
- Show the user the files you add, and don't edit `vite.config.*` beyond `base` without telling them.
- After changing a build-time value, run check 6. A green deploy doesn't prove the bundle changed.
- Patch and rebuild rather than deleting the service.

## FAQ

### Where should I deploy my Vue app?
Any host that serves static files over HTTPS and falls back to `index.html` for Vue Router's history mode will work: a static hosting service, a container platform or a VM with nginx. A static host is enough for a standalone frontend. A container platform suits you if you want a `Dockerfile` build, your own nginx config, and the app next to its backend.

### Does this work for Vue 2 or the Vue CLI?
The nginx part does. Vue CLI builds to `dist/` too, but it uses webpack, exposes `VUE_APP_*` variables, and puts hashed files in `/js/` and `/css/`, so adjust the `ARG` names and the asset location.

### What about Nuxt?
Nuxt with server rendering needs a Node server, not nginx. See [how to deploy a Nuxt app](deploy-nuxt-app.md). This guide covers client-only Vue.

### Can I change `VITE_*` values without rebuilding?
Not through `import.meta.env`. If you need per-environment values without a rebuild, have the app fetch a small config file at startup.

### How do I use my own domain?
Add and verify the domain, then link it to the service's `http` port. See [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank).

## Conclusion

You now have a Vue 3 + Vite app built from Git, served by nginx with a history-mode fallback and long-lived caching for hashed assets, and build-time values that come from build arguments instead of your repo.

**Next:** deploy the API it calls, add your domain, or raise `deployment.instances` when traffic grows. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Vite React app](deploy-vite-react-app.md)
- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a FastAPI app](deploy-fastapi-app.md)
- [How to deploy a full stack app](deploy-full-stack-app.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
