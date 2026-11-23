<!--
agent-guide-meta:
  id: NF-DEP-007
  description: "How to deploy a Django project to production with Docker, gunicorn, WhiteNoise and managed PostgreSQL, with migrations as a job and every step scripted."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/project/jobs/run-job
    - https://northflank.com/docs/v1/api/project/jobs/get-run-details
    - https://northflank.com/docs/v1/application/release/run-migrations
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
  related: [NF-DB-001, NF-DB-003, NF-JOB-002, NF-DEP-006]
-->

# How to deploy a Django project

To deploy a Django project to production, package it in a Docker image that runs `collectstatic` at build time and serves with gunicorn on `0.0.0.0`, connect a managed PostgreSQL database through `DATABASE_URL`, run `migrate` as a separate step, and set `ALLOWED_HOSTS` and `CSRF_TRUSTED_ORIGINS` to your public hostname. Skipping any of these is the usual reason a Django deploy returns 400s or unstyled pages.

This guide does each step on Northflank with the CLI: WhiteNoise for static files, Postgres linked as `DATABASE_URL`, migrations as a one-off job with a clear pass or fail, and host settings read from the URL Northflank assigns. You end with a public HTTPS URL serving your admin login and static files, verified with `curl`. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a Django project

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a Django project, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Read settings from the environment, add a `Dockerfile` that runs `collectstatic` and starts gunicorn on `0.0.0.0:$PORT`, and push.
4. Create a project, Postgres (`northflank put addon`), a secret group with `DATABASE_URL` and `DJANGO_SECRET_KEY`, and the service (`northflank put service combined`), then set `ALLOWED_HOSTS` and `CSRF_TRUSTED_ORIGINS` from its public host.
5. Run `python manage.py migrate --noinput` as a manual job with `northflank put job` and `northflank start job run`.
6. Check that the live build is your pushed commit, and that `/admin/login/` and a static file both return 200.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why separate the Django steps?

Each Django production step has a different lifetime:

- **`collectstatic`** produces files, so it belongs in the image build.
- **`migrate`** changes the database once per release, so it belongs in a job.
- **gunicorn** runs forever, so it belongs in the service.

Mixing them up causes most failed Django deploys. On Northflank, those three map to a `Dockerfile` build, a manual job and a combined service.

That's also the answer to where to deploy a Django project: you need a host that keeps a Python process running, gives you a Postgres database, and can run a one-off command per release. Static and serverless hosts are built for short request-scoped functions, so Django there usually needs an adapter and a database from somewhere else. A raw VM or your own Kubernetes cluster runs anything, but you set up the process manager, reverse proxy, TLS and database yourself. A container platform with managed databases and jobs covers all three pieces directly.

## Prerequisites

- **A Django project that passes its checks locally:**
  ```bash
  pip install -r requirements.txt && python manage.py check && echo CHECK_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

Names used below: project `django-app`, addon `django-db`, secret group `django-env`, service `web`, job `web-migrate`, and `myproject` as the Django project package.

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
grep -ioE '\b(django|gunicorn|whitenoise|dj-database-url|psycopg2?(-binary)?|flask|fastapi)\b' requirements.txt pyproject.toml 2>/dev/null | sort -u
ls manage.py requirements.txt pyproject.toml poetry.lock uv.lock Pipfile.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId django-app -o json >/dev/null 2>&1 \
  && echo "project: django-app exists and will be reused" \
  || echo "project: django-app will be created"
```

- If `django` or `manage.py` is missing, this isn't a Django project. Stop and use a guide for your framework.
- The files tell you the package manager. Step 5 installs from `requirements.txt`; with Poetry or uv, export one (`poetry export -o requirements.txt` or `uv export --format requirements-txt -o requirements.txt`) or swap the install command.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "django-app", "region": "europe-west" }'
export PROJECT_ID=django-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3. `northflank get job repository-access` takes the same body if you want to check the job separately.

## Step 5: Read settings from the environment and add the `Dockerfile`

Make sure `requirements.txt` includes `gunicorn`, `whitenoise`, `dj-database-url` and `psycopg[binary]` (or `psycopg2-binary`).

`settings.py` reads production values from the environment. This is standard Django configuration:

```python
import os
import dj_database_url

SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
DEBUG = os.environ.get("DJANGO_DEBUG", "0") == "1"
ALLOWED_HOSTS = [h for h in os.environ.get("ALLOWED_HOSTS", "").split(",") if h]
CSRF_TRUSTED_ORIGINS = [o for o in os.environ.get("CSRF_TRUSTED_ORIGINS", "").split(",") if o]
DATABASES = {"default": dj_database_url.config(conn_max_age=600)}  # reads DATABASE_URL

STATIC_URL = "static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
STORAGES = {
    "default": {"BACKEND": "django.core.files.storage.FileSystemStorage"},
    "staticfiles": {"BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage"},
}
# In MIDDLEWARE, add "whitenoise.middleware.WhiteNoiseMiddleware" right after SecurityMiddleware.
```

If there's no `Dockerfile`, write one:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
# collectstatic needs settings to import; give it throwaway values that never reach runtime
RUN DJANGO_SECRET_KEY=build-only DATABASE_URL=sqlite:////tmp/build.db python manage.py collectstatic --noinput
CMD ["sh", "-c", "exec gunicorn myproject.wsgi:application --bind 0.0.0.0:${PORT:-8000} --workers 3 --access-logfile -"]
```

Northflank builds what's in your repository, not the files on your machine, so commit and push them:

```bash
git add Dockerfile requirements.txt myproject/settings.py
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create the PostgreSQL addon and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "django-db",
  "type": "postgresql",
  "version": "16-latest",
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId django-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon in the UI"; false; }
```

## Step 7: Create the secret group

Generate a secret key locally without echoing it, then build the JSON with `jq` so the key never appears in your command line or logs:

```bash
DJANGO_SECRET_KEY=$(python3 -c 'import secrets; print(secrets.token_urlsafe(50))')
jq -n --arg key "$DJANGO_SECRET_KEY" '{
  name: "django-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "django-db", keys: [ { keyName: "POSTGRES_URI", aliases: ["DATABASE_URL"] } ] } ],
  secrets: { variables: { DJANGO_SECRET_KEY: $key } }
}' > django-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file django-env.json >/dev/null && rm django-env.json
```

The group is unrestricted, so both the service and the migration job can read it.

## Step 8: Deploy the web service

Django's host check rejects requests whose `Host` header isn't in `ALLOWED_HOSTS`. Platform health probes may not send your public hostname, so use a **TCP** probe on the gunicorn port rather than an HTTP probe. Write `web.json`:

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "8000" },
  "healthChecks": [
    { "protocol": "TCP", "type": "readinessProbe", "port": 8000,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file web.json
```

Use the same repository URL, provider and branch as in step 4. The first build starts automatically.

## Step 9: Set `ALLOWED_HOSTS` and `CSRF_TRUSTED_ORIGINS` from the public host

Read the hostname Northflank assigned to the public port. It's a `*.code.run` name. If it's empty, wait for the rollout in step 10 and read it again.

```bash
HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
  | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
# Older API responses only list the host on the deprecated ports endpoint
[ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
  | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
echo "$HOST"

northflank patch service combined --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json --input "$(jq -n --arg h "$HOST" '{
  runtimeEnvironment: { PORT: "8000", ALLOWED_HOSTS: $h, CSRF_TRUSTED_ORIGINS: ("https://" + $h) }
}')"
northflank restart service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json
```

Send the complete `runtimeEnvironment` map every time you patch it, because existing keys may be replaced. When you add a custom domain later, append it to both values (comma-separated). See [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank).

`CSRF_TRUSTED_ORIGINS` is what lets the admin login form POST over HTTPS when TLS ends at the load balancer. It hasn't been confirmed that the load balancer sets `X-Forwarded-Proto`. If you want `request.is_secure()` to be true, add `SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")` and check that it takes effect.

## Step 10: Wait for the rollout

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

`COMPLETED` means rolled out and serving. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web`. If `HOST` was empty in step 9, redo the step 9 patch now with the host printed here.

## Step 11: Create and run the migrate job

Write `migrate.json`. It builds the same repo and overrides the command. `put job` makes a manual job because `settings.cron` is absent (`put job manual` is deprecated in the API):

```json
{
  "name": "web-migrate",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "settings": { "backoffLimit": 0, "activeDeadlineSeconds": 900, "runOnSourceChange": "never" },
  "deployment": {
    "vcs": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
    "docker": { "configType": "customCommand", "customCommand": "python manage.py migrate --noinput" }
  },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } }
}
```

Use the same repository URL, provider and branch as in step 4.

```bash
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file migrate.json

# wait for the job image; start a build if none has appeared after ~1 minute
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

RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId web-migrate --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 90); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId web-migrate --runId "$RUN_ID" --noDefaults -o json)
  ST=$(echo "$R" | jq -r '(.data // .).status'); echo "run=$ST"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$ST" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId web-migrate --runId "$RUN_ID" --lineLimit 200
[ "$OK" = 1 ] || { echo "MIGRATION_NOT_SUCCEEDED: stop here and read the job logs above"; false; }
```

Run status is `RUNNING`, `SUCCESS` or `FAILED`. For other one-off commands, such as `createsuperuser --noinput` with `DJANGO_SUPERUSER_*` variables, see [run an image once or on a schedule](https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule).

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The migrate run concluded with `status == "SUCCESS"` (step 11 prints it).
3. The admin login page renders (this needs the database, `ALLOWED_HOSTS` and the templates):
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/admin/login/"   # 200
   ```
4. A collected static file is served:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/static/admin/css/base.css"   # 200
   ```
   With `CompressedManifestStaticFilesStorage`, Django templates use hashed file names, but WhiteNoise also serves the original path.
5. No migrations are pending. `exec` needs `script` to keep output without a TTY (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId web \
     --shell-cmd 'sh -c' --cmd '"python manage.py migrate --check && echo NO_PENDING_MIGRATIONS"'
   ```
   This prints `NO_PENDING_MIGRATIONS`. `migrate --check` exits non-zero if any migration is unapplied.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `400 Bad Request` on every URL | Hostname missing from `ALLOWED_HOSTS` | Redo step 9 with the exact `dns` value; restart |
| Admin login returns `403 CSRF verification failed` | `CSRF_TRUSTED_ORIGINS` missing `https://<host>` | Redo step 9 |
| Build fails at `collectstatic` with `KeyError` | Settings need an env var at import time | Pass a throwaway value inline in the `RUN` line, as in the `Dockerfile` above |
| Static files 404, pages render | WhiteNoise middleware missing or `STATIC_ROOT` unset | Add the middleware and `STATIC_ROOT`; rebuild |
| Migrate run `FAILED`, `relation already exists` | Migrations were applied manually before | Inspect with `showmigrations`; ask the user before using `--fake` |
| Pod restarts, logs show `Booting worker` loops | Worker timeout or out of memory | Lower `--workers`, or use a bigger plan; see [view logs](https://northflank.com/docs/v1/application/observe/view-logs) |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` (and the job's `deployment.vcs`) as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service or job until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 10 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print `DJANGO_SECRET_KEY`, `DATABASE_URL` or superuser passwords. Build JSON with `jq --arg` and delete temp files.
- Never set `DJANGO_DEBUG=1` in production to debug a 500. Read the logs with `northflank get service logs` instead.
- Take a backup before running migrations against a database with data (see [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data)). Ask before using `migrate --fake` or rolling migrations back.
- Don't delete the addon, service or job without explicit, per-resource confirmation.

## FAQ

### Where should I deploy my Django project?
Somewhere that runs a long-lived gunicorn process, provides Postgres and lets you run `migrate` as a separate step. A container platform with managed databases fits that shape with the least setup. A VM works too if you're happy to manage nginx, TLS, process supervision and backups yourself.

### Why not run `migrate` in the container start command?
With more than one instance, every instance would race to apply the same migrations on each restart. A job runs once and gives you a pass or fail status.

### Do I need nginx for static files?
No. WhiteNoise serves the files that `collectstatic` wrote into the image. For user uploads (`MEDIA_ROOT`), use object storage or [a persistent volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume), because container filesystems are replaced on every deploy.

### Can I use `ALLOWED_HOSTS = ["*"]`?
It works, but it turns off Django's host header protection. Prefer the exact hostnames.

### How do I run Celery alongside Django?
Deploy a second service from the same repo with a custom command such as `celery -A myproject worker`. See [how to deploy a background worker](deploy-celery-in-production.md).

## Conclusion

You now have Django running behind gunicorn on Northflank, with static files built into the image, host and CSRF settings matching the real URL, a managed Postgres database, and migrations that run as a tracked job.

**Next:** add your own domain with [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank), then automate the migrate step with [run migrations](https://northflank.com/docs/v1/application/release/run-migrations). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy a FastAPI app](deploy-fastapi-app.md)
