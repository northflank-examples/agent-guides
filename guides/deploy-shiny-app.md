<!--
agent-guide-meta:
  id: NF-STK-SHINY
  description: "How to deploy a Shiny app with Docker: R and renv restored in the image, shiny::runApp on 0.0.0.0:3838, WebSockets over HTTPS and one sticky instance."
  sources:
    - https://rocker-project.org/images/versioned/shiny.html
    - https://github.com/rocker-org/rocker-versioned2/blob/master/scripts/init_set_env.sh
    - https://github.com/rocker-org/rocker-versioned2/blob/master/dockerfiles/r-ver_4.6.2.Dockerfile
    - https://github.com/rocker-org/rocker-versioned2/blob/master/dockerfiles/shiny_4.6.2.Dockerfile
    - https://rstudio.github.io/renv/articles/docker.html
    - https://rstudio.github.io/renv/reference/config.html
    - https://shiny.posit.co/py/get-started/deploy-on-prem.html
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/load-balancing
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-STK-STREAMLIT, NF-STK-DASH, NF-STK-GRADIO, NF-DB-001]
-->

# How to deploy a Shiny app

To deploy a Shiny app, build a Docker image from an R base image, restore packages from `renv.lock` at build time, start the app with `shiny::runApp(host = "0.0.0.0", port = 3838)`, and serve it behind a proxy that passes WebSockets through. Each session's state lives in one R process, so serverless hosts don't fit, and more than one instance needs sticky routing.

This guide does it on Northflank with the CLI: a `Dockerfile` on `rocker/r-ver` with an `renv` restore layer, built from Git, served on port 3838 with HTTP health checks and secrets in a secret group. You end with a public HTTPS URL where the app page returns 200 and the WebSocket connects. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a Shiny app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a Shiny app with a committed `renv.lock`, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Write a `Dockerfile` on `rocker/r-ver:4.6` that runs `renv::restore()` and starts `shiny::runApp("/app", host = "0.0.0.0", port = 3838)`, then push.
4. Create a project, put any API keys in a secret group, and create the service with `northflank put service combined --file shiny.json` (HTTP port 3838, one instance).
5. Wait for the deploy, then check that the live build is your pushed commit, the page returns 200 and the app connects in a browser.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why a container for Shiny?

A Shiny app is a long-running R process. The browser loads the page over HTTP, then keeps a WebSocket open for the whole session, and every reactive value for that session lives in the R process that served it. That rules out static hosting (GitHub Pages can't run R) and serverless functions. It also means package versions matter: an app that works locally breaks in production when the server has a different `dplyr`. `renv.lock` pins those versions, and restoring it inside the image makes every deploy identical.

So when choosing where to host a Shiny app, you need something that keeps an R process running, passes WebSockets through and lets you control the R library. shinyapps.io and Posit Connect are the managed Shiny options. Shiny Server on your own Linux VM works, but you install R, system libraries, a reverse proxy and TLS yourself. A container platform runs your own image with HTTPS and WebSockets working by default. On Northflank, one **combined service** builds the image from Git on every push and serves it on an HTTPS URL.

## Prerequisites

- **A Shiny app that runs locally.** This starts it; open `http://127.0.0.1:3838` and stop it with Ctrl+C:
  ```bash
  R -s -e "shiny::runApp('.', port = 3838)"
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
ls app.R ui.R server.R renv.lock .Rprofile renv/activate.R renv/settings.json Dockerfile 2>/dev/null
grep -c '"Package": "shiny"' renv.lock
grep -A2 '"R"' renv.lock | grep Version

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId shiny-app -o json >/dev/null 2>&1 \
  && echo "project: shiny-app exists and will be reused" \
  || echo "project: shiny-app will be created"
```

- If there's no `app.R` (or `ui.R` and `server.R`) at the root, this isn't an R Shiny app in the layout this guide expects. Stop and use a guide for your framework. For Shiny for Python, see the Shiny for Python note in step 5.
- `renv` is the package manager. If `renv.lock` is missing or the `grep` prints `0`, step 5 shows how to create the lockfile.
- The R version tells you the image tag (`4.6` below).
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "shiny-app", "region": "europe-west" }'
export PROJECT_ID=shiny-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Write the `Dockerfile`

The project has to use renv with the lockfile committed. If it doesn't, run this in the project locally and commit `renv.lock`, `.Rprofile` and `renv/activate.R` (not `renv/library/`):

```r
renv::init()      # first time only
renv::snapshot()  # after installing or updating packages
```

Make sure `shiny` is in the lockfile: `grep -c '"Package": "shiny"' renv.lock` prints `1`.

`rocker/r-ver` is a minimal, versioned R image. Its `CRAN` environment variable points at Posit Package Manager, which serves prebuilt Linux binaries, so packages install much faster than from source. The renv layers follow the renv Docker guide: copy only the lockfile and activation files, restore, then copy the code, so a code change doesn't reinstall packages.

```dockerfile
FROM rocker/r-ver:4.6

# System libraries some packages need at runtime; add more if renv::restore() fails (see Troubleshooting)
RUN apt-get update && apt-get install -y --no-install-recommends \
      libcurl4-openssl-dev libssl-dev libxml2-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
ENV RENV_PATHS_LIBRARY=/renv/library \
    RENV_CONFIG_CACHE_SYMLINKS=FALSE \
    RENV_CONFIG_REPOS_OVERRIDE=${CRAN}
RUN mkdir -p renv
COPY renv.lock renv.lock
COPY .Rprofile .Rprofile
COPY renv/activate.R renv/activate.R
COPY renv/settings.json renv/settings.json
RUN R -s -e "renv::restore()"

COPY . .
RUN useradd --create-home shiny && chown -R shiny /app /renv
USER shiny

EXPOSE 3838
CMD ["R", "-s", "-e", "shiny::runApp('/app', host = '0.0.0.0', port = 3838, launch.browser = FALSE)"]
```

What matters here:

- **`host = '0.0.0.0'`**: `runApp` listens on `127.0.0.1` by default, which isn't reachable from outside the container.
- **`port = 3838`** is the conventional Shiny port, and the same one `rocker/shiny` uses. It must match the service port in step 7.
- **`RENV_CONFIG_REPOS_OVERRIDE`** makes `renv::restore()` install from the image's Package Manager URL instead of the repository recorded in the lockfile. The package versions still come from `renv.lock`. Drop the line if your lockfile points at a private repository.
- **`RENV_PATHS_LIBRARY`** puts the library outside `/app`, so `COPY . .` can't overwrite it with a local `renv/library`.
- **`RENV_CONFIG_CACHE_SYMLINKS=FALSE`** copies packages into the library instead of symlinking them from root's cache, which the non-root `shiny` user couldn't read.
- If `renv/settings.json` doesn't exist in your repo, delete that `COPY` line.

Add a `.dockerignore` with `renv/library`, `renv/staging`, `.Rproj.user`, `.Renviron` and `.git`.

**Alternative: Shiny Server.** `rocker/shiny:4.6` ships Shiny Server, which listens on 3838 and serves every app under `/srv/shiny-server`. Copy the app to `/srv/shiny-server/` and keep the image's default command. Rocker's docs say the server logs to stdout unless `APPLICATION_LOGS_TO_STDOUT=false`. Shiny Server starts the R processes itself; the image's start-up script copies container environment variables into `Renviron.site` so they reach `Sys.getenv()`, but it skips a few names (`HOME`, `PASSWORD`, `RSTUDIO_VERSION`, `BATCH_USER_CREATION`), so don't name a secret `PASSWORD`. One app per container with `runApp`, as above, avoids that indirection.

**Shiny for Python.** The same deployment works with a Python base image. The Shiny for Python docs run `uvicorn app:app --host 0.0.0.0 --port 80`; `shiny run --host 0.0.0.0 --port 3838 app.py` does the same. Don't run it under gunicorn with more than one worker: the docs say Shiny needs sticky sessions, which gunicorn doesn't provide.

Northflank builds what's in your repository, not the files on your machine, so commit and push them:

```bash
git add Dockerfile .dockerignore renv.lock .Rprofile renv/activate.R   # plus renv/settings.json if it exists, and app files you changed
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Put secrets in a secret group

Don't commit `.Renviron`. Read keys from the environment in R:

```r
api_key <- Sys.getenv("API_KEY")
if (!nzchar(api_key)) stop("API_KEY is not set")
```

Create the group without echoing the value. `secretType: "environment"` injects variables at runtime only, never into the build:

```bash
jq -n --arg k "$API_KEY" '{
  name: "shiny-secrets", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "shiny", type: "service" } ] },
  secrets: { variables: { API_KEY: $k } }
}' > shiny-secrets.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file shiny-secrets.json >/dev/null && rm shiny-secrets.json
```

Restricting the group to service `shiny` before it exists hasn't been confirmed to work. If the call is rejected, use `"restricted": false`, or run it after step 7 and restart the service. Skip this step if the app has no secrets.

## Step 7: Create the service

R and loaded data sit in memory, so start at `nf-compute-50` (1 GB); `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[].id'` lists the plans.

Write `shiny.json`:

```json
{
  "name": "shiny",
  "billing": { "deploymentPlan": "nf-compute-50" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 3838, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "TZ": "UTC" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/", "port": 3838,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "TCP", "type": "livenessProbe", "port": 3838,
      "initialDelaySeconds": 30, "periodSeconds": 20, "timeoutSeconds": 3, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file shiny.json
```

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined`. The first build starts automatically; if none appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId shiny --noDefaults -o json --input '{}'`.

Shiny has no dedicated health endpoint. The readiness probe loads the UI page, which returns 200 once the app has started. The liveness probe only checks the port, so a slow page doesn't get the container restarted. If the app loads large data at start-up, raise `initialDelaySeconds`.

**Instances and WebSockets.** Keep `instances: 1` until you need more. With several, each reconnect must reach the instance that holds the session. Add sticky routing to `shiny.json` with `"loadBalancing": { "mode": "consistentHash", "consistentHash": { "mode": "ip" } }`. Whether this keeps sessions intact across reconnects hasn't been confirmed, so test it before relying on it. A bigger plan (more memory for one R process) is usually the simpler first step.

## Step 8: Wait for the rollout and get the URL

The first build downloads every package in the lockfile, so allow several minutes.

```bash
OK=0
for i in $(seq 1 120); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId shiny --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId shiny --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId shiny --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. If the build fails, read `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId shiny`.

## How to know it worked

You're done when these hold. Skip check 5 if the app has no secrets.

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId shiny --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The app page loads and is Shiny's:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"          # 200
   curl -s "https://$HOST/" | grep -c 'shiny'                          # > 0
   ```
3. R logged that it's listening on all interfaces:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId shiny --lineLimit 200 --textIncludes "Listening on"
   ```
   This prints `Listening on http://0.0.0.0:3838`.
4. The WebSocket works: open the URL in a browser. Inputs update outputs and the page doesn't grey out with "Disconnected from the server". An HTTP check can't confirm this, so ask the user or use a browser tool.
5. Secrets are injected (names only), if you did step 6. `exec` drops output without a TTY, so wrap it in `script` (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId shiny \
     --shell-cmd 'sh -c' --cmd '"env | cut -d= -f1 | grep -x API_KEY"'
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| 502/503 while `COMPLETED`, logs show `Listening on http://127.0.0.1:3838` | `host` not set | Pass `host = '0.0.0.0'` to `runApp` |
| `renv::restore()` fails with `configuration failed for package` | Missing system library for a source package | Add the `-dev` package named in the error to the `apt-get` line; rebuild |
| `there is no package called 'x'` at runtime | Package not in `renv.lock` | Run `renv::snapshot()` locally, commit the lockfile |
| Page greys out, "Disconnected from the server" | WebSocket dropped, app crashed, or request reached another instance | Check `northflank get service logs`; use one instance or sticky routing |
| Container restarts while loading data | Liveness probe fires too early, or out of memory | Raise `initialDelaySeconds`; move to a plan with more memory |
| Build is slow on every push | `COPY . .` before the restore | Keep the lockfile layers above `COPY . .` |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to |
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
- Never commit `.Renviron` or print secret values. Print variable names only.
- Don't edit `renv.lock` by hand. Change packages locally with `renv::install()` and `renv::snapshot()`, and tell the user what changed.
- The URL is public and Shiny has no built-in login. If the app shows private data, ask the user how it should be protected before sharing the link (see [port security policies](https://northflank.com/docs/v1/application/network/add-security-policies-for-ports)).
- Don't delete the service or project without explicit confirmation.

## FAQ

### Where should I deploy my Shiny app?
On a host that keeps an R process running and supports WebSockets. shinyapps.io and Posit Connect are the managed Shiny products. If you want your own image, private packages, secrets or a database next to the app, a container platform runs the `Dockerfile` above with HTTPS. A VM with Shiny Server works if you want to manage R, the proxy and TLS yourself.

### Can I host a Shiny app on GitHub Pages?
Not a standard Shiny app, because GitHub Pages serves static files and can't run R. Shinylive compiles some apps to WebAssembly so they run in the browser, and those can be served statically. Apps that need server-side packages, secrets or a database still need a server.

### Should I use `rocker/shiny` or `rocker/r-ver`?
Use `rocker/r-ver` with `runApp` for one app per container: it's smaller and environment variables reach the app directly. Use `rocker/shiny` if you want Shiny Server's multi-app directory and its configuration file.

### How do I connect a database?
Create a managed database in the same project, link its connection details into the secret group, and read them with `Sys.getenv()` in `DBI::dbConnect()`. See [how to deploy a PostgreSQL database](deploy-postgresql-database.md).

### Does the app need persistent storage?
Usually not. Session state is in memory and resets on restart. Write anything durable to a database or object storage.

## Conclusion

You now have a Shiny app built from Git with pinned packages from `renv.lock`, running on `0.0.0.0:3838` behind HTTPS with WebSockets passed through, secrets injected from a secret group and health checks on the app port.

**Next:** add your own domain with [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank), or add a database with [how to deploy a PostgreSQL database](deploy-postgresql-database.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Streamlit app](deploy-streamlit-app.md)
- [How to deploy a Dash app](deploy-dash-app.md)
- [How to deploy a Gradio app](deploy-gradio-app.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to host a Jupyter notebook](host-jupyter-notebook.md)
