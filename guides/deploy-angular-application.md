<!--
agent-guide-meta:
  id: NF-STK-ANGULAR
  description: "How to deploy an Angular application in Docker: build with ng build, serve with nginx and an SPA fallback, and inject API URLs at build time with define."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/start-service-build
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/inject-build-arguments
    - https://northflank.com/docs/v1/application/run/inject-runtime-variables
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/stacks/deploy-angular
    - https://angular.dev/tools/cli/deployment
    - https://angular.dev/tools/cli/build-system-migration
    - https://angular.dev/tools/cli/environments
    - https://angular.dev/guide/ssr
    - https://angular.dev/best-practices/security
    - https://github.com/angular/angular-cli/blob/main/packages/schematics/angular/ssr/files/application-builder/server.ts.template
  related: [NF-DEP-001, NF-DEP-004, NF-DEP-012, NF-STK-REACT, NF-STK-VUE]
-->

# How to deploy an Angular application

To deploy an Angular application, build it with `ng build` in a Node container stage, serve the `browser/` output with nginx and a `try_files` fallback to `index.html`, and inject per-deployment values such as an API URL at build time through Angular's `define` option. Angular has no runtime environment variables in the browser, and without the fallback, deep links return 404 on refresh.

This guide does it on Northflank with the CLI: a two-stage Dockerfile, a build argument for `API_URL` and one service deployed from Git, with an optional Angular SSR section. You end with a public HTTPS URL where deep links load, missing scripts return 404 and the build-time value is in the bundle, each checked with `curl`. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run; a human links a private Git repo once.

## TL;DR: how to deploy an Angular application

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is an Angular app and confirm the team, repository, branch and project before changing anything, then give Northflank access to your repository (once).
3. Wire `API_URL` in with `define`, and add a two-stage `Dockerfile` (Node runs `ng build --define`, nginx serves `dist/<project>/browser`) and an `nginx.conf` with an `index.html` fallback and `/healthz` on port 8080, then push.
4. Create a project and a service built from your repo with `northflank put service combined --file web.json`, with the API URL in `buildArguments`.
5. Wait for the deploy, then check that the live build is your pushed commit and that the homepage, its scripts and stylesheets, and a deep link load.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your Angular application on Northflank

A production Angular build is static files, so the host needs to serve them over HTTPS, fall back to `index.html` for client-side routes, and get build-time variables into the build. Static hosting services and CDNs are a natural fit for that alone. A container platform fits when you want the frontend built from Git next to its API and database, with your own nginx config and health check. A raw VM or Kubernetes cluster works too, but you install and maintain the web server and TLS yourself.

### Why build in a container and serve with nginx

With the application builder (Angular 17 and later), `ng build` writes the browser bundle to `dist/<project-name>/browser`. Angular's deployment docs say you copy that folder to a web server and, if you use the router, configure the server to return `index.html` for files that don't exist. nginx does both in a tiny image, with no Node at runtime.

For configuration, Angular offers `environment.ts` files swapped by `fileReplacements`, which are committed to the repo, and a `define` option whose command-line form Angular recommends for values that change per build. Using `--define` with a Docker build argument keeps URLs out of the repo. On Northflank, build arguments are passed with `--build-arg` and don't persist in the final image.

Prefer the UI? Northflank has a one-click [Angular stack template](https://northflank.com/stacks/deploy-angular).

## Prerequisites

- **An Angular app that builds locally with the application builder** and writes `browser/`:
  ```bash
  npm ci && npx ng build && ls -d dist/*/browser && echo BUILD_OK
  ```
  If there's no `browser/` folder, the project uses the older browser builder; copy `dist/<project-name>` instead in step 6.
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
jq -r '"angular: \(.dependencies["@angular/core"] // "missing")  builder: \(.devDependencies["@angular/build"] // .devDependencies["@angular-devkit/build-angular"] // "missing")  ssr: \(.dependencies["@angular/ssr"] // "no")"' package.json
ls package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId angular-app -o json >/dev/null 2>&1 \
  && echo "project: angular-app exists and will be reused" \
  || echo "project: angular-app will be created"
```

- If `angular` says `missing`, this isn't an Angular app. Stop and use a guide for your framework.
- If `ssr` shows a version, the app uses server-side rendering: follow the optional SSR section after step 9 as well.
- The lockfile tells you the package manager. Step 6 assumes `package-lock.json`; for pnpm or yarn, swap the install command.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "angular-app", "region": "europe-west" }'
export PROJECT_ID=angular-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Wire a build-time constant into the app

Declare the constant so TypeScript accepts it, for example in `src/types.d.ts`:

```ts
declare const API_URL: string;
```

Use it from your environment file, so the rest of the app is unchanged:

```ts
// src/environments/environment.ts
export const environment = { apiUrl: API_URL };
```

Give it a local default in `angular.json` under `projects.<name>.architect.build.options`, so `ng serve` keeps working. The string value needs inner single quotes:

```json
"define": { "API_URL": "'http://localhost:3000'" }
```

Angular's docs say a `--define` on the command line takes precedence over `angular.json`, which is what the container build uses.

## Step 6: Add the `Dockerfile`, `nginx.conf` and `.dockerignore`

Angular's production build puts hashed `main-*.js`, `chunk-*.js` and `styles-*.css` files at the root of `browser/`, not in an `/assets/` folder. So `nginx.conf` matches by extension: JavaScript and CSS never fall back to `index.html` and are cached for a year, while everything else gets the SPA fallback.

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

  location ~* \.(?:js|css)$ {
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

If you've turned output hashing off, or serve unhashed `.js` files from `public/`, drop the `immutable` header.

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ARG API_URL
RUN npx ng build --define "API_URL='${API_URL}'" \
 && mkdir -p /out && cp -r dist/*/browser/. /out/

FROM nginx:stable-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /out /usr/share/nginx/html
EXPOSE 8080
```

The `cp` assumes one application in `dist/`. In a multi-project workspace, build with `npx ng build <project-name>` and copy `dist/<project-name>/browser`. Add a `.dockerignore` with `node_modules`, `dist`, `.angular` and `.git`.

Northflank builds what's in your repository, not the files on your machine, so commit and push them:

```bash
git add Dockerfile nginx.conf .dockerignore angular.json src/   # step 5 changes plus the files above
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Create the service

Write `web.json`. `buildArguments` holds the build-time API URL. There's no `runtimeEnvironment`, because nginx doesn't read it.

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-10" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "buildArguments": { "API_URL": "https://api.example.com" },
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

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined` with the same body. The first build starts automatically.

## Step 8: Wait for the deploy and get your URL

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

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web`. A budget error there comes from the `budgets` in `angular.json`, not the platform. The troubleshooting table covers the other common causes.

## Step 9 (optional): Change the API URL

A new `API_URL` needs a new build; a restart won't do it. Patch the **complete** `buildArguments` map, then start a build:

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
  --input '{ "buildArguments": { "API_URL": "https://api.new-domain.com" } }'
northflank start service build --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
  --input '{ "branch": "main" }'
```

Then run the step 8 loop again. Don't use the deprecated `build-arguments` endpoint.

## Optional: Angular SSR on a Node server

If the project uses server-side rendering (`ng new --ssr` or `ng add @angular/ssr`), the build also produces `dist/<project-name>/server/server.mjs`, an Express server that listens on `PORT` (default 4000). Add a health route in `server.ts` (in `src/` or the project root, depending on the Angular version) before the static and Angular handlers:

```ts
app.get('/healthz', (_req, res) => { res.send('ok'); });
```

Replace the nginx stage with a Node runtime:

```dockerfile
FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production PORT=4000
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
EXPOSE 4000
CMD ["node", "dist/<project-name>/server/server.mjs"]
```

Drop the `mkdir`/`cp` line from the build stage. In `web.json`, use `nf-compute-20` or larger, port and health checks on 4000, and `"runtimeEnvironment": { "PORT": "4000" }`. Server-only secrets go in runtime variables or a secret group, never in build arguments.

Angular's security guide says requests with a host that isn't allowed get `400 Bad Request`, and the Node server reads allowed hosts from `NG_ALLOWED_HOSTS`. After the first deploy, add the public host (send the complete map) and restart:

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
  --input "{ \"runtimeEnvironment\": { \"PORT\": \"4000\", \"NG_ALLOWED_HOSTS\": \"$HOST\" } }"
northflank restart service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json
```

## How to know it worked

You're done when all of these hold. For SSR, skip check 5, and check 6 also applies.

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
3. The homepage and every script and stylesheet it references load. Angular references its hashed files (`main-*.js`, `polyfills-*.js`, `styles-*.css`) relative to `<base href="/">`. Each line should start with `200`:
   ```bash
   curl -s "https://$HOST/" | grep -oE '(src|href)="[^"]+\.(js|css)"' | sed -E 's/^(src|href)="//; s/"$//' \
     | grep -v '^https\?://' | sed 's|^/*|/|' | sort -u | while read -r p; do
     echo "$(curl -s -o /dev/null -w '%{http_code}' "https://$HOST$p") $p"
   done
   ```
   If nothing prints, the homepage isn't serving the Angular build: open `https://$HOST/` and check what it returns.
4. A deep link returns the app shell (200, HTML), not a 404:
   ```bash
   curl -s -o /dev/null -w '%{http_code} %{content_type}\n' "https://$HOST/orders/42"
   ```
5. A missing script returns a real 404, not `index.html`:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/main-missing.js"
   ```
6. The build-time value is in the bundle (skip this if the app doesn't use `API_URL`):
   ```bash
   EXPECTED='https://api.example.com'
   curl -s "https://$HOST/" | grep -oE 'src="[^"]+\.js"' | sed -E 's/^src="//; s/"$//; s|^/*|/|' | sort -u | while read -r p; do
     curl -s "https://$HOST$p" | grep -qF "$EXPECTED" && echo "FOUND in $p"
   done
   ```
   At least one `FOUND` line should print. Values used only in lazy-loaded routes live in chunks that `index.html` doesn't list; open that route in a browser to confirm instead.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| 404 when refreshing a route | No fallback | Keep `try_files $uri $uri/ /index.html` |
| `Cannot find name 'API_URL'` | Missing declaration | Add `declare const API_URL: string;` in a `.d.ts` under `src/` |
| `API_URL` is the localhost default | Build argument not set | Set it in `buildArguments`, rebuild |
| `COPY failed: … /out` or empty site | No `browser/` folder (browser builder) | Copy `dist/<project-name>` instead |
| Build killed | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| SSR returns 400 | Host not allowed | Set `NG_ALLOWED_HOSTS`, restart |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never put secrets in `define`, `environment.ts` or `buildArguments`. Angular's docs warn that environment files are visible to anyone who loads the page.
- Show the user any change to `angular.json`, `src/types.d.ts` or `server.ts`.
- Don't set `NG_ALLOWED_HOSTS` to `*`; Angular discourages it. List real hostnames.
- After changing a value, run check 6, then patch and rebuild rather than deleting the service.

## FAQ

### Where should I deploy my Angular application?
A browser-only Angular build runs on any host that serves static files over HTTPS with an `index.html` fallback: a static hosting service, a container platform or a server with nginx. Angular SSR needs a host that runs Node.js instead. A container platform covers both cases with a `Dockerfile`, and keeps the app next to its API.

### Can I keep using `environment.prod.ts`?
Yes. If the values are the same for every deployment, `fileReplacements` is enough and you need no build arguments.

### How do I add a custom domain?
Add and verify the domain, then link it to the service's port. See [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank). For SSR, add that hostname to `NG_ALLOWED_HOSTS` as well.

### SPA or SSR?
Start with the SPA unless you need server-rendered HTML for SEO or first paint. It's cheaper and simpler to run.

## Conclusion

You now have an Angular app built from Git in a container, served by nginx with route fallback and long-lived caching for hashed files, with per-deployment values injected at build time. You also know how to switch to a Node runtime for SSR.

**Next:** deploy the API the app calls and add your own domain. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a NestJS app](deploy-nestjs-app.md)
- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a full stack app](deploy-full-stack-app.md)
- [How to deploy a Vite React app](deploy-vite-react-app.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
