<!--
agent-guide-meta:
  id: NF-STK-ASTRO
  description: "How to deploy an Astro website with Docker: static output on nginx with proper 404s, or SSR with the Node adapter, plus build-time PUBLIC_ variables."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/start-service-build
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/inject-build-arguments
    - https://northflank.com/docs/v1/application/run/inject-runtime-variables
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/domains/domains-on-northflank
    - https://northflank.com/stacks/deploy-astro
    - https://docs.astro.build/en/recipes/docker/
    - https://docs.astro.build/en/guides/integrations-guide/node/
    - https://docs.astro.build/en/guides/environment-variables/
  related: [NF-DEP-001, NF-DEP-004, NF-DEP-005, NF-STK-REACT, NF-STK-VUE]
-->

# How to deploy an Astro website

To deploy an Astro website, build it with `npm run build` and serve the static `dist/` output from a web server that returns `404.html` for unknown paths, passing `PUBLIC_*` values in at build time. For pages with on-demand rendering, add the `@astrojs/node` adapter and run `node ./dist/server/entry.mjs` bound to `0.0.0.0`, with server secrets read at runtime. Setting `PUBLIC_*` variables at runtime is the usual mistake: Astro compiles them in.

This guide does it on Northflank with the CLI as one service built from Git: nginx for static output, then the Node adapter for server rendering. You end with a public HTTPS URL where pages return 200, unknown paths return your 404 page and build-time values appear in the HTML, each checked with `curl`. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run; a human links a private Git repo once.

## TL;DR: how to deploy an Astro website

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is an Astro site, see whether it's static or SSR, and confirm the team, repository, branch and project before changing anything. Give Northflank access to your repository (once).
3. Add a two-stage `Dockerfile` (Node runs `npm run build`, nginx serves `dist/`) and an `nginx.conf` with `try_files $uri $uri/index.html =404`, `error_page 404 /404.html` and `/healthz` on port 8080, then push. For SSR, use the Node adapter's standalone server on port 4321 instead.
4. Create a project and a service built from your repo with `northflank put service combined --file web.json`, with each `PUBLIC_*` value in `buildArguments`.
5. Wait for the deploy, then check that the live build is your pushed commit, that pages and their `/_astro/` files load and that unknown paths return 404.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your Astro website on Northflank

An Astro site's hosting needs depend on its output. A fully static build only needs a web server that serves HTML over HTTPS and returns your `404.html`, so static hosting services and CDNs fit it well. Once pages use on-demand rendering, the host has to run a Node server, which means a serverless platform with an Astro adapter, a container platform or a VM. A container platform runs either shape from one `Dockerfile`; a raw VM or Kubernetes cluster works too if you manage the web server and TLS yourself.

### Why containerize an Astro site

Astro's Docker recipe gives two shapes: an nginx image serving `dist/` for static sites, and a Node image running the adapter's entry file for server rendering. Building inside the `Dockerfile` means the same commit and build arguments always produce the same site, and moving from static to SSR changes only the final stage.

Unlike a single-page app, a static Astro site has one HTML file per page, so unknown paths shouldn't fall back to `index.html`. They should get your `404.html`. Astro exposes only `PUBLIC_`-prefixed variables to client code, and `import.meta.env` values are replaced statically at build time. In SSR, server code can read real environment variables at runtime. On Northflank, build arguments reach the `Dockerfile` with `--build-arg` and don't persist in the final image, while runtime variables reach the running container.

Prefer the UI? Northflank has a one-click [Astro stack template](https://northflank.com/stacks/deploy-astro).

## Prerequisites

- **An Astro site that builds locally:**
  ```bash
  npm ci && npm run build && ls dist && echo BUILD_OK
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

Before changing anything, confirm that this guide fits the app, find out whether it's static or server-rendered, and print the deployment target. Run this from the repository root:

```bash
jq -r '"astro: \(.dependencies.astro // .devDependencies.astro // "missing")  node adapter: \(.dependencies["@astrojs/node"] // "none")"' package.json
grep -nE "adapter|output" astro.config.* ; grep -rln "prerender = false" src
grep -rhoE 'import\.meta\.env\.[A-Z0-9_]+' src | sort -u
ls package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId astro-site -o json >/dev/null 2>&1 \
  && echo "project: astro-site exists and will be reused" \
  || echo "project: astro-site will be created"
```

- If `astro` says `missing`, this isn't an Astro site. Stop and use a guide for your framework.
- If the config has no adapter and no page sets `prerender = false`, the site is static: follow steps 3 to 8. If any page sets `prerender = false`, or the config sets `output: 'server'`, you need SSR: also follow step 9.
- The `import.meta.env` list shows your build-time variables. `PUBLIC_*` ones go in `buildArguments`.
- The lockfile tells you the package manager. Step 5 assumes `package-lock.json`; for pnpm or yarn, swap the install command.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another), static or SSR, and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "astro-site", "region": "europe-west" }'
export PROJECT_ID=astro-site

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Add the static `Dockerfile` and `nginx.conf`

This follows Astro's Docker recipe. Astro writes each page as `<path>/index.html` and puts hashed assets in `/_astro/`:

```nginx
server {
  listen 8080;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;
  error_page 404 /404.html;

  location = /healthz {
    access_log off;
    default_type text/plain;
    return 200 'ok';
  }

  location /_astro/ {
    try_files $uri =404;
    add_header Cache-Control "public, max-age=31536000, immutable";
  }

  location / {
    try_files $uri $uri/index.html =404;
  }

  gzip on;
  gzip_types text/css application/javascript application/json image/svg+xml;
}
```

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ARG PUBLIC_SITE_URL
RUN npm run build

FROM nginx:stable-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 8080
```

Add one `ARG` per build-time variable before `RUN npm run build`; each `ARG` is visible to that `RUN` as an environment variable. In static mode, that includes non-public values the build itself uses, such as a CMS token for fetching content. Those stay in the build stage and don't reach the nginx image, but a non-public value rendered into a page still ends up in that page's HTML. If the site reads no build-time variables, leave out the `ARG` line. Add a `.dockerignore` with `node_modules`, `dist`, `.astro`, `.env` and `.git`.

Northflank builds what's in your repository, not the files on your machine, so commit and push them:

```bash
git add Dockerfile nginx.conf .dockerignore
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create the service

Write `web.json`. `buildArguments` holds the build-time values; if the site reads none, leave it out.

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-10" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "buildArguments": { "PUBLIC_SITE_URL": "https://example.com" },
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

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined` with the same body. The first build starts automatically. For a build-time secret like a CMS token, use a secret group with `secretType: "arguments"` instead of `buildArguments`, so it isn't stored in the service spec.

## Step 7: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds and prints your URL when the site is live. You can also watch the build in the Northflank dashboard.

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

Content and build-time values only change with a new build. Push to the branch, or patch the **complete** `buildArguments` map and start a build:

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
  --input '{ "buildArguments": { "PUBLIC_SITE_URL": "https://www.example.com" } }'
northflank start service build --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
  --input '{ "branch": "main" }'
```

Then run the step 7 loop again. Don't use the deprecated `build-arguments` endpoint.

## Step 9 (SSR only): Switch to the Node adapter

Add the adapter. Astro's docs give `npx astro add node`, which installs `@astrojs/node` and updates the config. Use `standalone` mode, which starts its own server and serves static files too:

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import node from '@astrojs/node';
export default defineConfig({
  output: 'server',            // or keep the default and set `prerender = false` per page
  adapter: node({ mode: 'standalone' }),
});
```

Add a health route, `src/pages/healthz.ts`:

```ts
export const prerender = false;
export const GET = () => new Response('ok');
```

Add a tiny page that renders a value only the server can produce on each request, `src/pages/rendered-at.astro`. Check 6 uses it to prove pages are rendered per request:

```astro
---
export const prerender = false;
const renderedAt = new Date().toISOString();
---
<p id="rendered-at">{renderedAt}</p>
```

Replace the nginx stage with a Node runtime, following Astro's recipe:

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
ENV HOST=0.0.0.0 PORT=4321
EXPOSE 4321
CMD ["node", "./dist/server/entry.mjs"]
```

Then update `web.json`: `deploymentPlan` `nf-compute-20` or larger, `internalPort` and both health check ports `4321`, and `"runtimeEnvironment": { "HOST": "0.0.0.0", "PORT": "4321" }`. Commit and push the changed files as in step 5 (`git add astro.config.mjs src/pages/healthz.ts src/pages/rendered-at.astro Dockerfile package.json package-lock.json`, then commit and `git push`), then re-run `northflank put service combined` and the step 7 loop.

Server secrets (database URLs, API keys) are read at runtime, through `process.env` or `astro:env/server` with `access: "secret"`. Put them in a secret group with `secretType: "environment"`, never in build arguments:

```bash
jq -n --arg k "$API_KEY" '{ name: "web-runtime", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "web", type: "service" } ] },
  secrets: { variables: { API_KEY: $k } } }' > web-runtime.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file web-runtime.json >/dev/null && rm web-runtime.json
northflank restart service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json
```

`PUBLIC_*` values stay build arguments in SSR too.

## How to know it worked

You're done when checks 1 to 5 hold. Run check 6 only for SSR, and check 7 only if the site renders a `PUBLIC_*` value into the home page.

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
3. The homepage and every script and stylesheet it references from `/_astro/` load. Each line should start with `200`:
   ```bash
   curl -s "https://$HOST/" | grep -oE '/_astro/[^"]+\.(js|css)' | sort -u | while read -r p; do
     echo "$(curl -s -o /dev/null -w '%{http_code}' "https://$HOST$p") $p"
   done
   ```
   If nothing prints, the page may inline all its CSS and ship no JavaScript, which Astro does for small pages. Check that `curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"` prints `200` instead.
4. Another real page returns 200: `curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/<A_PAGE>/"`.
5. An unknown path returns 404 with your 404 page: `curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/no-such-page"` prints `404`.
6. SSR only: the server renders per request. Two requests to the page from step 9 should return different timestamps:
   ```bash
   a=$(curl -s "https://$HOST/rendered-at" | grep -oE '<p id="rendered-at">[^<]+' | cut -d'>' -f2)
   sleep 2
   b=$(curl -s "https://$HOST/rendered-at" | grep -oE '<p id="rendered-at">[^<]+' | cut -d'>' -f2)
   echo "$a"; echo "$b"
   [ -n "$a" ] && [ -n "$b" ] && [ "$a" != "$b" ] && echo SSR_OK
   ```
7. A public build-time value appears in the output: `curl -s "https://$HOST/" | grep -cF 'https://example.com'` is at least 1 if the home page renders it.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `import.meta.env.PUBLIC_X` is `undefined` | Set at runtime or `ARG` missing | Add `ARG PUBLIC_X`, set `buildArguments`, rebuild |
| `/about` 404 but `/about/` works | Directory-style output | Keep `$uri/index.html` in `try_files` |
| Unknown paths show nginx's default 404 | No `src/pages/404.astro` | Add one and rebuild |
| SSR container fails health check | Server bound to localhost or wrong port | Set `HOST=0.0.0.0` and `PORT=4321`, match port 4321 |
| `Cannot find module` at SSR start | Runtime stage lacks dependencies | Keep `npm ci --omit=dev` in the runtime stage |
| Secret is `undefined` in SSR | Set as a build argument, or group not restricted to `web` | Use `secretType: "environment"`, restart |
| Check 6 prints the same timestamp twice | The page was prerendered | Keep `export const prerender = false` in `rendered-at.astro`, and use the Node adapter |
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
- Never put secrets in `PUBLIC_*` variables or `buildArguments`. Use an `arguments` secret group for build-time tokens and an `environment` group for runtime ones, and never print their values.
- Don't switch a static site to SSR without asking. It changes the cost and the runtime.
- Show the user changes to `astro.config.*` and the files you add.
- Patch and rebuild rather than deleting the service.

## FAQ

### Where should I deploy my Astro website?
For a static Astro site, any host that serves files over HTTPS works: a static hosting service, a container platform or a server running nginx. For on-demand rendering, pick a host that runs Node.js, such as a container platform or a serverless platform with an adapter. A `Dockerfile` keeps the choice open, because moving from static to SSR changes only the final stage.

### Do I need an adapter for a static site?
No. Astro's docs say an adapter is only needed for on-demand rendering.

### Can I keep most pages static in SSR mode?
Yes. In Astro 5 and later, keep the default output and set `export const prerender = false` only on pages that need it. The standalone server serves the prerendered files too.

### How do I add my own domain?
Add and verify the domain, then link it to the service's port. See [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank). Update `site` in `astro.config.*` if canonical URLs or sitemaps use it.

## Conclusion

You now have an Astro site built from Git in a container: static output behind nginx with proper 404s and cached assets, or the Node adapter's standalone server with runtime secrets. Build-time values come from build arguments in both cases.

**Next:** add your domain, or deploy the API or CMS the site reads from. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy Strapi](deploy-strapi.md)
- [How to deploy Payload CMS](deploy-payload-cms.md)
- [How to deploy a Vite React app](deploy-vite-react-app.md)
- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
