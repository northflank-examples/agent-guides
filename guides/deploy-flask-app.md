<!--
agent-guide-meta:
  id: NF-STK-FLASK
  description: "How to deploy a Flask app to production with gunicorn in Docker and managed PostgreSQL, running Flask-Migrate as a one-off job and serving a public HTTPS URL."
  sources:
    - https://northflank.com/stacks/deploy-flask
    - https://flask.palletsprojects.com/en/stable/deploying/gunicorn/
    - https://flask-migrate.readthedocs.io/en/latest/
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/project/jobs/start-job-build
    - https://northflank.com/docs/v1/api/project/jobs/run-job
    - https://northflank.com/docs/v1/api/project/jobs/get-run-details
    - https://northflank.com/docs/v1/application/release/run-migrations
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
  related: [NF-DEP-006, NF-DB-001, NF-DB-003, NF-JOB-002, NF-DEP-007]
-->

# How to deploy a Flask app

To deploy a Flask app, package it in a Docker image that serves it with gunicorn on `0.0.0.0:$PORT` instead of `flask run`, connect a managed PostgreSQL database through `DATABASE_URL`, and run `flask db upgrade` once per release as a separate step. The development server isn't built for production, and gunicorn binds to `127.0.0.1` unless told otherwise, which leaves the app unreachable.

This guide does it on Northflank with the CLI: the app built from its `Dockerfile`, Postgres linked as `DATABASE_URL` through a secret group, and Flask-Migrate as a one-off job that reports `SUCCESS` or `FAILED`. You end with a public HTTPS URL where `/health` returns 200 and the schema is at head. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a Flask app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a Flask app, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Add a `Dockerfile` that runs `gunicorn 'app:create_app()'` on `0.0.0.0:$PORT`, and push it.
4. Create a project, Postgres (`northflank put addon`), a secret group that links `POSTGRES_URI` as `DATABASE_URL`, the service (`northflank put service combined`) and a manual job that runs `flask db upgrade`.
5. Wait for the deploy, run the job with `northflank start job run` until it reports `SUCCESS`.
6. Check that the live build is your pushed commit and that `https://$HOST/health` returns 200.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your Flask app on Northflank

A Flask app in production needs a WSGI server process that stays up (gunicorn), a database it can reach privately, secrets injected at runtime, and a migration step that runs once per release. Static and serverless hosts are built for files or short-lived functions, not a persistent WSGI server with a database beside it. A raw VM or Kubernetes cluster can run anything, but you set up TLS, builds, restarts and secrets yourself. A container platform sits between the two: you bring a `Dockerfile`, it builds it from Git, runs it and serves it over HTTPS.

On Northflank the app is a combined service (build and run from Git), Postgres is an addon, and migrations are a job with a run ID, a terminal status and its own logs. Chaining `flask db upgrade && gunicorn …` in the web container would run the migration on every instance at every restart, and a failure would show up as a crash loop.

Prefer the UI? Northflank has a one-click [Flask stack template](https://northflank.com/stacks/deploy-flask) that clones a starter repo and deploys it as a combined service (no database).

Building an async API instead? See [how to deploy a FastAPI app](deploy-fastapi-app.md).

## Prerequisites

- **A Flask app that imports locally:**
  ```bash
  pip install -r requirements.txt && python -c "import flask, app" && echo IMPORT_OK
  ```
  Replace `app` with your package or module name.
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

Names used below: project `flask-app`, addon `web-db`, secret group `web-env`, service `web`, job `web-migrate`.

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
grep -ioE '\b(flask|flask-sqlalchemy|flask-migrate|gunicorn|django|fastapi)\b' requirements.txt pyproject.toml 2>/dev/null | sort -u
ls requirements.txt pyproject.toml poetry.lock uv.lock Pipfile.lock Dockerfile migrations 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId flask-app -o json >/dev/null 2>&1 \
  && echo "project: flask-app exists and will be reused" \
  || echo "project: flask-app will be created"
```

- If `flask` doesn't appear (or `django` or `fastapi` does instead), this isn't a Flask app. Stop and use a guide for your framework.
- The files tell you the package manager. Step 5 installs from `requirements.txt`; with Poetry or uv, export one (`poetry export -o requirements.txt` or `uv export --format requirements-txt -o requirements.txt`) or swap the install command.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "flask-app", "region": "europe-west" }'
export PROJECT_ID=flask-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3. `northflank get job repository-access` takes the same body if you want to check the job separately.

## Step 5: Prepare the app and add the `Dockerfile`

The app uses Flask-SQLAlchemy and Flask-Migrate and reads `DATABASE_URL`. With an app factory in `app/__init__.py`:

```python
import os
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from sqlalchemy import text

db = SQLAlchemy()
migrate = Migrate()

def create_app():
    app = Flask(__name__)
    app.config["SQLALCHEMY_DATABASE_URI"] = os.environ["DATABASE_URL"]
    db.init_app(app)
    migrate.init_app(app, db)

    @app.get("/health")
    def health():
        db.session.execute(text("SELECT 1"))
        return {"status": "ok"}

    return app
```

`POSTGRES_URI` starts with `postgresql://`, which SQLAlchemy maps to psycopg2. Commit the `migrations/` folder that `flask db init` and `flask db migrate` created locally; the job only applies it.

If there's no `Dockerfile` at the repo root, write one:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 FLASK_APP=app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt   # flask, flask-sqlalchemy, flask-migrate, gunicorn, psycopg2-binary
COPY . .
CMD ["sh", "-c", "exec gunicorn 'app:create_app()' --bind 0.0.0.0:${PORT:-8080} --workers 2 --access-logfile -"]
```

Use `app:app` if you don't use a factory. The Flask docs note gunicorn defaults to one worker and binds to `127.0.0.1` unless you pass `-b 0.0.0.0`. Flask-Migrate needs `FLASK_APP` for `flask db` commands; setting it in the image covers the job too. Flask serves `/static` itself, which is fine for small apps.

Northflank builds what's in your repository, not the files on your machine, so commit and push them:

```bash
git add Dockerfile app/__init__.py migrations/
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create the PostgreSQL addon and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "web-db",
  "type": "postgresql",
  "version": "16-latest",
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId web-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in
    running) OK=1; break ;;
    failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break ;;
  esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon in the UI"; false; }
```

Storage can only be increased later, never reduced. More options are in [how to deploy a PostgreSQL database](deploy-postgresql-database.md).

## Step 7: Link the connection string as `DATABASE_URL`

The service and the migration job both need the URL, so the group is unrestricted (every workload in the project can read it).

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "web-env",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": false },
  "addonDependencies": [
    { "addonId": "web-db", "keys": [ { "keyName": "POSTGRES_URI", "aliases": ["DATABASE_URL"] } ] }
  ]
}'
```

`secretType: "environment"` injects at runtime only, so `DATABASE_URL` isn't available during the image build. Don't touch the database in the `Dockerfile`. Put other secrets such as `SECRET_KEY` in the same group under `secrets.variables`, read from your shell rather than typed into chat.

## Step 8: Deploy the Flask service

Write `web.json`:

```json
{
  "name": "web",
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
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file web.json
```

Use the same repository URL, provider and branch as in step 4. The first build starts automatically.

## Step 9: Create the migration job

The job builds the same repo with the same `Dockerfile` and overrides the command. Write `migrate.json`:

```json
{
  "name": "web-migrate",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "settings": { "backoffLimit": 0, "activeDeadlineSeconds": 600, "runOnSourceChange": "never" },
  "deployment": {
    "vcs": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
    "docker": { "configType": "customCommand", "customCommand": "flask db upgrade" }
  },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "runtimeEnvironment": { "FLASK_APP": "app" }
}
```

```bash
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file migrate.json
```

- Use the same repository URL, provider and branch as in step 4.
- `put job manual` is deprecated in the API, so this uses `put job`, which makes a manual job when `settings.cron` is absent. Run settings go under `settings`.
- `backoffLimit: 0`: a failed migration isn't retried. `activeDeadlineSeconds` fails a run that takes longer.
- `runOnSourceChange` accepts `never`, `cd-promote` or `always`. Keep `never` so migrations only run when you start them.
- The command runs in exec form. If your `Dockerfile` has an `ENTRYPOINT`, use `"configType": "customEntrypointCustomCommand"` with `"customEntrypoint": "sh -c"` and `"customCommand": "\"flask db upgrade\""`.

## Step 10: Wait for the rollout and the job build

`COMPLETED` means rolled out and serving.

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

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web`. `/health` can pass before the schema exists; table-backed routes fail until step 11.

It hasn't been confirmed that creating a job starts a build, so if none appears within a minute, start one:

```bash
OK=0
for i in $(seq 1 90); do
  JB=$(northflank get job builds --projectId "$PROJECT_ID" --jobId web-migrate --noDefaults -o json \
       | jq -r '(.data // .).builds | sort_by(.createdAt) | last | .status // "NONE"')
  echo "job build=$JB"
  case "$JB" in
    SUCCESS) OK=1; break ;;
    FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo JOB_BUILD_FAILED; break ;;
    NONE) [ "$i" = 6 ] && northflank start job build --projectId "$PROJECT_ID" --jobId web-migrate --noDefaults -o json --input '{}' ;;
  esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "JOB_BUILD_NOT_COMPLETE: stop here and read the job build logs"; false; }
```

## Step 11: Run the migration and wait for it to finish

```bash
RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId web-migrate --noDefaults -o json --input '{}' \
         | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 60); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId web-migrate --runId "$RUN_ID" --noDefaults -o json)
  ST=$(echo "$R" | jq -r '(.data // .).status'); C=$(echo "$R" | jq -r '(.data // .).concluded')
  echo "run=$ST concluded=$C"
  [ "$C" = "true" ] && { [ "$ST" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId web-migrate --runId "$RUN_ID" --lineLimit 200
[ "$OK" = 1 ] || { echo "MIGRATION_NOT_SUCCEEDED: stop here and read the job logs above"; false; }
```

A run's `status` is `RUNNING`, `SUCCESS` or `FAILED`. Repeat this step after each merge that adds a migration, or automate it with [run migrations](https://northflank.com/docs/v1/application/release/run-migrations).

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The migration run concluded with `status == "SUCCESS"`:
   ```bash
   northflank get job run --projectId "$PROJECT_ID" --jobId web-migrate --runId "$RUN_ID" --noDefaults -o json | jq -r '(.data // .).status'
   ```
3. The health endpoint returns 200:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/health"
   ```
4. Flask-Migrate reports the head revision from inside the running container. `exec` drops output without a TTY, so wrap it with `script` (macOS shown; Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId web \
     --shell-cmd 'sh -c' --cmd '"flask db current"'
   ```
   The output should end with `(head)`.
5. A route that reads a migrated table returns 200.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Job fails with `Could not locate a Flask application` | `FLASK_APP` not set | Set it in the image or the job's `runtimeEnvironment` |
| Job or app fails with `KeyError: 'DATABASE_URL'` | Secret group restricted away, or created after the run | Check `restrictions`; restart or re-run step 11 |
| `ModuleNotFoundError: psycopg2` | Driver missing | Add `psycopg2-binary` to `requirements.txt` |
| Crash loop, logs show `Listening at: http://127.0.0.1:8000` | gunicorn bound to localhost | Bind `0.0.0.0:${PORT}` |
| Health check fails right after deploy | Workers slow to boot | Raise `initialDelaySeconds`; see [configure health checks](https://northflank.com/docs/v1/application/observe/configure-health-checks) |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` (and the job's `deployment.vcs`) as `"vcsLinkId": "<ID>"` |
| Job build `FAILURE` | Wrong Dockerfile path or failing `pip install` | `northflank get job build-logs --projectId "$PROJECT_ID" --jobId web-migrate`; see [build with a Dockerfile](https://northflank.com/docs/v1/application/build/build-with-a-dockerfile) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service or job until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 10 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print `DATABASE_URL`, `SECRET_KEY` or addon passwords. Print key names only.
- Don't deploy with `flask run` or `debug=True`. The debugger allows code execution from the browser.
- Before migrating a database that already holds data, take a backup (see [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data)) and tell the user which revision you'll apply.
- Don't delete the addon, service or job without explicit, per-resource confirmation. Prefer pausing.

## FAQ

### Where should I deploy my Flask app?
On a host that runs a long-lived container with gunicorn and gives it a private Postgres database, runtime secrets and a migration step. A container platform does that from your `Dockerfile` without managing servers. Serverless hosts need an adapter and don't keep a process running, and a raw VM leaves TLS, restarts and builds to you.

### Do I need nginx in front of gunicorn?
Not here. The platform terminates HTTPS and routes traffic to your port, so gunicorn binds `0.0.0.0` directly.

### Can I use SQLite instead of Postgres?
A SQLite file lives in the container's filesystem, which doesn't survive redeploys unless you attach a volume, and it can't be shared between instances. Use the Postgres addon.

### How do I run Celery tasks for my Flask app?
Deploy a worker service from the same repo with no ports and `celery -A make_celery worker`. See [how to deploy Celery in production](deploy-celery-in-production.md).

## Conclusion

You now have a Flask app served by gunicorn on Northflank, with a managed Postgres database, `DATABASE_URL` injected from a secret group, and Flask-Migrate migrations that run as a job with a clear success or failure status.

**Next:** for logs, see [view logs](https://northflank.com/docs/v1/application/observe/view-logs). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a FastAPI app](deploy-fastapi-app.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a Django project](deploy-django-project.md)
