<!--
agent-guide-meta:
  id: NF-STK-DASH
  description: "How to deploy a Plotly Dash app to production: expose app.server, run gunicorn app:server on 0.0.0.0:$PORT in Docker, and verify the layout endpoint."
  sources:
    - https://dash.plotly.com/reference
    - https://dash.plotly.com/sharing-data-between-callbacks
    - https://dash.plotly.com/deployment
    - https://dash.plotly.com/background-callbacks
    - https://docs.gunicorn.org/en/stable/settings.html#bind
    - https://pypi.org/project/dash/
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/scale/scale-instances
  related: [NF-STK-STREAMLIT, NF-STK-FLASK, NF-STK-SHINY, NF-DB-001, NF-DB-003]
-->

# How to deploy a Dash app

To deploy a Dash app, expose the underlying Flask server with `server = app.server`, run it with gunicorn (`gunicorn app:server --bind 0.0.0.0:$PORT`) in a Docker container instead of `app.run()`, keep callback state out of global variables, and read secrets from environment variables. `app.run()` is Dash's development server and listens on `127.0.0.1:8050` by default, which is the usual reason a deployed Dash app is unreachable.

This guide does it on Northflank with the CLI: a small `Dockerfile` built from Git, gunicorn workers on the port in `PORT`, secrets in a secret group, and health checks on the app. You end with a public HTTPS URL where the page and Dash's `/_dash-layout` endpoint both return 200. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a Dash app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Dash app and confirm the team, repository, branch and project before changing anything, then give Northflank access to the repository (once).
3. Add `server = app.server` to `app.py` and `gunicorn` to `requirements.txt`, start with `gunicorn app:server --bind 0.0.0.0:${PORT} --workers 2` in the `Dockerfile`, and push.
4. Create a project, a secret group for API keys, and a service with `northflank put service combined --file dash.json`: HTTP port 8050, `PORT=8050`, health checks on `/`.
5. Wait for the deploy, then check that the live build is your pushed commit and that `/_dash-layout` returns JSON.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why gunicorn, and where to run it

A Dash app is a Flask app underneath. The Dash reference calls `app.server` "the Flask server associated with this app" and points to `gunicorn app:server` for production, and it warns that `app.run` shouldn't be used on a production server. Unlike Streamlit or Shiny, Dash callbacks are ordinary HTTP POST requests to `/_dash-update-component`, not a WebSocket session. That makes Dash easy to scale: any worker on any instance can answer any callback, as long as the app follows one rule from the Dash docs: callbacks "must never modify variables outside of their scope", because workers don't share memory. Put shared state in `dcc.Store`, a database or Redis.

So when choosing where to deploy a Dash app, you need a host that keeps a Python web process running. Plotly's own cloud offering is the managed option. A Linux VM works with gunicorn, a reverse proxy and TLS you set up yourself. Serverless function hosts can run Flask with an adapter, but large Pandas data and cold starts make that awkward. A container platform runs your `Dockerfile` with HTTPS and lets you add instances later. On Northflank, one **combined service** builds from Git on every push and serves the app on an HTTPS URL.

## Prerequisites

- **A Dash app that runs locally.** This guide assumes the Dash object is `app` in `app.py` at the repo root, and that `requirements.txt` pins `dash` plus anything the app imports (`pandas`, `plotly`…). Run it, then open `http://localhost:8050`:
  ```bash
  pip install -r requirements.txt && python app.py
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

Names used below: project `dash-app`, secret group `dash-secrets`, service `dash`.

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
grep -hiE '^[[:space:]"]*dash\b' $(ls requirements.txt pyproject.toml 2>/dev/null) /dev/null || echo "dash: missing"
ls app.py requirements.txt pyproject.toml poetry.lock uv.lock Pipfile.lock Dockerfile 2>/dev/null
grep -n 'global ' app.py

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId dash-app -o json >/dev/null 2>&1 \
  && echo "project: dash-app exists and will be reused" \
  || echo "project: dash-app will be created"
```

- If it prints `dash: missing`, this isn't a Dash app. Stop and use a guide for your framework.
- The lockfile tells you the package manager. Step 5 assumes `requirements.txt`; for Poetry, uv or Pipenv, export a `requirements.txt` or swap the install command.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If `grep 'global '` finds a callback that writes to a global variable, move that state into `dcc.Store` or a database first: gunicorn workers don't share memory.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "dash-app", "region": "europe-west" }'
export PROJECT_ID=dash-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Expose the server and write the `Dockerfile`

Add `gunicorn` to `requirements.txt`. In `app.py`, keep the development entry point for local use and expose the Flask server for gunicorn:

```python
import os
from dash import Dash, html, dcc

app = Dash(__name__)
server = app.server          # gunicorn imports this: app:server

app.layout = html.Div([html.H1("Dashboard"), dcc.Graph(id="chart")])

if __name__ == "__main__":
    app.run(debug=True)      # local development only
```

Gunicorn imports the module, so the `if __name__ == "__main__"` block doesn't run in production and debug mode stays off. Read secrets with `os.environ["API_KEY"]`; step 6 injects them. Then write the `Dockerfile`:

```dockerfile
FROM python:3.13-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 PORT=8050
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN useradd --create-home appuser && chown -R appuser /app
USER appuser
EXPOSE 8050
CMD ["sh", "-c", "exec gunicorn app:server --bind 0.0.0.0:${PORT} --workers ${WEB_CONCURRENCY:-2} --timeout 120 --access-logfile -"]
```

- **`app:server`** means module `app`, attribute `server`. If your file is `src/dashboard.py`, it becomes `src.dashboard:server` (with an `__init__.py`), or set `--chdir src` and use `dashboard:server`.
- **`--bind 0.0.0.0:${PORT}`** listens on every interface on the port the platform sends. The shell form expands `$PORT`; `exec` makes gunicorn PID 1 so it receives stop signals.
- **`--workers`** runs callbacks in parallel, as the Dash docs show with `gunicorn --workers 4 app:server`. `WEB_CONCURRENCY` lets you change it without rebuilding. Start with 2 per vCPU and watch memory.
- **`--timeout 120`** gives slow callbacks (big queries, model calls) more than gunicorn's 30-second default before the worker is killed. Long jobs belong in Dash background callbacks with a Celery manager (the Dash docs don't recommend the diskcache manager for production).

Add a `.dockerignore` with `.venv`, `__pycache__`, `.env` and `.git`. Northflank builds what's in your repository, not the files on your machine, so commit and push:

```bash
git add app.py Dockerfile .dockerignore requirements.txt
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Put secrets in a secret group

Skip this step if the app reads no secrets. Create the group without echoing values. `secretType: "environment"` injects them at runtime only:

```bash
jq -n --arg k "$API_KEY" '{
  name: "dash-secrets", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "dash", type: "service" } ] },
  secrets: { variables: { API_KEY: $k } }
}' > dash-secrets.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file dash-secrets.json >/dev/null && rm dash-secrets.json
```

Restricting the group to service `dash` before it exists hasn't been confirmed to work. If the call is rejected, use `"restricted": false`, or run it after step 7 and restart the service. If the app reads a database, link the database into this group instead of pasting a URL (see [how to deploy a PostgreSQL database](deploy-postgresql-database.md)).

## Step 7: Create the service

Each gunicorn worker holds its own copy of the app's data, so size memory for `workers × dataset`. `nf-compute-50` (1 GB) is a reasonable start (`curl -s https://api.northflank.com/v1/plans` lists the others). Write `dash.json`:

```json
{
  "name": "dash",
  "billing": { "deploymentPlan": "nf-compute-50" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8050, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "8050", "WEB_CONCURRENCY": "2" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/", "port": 8050,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/", "port": 8050,
      "initialDelaySeconds": 20, "periodSeconds": 20, "timeoutSeconds": 5, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file dash.json
```

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined` with the same body. The first build starts automatically; if none appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId dash --noDefaults -o json --input '{}'`.

`PORT`, `internalPort` and the probe ports must all be the same number. The index page is a cheap static HTML shell, so it's a reasonable probe; if the app loads a large dataset at import time, raise `initialDelaySeconds`. Because callbacks are stateless HTTP requests, you can raise `instances` later without sticky sessions, as long as no state lives in globals.

## Step 8: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds and prints your URL when the app is live. You can also watch the build in the Northflank dashboard.

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId dash --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId dash --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId dash --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId dash`. The troubleshooting table covers the common causes.

`COMPLETED` means rolled out and serving. To change workers later, patch the **complete** `runtimeEnvironment` map and restart:

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId dash --noDefaults -o json \
  --input '{ "runtimeEnvironment": { "PORT": "8050", "WEB_CONCURRENCY": "4" } }'
northflank restart service --projectId "$PROJECT_ID" --serviceId dash --noDefaults -o json
```

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId dash --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The page shell loads:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"     # 200
   ```
3. Dash's layout endpoint answers with JSON, which proves the Dash app (not just a web server) is running:
   ```bash
   curl -s -o /dev/null -w '%{http_code} %{content_type}\n' "https://$HOST/_dash-layout"   # 200 application/json
   ```
4. Gunicorn started its workers on the right address:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId dash --lineLimit 200 --textIncludes "Listening at"
   ```
   This prints `Listening at: http://0.0.0.0:8050`.
5. Charts render and react to input in a browser. Ask the user or use a browser tool; failed callbacks show as red error bubbles only in debug mode, so also check the logs for tracebacks.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |
| `Failed to find attribute 'server' in 'app'` | `server = app.server` missing | Add it below `app = Dash(__name__)` |
| `ModuleNotFoundError: No module named 'app'` | Wrong module path in `app:server` | Use the file's module path or `--chdir` |
| 502/503 while `COMPLETED` | Port mismatch, or `app.run()` used as the start command | Use gunicorn with `--bind 0.0.0.0:${PORT}`; match `internalPort` |
| `WORKER TIMEOUT` then 502 on some callbacks | Callback takes longer than `--timeout` | Raise `--timeout`, or move it to a background callback |
| Values differ between refreshes | State kept in globals across workers | Move it to `dcc.Store`, Redis or a database |
| `OOMKilled` / restarts | Each worker loads the full dataset | Fewer workers, `--preload`, or a plan with more memory |

Logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId dash` and `northflank get service logs --projectId "$PROJECT_ID" --serviceId dash --lineLimit 200`.

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never set `debug=True` in production to see an error. It enables the dev tools for every visitor. Read the logs instead.
- Never print secret values; print variable names only.
- Dash has no built-in login. If the dashboard shows private data, ask the user how to protect it before sharing the URL (for example `dash-auth`, or [port security policies](https://northflank.com/docs/v1/application/network/add-security-policies-for-ports)).
- Don't delete the service or project without explicit confirmation.

## FAQ

### Where should I deploy my Dash app?
On a host that runs a long-lived Python web process with gunicorn. A container platform fits when you want your own `Dockerfile`, secrets and a database next to the app. A Linux VM works if you want to manage nginx, TLS and process supervision yourself.

### Do I still need `app.run()`?
Only for local development. In production gunicorn imports `server` from your module and `app.run()` never executes.

### Can I serve the app under a path such as `/dash/`?
Yes. Set `url_base_pathname="/dash/"` in `Dash(...)`, or the `DASH_URL_BASE_PATHNAME` environment variable, and update the probe paths to match.

### Does Dash need sticky sessions or WebSockets?
No. Callbacks are plain HTTP requests, so any worker or instance can answer them. That only holds if you keep shared state out of global variables.

### How do I run long callbacks?
Use Dash background callbacks with a Celery manager backed by Redis, and run the Celery worker as a second service. See [how to deploy Redis](deploy-redis.md) and [how to deploy Celery in production](deploy-celery-in-production.md).

## Conclusion

You now have a Dash app built from Git, served by gunicorn on `0.0.0.0:$PORT` behind HTTPS, with secrets from a secret group and health checks on the app, ready to add workers or instances because no state lives in globals.

**Next:** add a database with [how to deploy a PostgreSQL database](deploy-postgresql-database.md), or your own domain with [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Streamlit app](deploy-streamlit-app.md)
- [How to deploy a Flask app](deploy-flask-app.md)
- [How to deploy a Shiny app](deploy-shiny-app.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
