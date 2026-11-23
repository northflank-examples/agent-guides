<!--
agent-guide-meta:
  id: NF-STK-REACT
  description: "How to deploy a Vite React app with Docker and nginx: SPA fallback for deep links, VITE_* values as build arguments, and a public HTTPS URL verified."
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
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/domains/domains-on-northflank
    - https://northflank.com/stacks/deploy-create-react-app
    - https://vite.dev/guide/env-and-mode
    - https://create-react-app.dev/docs/adding-custom-environment-variables/
    - https://react.dev/blog/2025/02/14/sunsetting-create-react-app
    - https://hub.docker.com/_/nginx
  related: [NF-DEP-001, NF-DEP-004, NF-DEP-005, NF-DEP-012, NF-STK-VUE]
-->

# How to deploy a Vite React app

To deploy a Vite React app, build it with `npm run build` in a Node stage of a Docker image, serve the `dist/` output with nginx and an `index.html` fallback for client-side routes, and pass every `VITE_*` variable as a build argument. Vite compiles those values into the bundle at build time, so setting them on the running container does nothing, and without the fallback, deep links return 404.

This guide does it on Northflank with the CLI: one service built from Git, with nginx serving `/healthz` on port 8080, caching hashed assets and returning a real 404 for missing ones. You end with a public HTTPS URL, verified by `curl` checks on deep links, missing assets and the bundle contents. Every step is a command a developer or coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a Vite React app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Vite React app and confirm the team, repository, branch and project before changing anything.
3. Give Northflank access to your repository on the Git integrations page (once).
4. Add a two-stage `Dockerfile` (Node builds `dist/`, nginx serves it) and an `nginx.conf` with an `index.html` fallback and a `/healthz` route on port 8080, then push.
5. Create a project and a service built from your repo with `northflank put service combined --file web.json`. Pass any `VITE_*` values as `buildArguments`.
6. Wait for the deploy, then check that the live build is your pushed commit and that the homepage, its assets and a deep link like `/some/route` load.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your Vite React app on Northflank

A Vite React build is static files, so the host needs to serve them over HTTPS, fall back to `index.html` for client-side routes, and get build-time variables into the build. Static hosting services and CDNs are a natural fit for that alone. A container platform fits when you want the frontend built from Git next to its API and database, with your own nginx config and health check. A raw VM or Kubernetes cluster works too, but you install and maintain the web server and TLS yourself.

### Why build in a container and serve with nginx

A Vite build produces plain HTML, JavaScript and CSS, so the final image is a small nginx image holding `dist/`, with no Node at runtime. Building inside the `Dockerfile` keeps it reproducible: the same commit and build arguments give the same bundle.

Vite only exposes variables that start with `VITE_` to client code, through `import.meta.env`, and it replaces them statically at build time. Its docs also note that variables already present in the environment when Vite runs take priority over `.env` files, which is what lets a Docker build argument set them. On Northflank, build arguments are passed to the `Dockerfile` with `--build-arg` and don't persist in the final image. Runtime variables only reach the running nginx container, which ignores them.

If you prefer the UI, Northflank has a one-click [Create React App stack template](https://northflank.com/stacks/deploy-create-react-app). The steps below work for any React + Vite repository.

## Prerequisites

- **A Vite React app that builds locally:**
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
jq -r '"react: \(.dependencies.react // "missing")  vite: \(.devDependencies.vite // .dependencies.vite // "missing")"' package.json
ls package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId react-app -o json >/dev/null 2>&1 \
  && echo "project: react-app exists and will be reused" \
  || echo "project: react-app will be created"
```

- If `react` or `vite` says `missing`, this isn't a Vite React app. Stop and use a guide for your framework.
- The lockfile tells you the package manager. Step 5 assumes `package-lock.json`; for pnpm or yarn, swap the install command.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "react-app", "region": "europe-west" }'
export PROJECT_ID=react-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Add the `Dockerfile`, `nginx.conf` and `.dockerignore`

`nginx.conf` replaces the default server block of the official nginx image. It listens on 8080, answers health checks, caches Vite's hashed files in `/assets/` for a year, never falls back to `index.html` for a missing asset, and serves `index.html` for every other unknown path so React Router can handle it:

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

The `Dockerfile` builds in Node and copies only `dist/` into nginx. Add one `ARG` line per `VITE_*` variable in the **build** stage. An `ARG` is visible as an environment variable to the `RUN` commands after it in that stage, which is all Vite needs:

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

The nginx image already has a start command, so there's no `CMD`. Add a `.dockerignore` so local files don't leak into the build context:

```
node_modules
dist
.git
.env.local
.env.*.local
```

A few details:

- For pnpm or yarn, copy that lockfile and swap the install command.
- Keep each `VITE_*` value in one place. If it lives in a committed `.env.production`, don't also declare it as an `ARG`: an environment value wins over the file, and it hasn't been confirmed whether an `ARG` with no value stays unset or becomes an empty string.
- If the app is served under a sub-path, set `base` in `vite.config.*` and `basename` in React Router. This guide assumes the root path.
- If the app reads no `VITE_*` variables (the prerequisite grep prints nothing), leave out the `ARG` line.

Northflank builds what's in your repository, not the files on your machine, so commit and push them:

```bash
git add Dockerfile nginx.conf .dockerignore
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create the service

Write `web.json`. `buildArguments` holds the public build-time values. There's no `runtimeEnvironment`, because nginx doesn't read it. If the app reads no `VITE_*` variables, leave out `buildArguments`.

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

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined` with the same body. The first build starts automatically. For a monorepo, point `dockerFilePath` and `dockerWorkDir` at the app folder (for example `/apps/web/Dockerfile` and `/apps/web`).

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

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web`. The troubleshooting table covers the common causes.

## Step 8 (optional): Change a build-time value

Only needed if the app reads `VITE_*` variables. A new `VITE_*` value needs a new build. Patch the **complete** `buildArguments` map, then start a build:

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json --input '{
  "buildArguments": { "VITE_API_URL": "https://api.new-domain.com" }
}'
northflank start service build --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
  --input '{ "branch": "main" }'
```

Then run the step 7 loop again. To try a value for one build without saving it, pass it in `overrides.buildArguments` in the `start service build` body; it takes precedence for that build only. Don't use the older `build-arguments` endpoint, which is deprecated.

If several services share a build-time value, put it in a secret group with `secretType: "arguments"` instead. Public values still end up in the bundle either way.

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
4. A deep link returns the app shell (200, HTML), not a 404:
   ```bash
   curl -s -o /dev/null -w '%{http_code} %{content_type}\n' "https://$HOST/some/deep/route"
   ```
5. A missing asset returns a real 404, not `index.html`:
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
   At least one `FOUND` line should print. If the value is only used in a lazy-loaded route, it may sit in a chunk that `index.html` doesn't reference; open that route in a browser to confirm instead.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `import.meta.env.VITE_X` is `undefined` | Set as a runtime variable, or `ARG` missing from the build stage | Add `ARG VITE_X` before `RUN npm run build`, set it in `buildArguments`, rebuild |
| Value is empty though `.env.production` has it | The same name is also declared as an `ARG` (possibly passed empty) | Keep the value in one place: `buildArguments` or the file, not both |
| 404 on refresh of `/some/route` | No SPA fallback | Keep `try_files $uri $uri/ /index.html` in `location /` |
| Blank page, JS served as `text/html` | Asset path wrong (sub-path deploy) and fallback served `index.html` | Set Vite `base` to match the URL path |
| Health check fails | nginx on port 80, service checks 8080 | Copy `nginx.conf` to `/etc/nginx/conf.d/default.conf` with `listen 8080` |
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
- Never put a secret in a `VITE_*` variable or in `buildArguments`. Everything there ships to every browser. If the app needs a secret, it belongs in a backend.
- Show the user the `nginx.conf` and `Dockerfile` you add, and don't change `vite.config.*` beyond `base` without asking.
- After changing a build-time value, run check 6. A green deploy doesn't prove the bundle has the new value.
- Don't delete the service to start again. Patch it and rebuild.

## FAQ

### Where should I deploy my Vite React app?
Any host that serves static files over HTTPS with an `index.html` fallback can run it, from static hosting services to a container platform or a VM. Pick a static host if the frontend stands alone. Pick a container platform if you want a `Dockerfile` build, your own nginx config, and the frontend deployed next to its API and database.

### What changes for Create React App?
Create React App is deprecated for new apps and only in maintenance mode, and the React team suggests migrating to a framework or to a build tool like Vite. If you still deploy one, the same setup works with three changes: variables must start with `REACT_APP_` (read with `process.env.REACT_APP_X`), the output folder is `build/` instead of `dist/`, and hashed files live under `/static/`, so rename the `/assets/` location. Values are embedded at build time, exactly like Vite's.

### Can I change `VITE_*` values without rebuilding?
Not with the standard build. One common pattern is to fetch a small JSON config from the server at startup; another is to have nginx write one from runtime variables. Either way, it's your app code reading it, not `import.meta.env`.

### How do I put the app on my own domain?
Add and verify the domain, then link it to the service's `http` port. See [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank).

## Conclusion

You now have a React + Vite app built from Git in a container, served by nginx with SPA fallback and long-lived caching for hashed assets, and with build-time values set through build arguments rather than baked into the repo.

**Next:** deploy the backend it talks to, add a custom domain, or scale `deployment.instances` if traffic grows. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a full stack app](deploy-full-stack-app.md)
- [How to deploy a Vue app](deploy-vue-app.md)
- [How to deploy a Next.js app](deploy-nextjs-app.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
