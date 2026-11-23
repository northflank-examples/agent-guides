<!--
agent-guide-meta:
  id: NF-STK-SUPERSET
  description: "How to deploy Apache Superset in production with the official image, a fixed SECRET_KEY, PostgreSQL metadata, Redis caching and a one-off init job."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://superset.apache.org/docs/installation/docker-builds/
    - https://superset.apache.org/docs/configuration/configuring-superset/
    - https://github.com/apache/superset/blob/6.1.0/superset/config.py
    - https://github.com/apache/superset/blob/6.1.0/docker/docker-init.sh
    - https://github.com/apache/superset/blob/6.1.0/docker/entrypoints/run-server.sh
    - https://github.com/apache/superset/blob/6.1.0/Dockerfile
    - https://northflank.com/docs/v1/application/release/manage-ci-cd
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-build-service
    - https://northflank.com/docs/v1/api/project/services/start-service-build
    - https://northflank.com/docs/v1/api/project/services/list-service-builds
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/project/jobs/run-job
    - https://northflank.com/docs/v1/api/project/jobs/get-run-details
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-redis-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DB-001, NF-DB-003, NF-DEP-001, NF-STK-AIRFLOW]
-->

# How to deploy Apache Superset

To deploy Apache Superset in production, extend the official `apache/superset` image with the `psycopg2-binary` driver and a `superset_config.py`, set a fixed `SECRET_KEY`, store metadata in PostgreSQL instead of SQLite, point the caches at Redis, and run `superset db upgrade`, admin creation and `superset init` as a one-off job. The default image ships no database drivers, and a changed `SECRET_KEY` breaks every saved database password.

This guide does it on Northflank with the CLI: managed PostgreSQL and Redis linked through a secret group, a build service for your Superset image, an init job that reports pass or fail, and a gunicorn service behind a public HTTPS URL. You end with `/health` returning `OK` and an admin who can sign in. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human links Git once.

## TL;DR: how to deploy Apache Superset

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check the repo, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Create PostgreSQL and Redis with `northflank put addon`, then a secret group with `POSTGRES_URI` as `SUPERSET__SQLALCHEMY_DATABASE_URI`, `REDIS_URL` and a generated `SUPERSET_SECRET_KEY`.
4. Push a `Dockerfile` (`FROM apache/superset:6.1.0` plus `psycopg2-binary`), `superset_config.py` and an init script, build them with `northflank put service build`, and run the init job (`db upgrade`, `create-admin`, `init`).
5. Deploy the web service with `northflank put service deployment` on port 8088.
6. Check that the image was built from your pushed commit, `/health` returns `OK` and the login page returns 200.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Superset's defaults are for a laptop:

- **Metadata in SQLite** (`superset.db` under `SUPERSET_HOME`). Production needs PostgreSQL. Superset 6.1 reads `SUPERSET__SQLALCHEMY_DATABASE_URI` from the environment, so the database URL never goes in a file.
- **A placeholder `SECRET_KEY`.** Superset uses it to sign sessions and encrypt the connection passwords you save for your warehouses. It reads `SUPERSET_SECRET_KEY`; set it once and never change it.
- **No cache** (`NullCache`). Redis makes dashboards and chart data faster and gives the rate limiter shared storage.
- **No drivers.** The Superset docs say tags without a preset are "lean" builds with no database drivers, "AND the metadata database". You extend the image with `psycopg2-binary` and any warehouse drivers.

The init commands (`db upgrade`, `fab create-admin`, `init`) change the database once per release, which is why they run as a job and not on every container start.

Where to deploy Superset comes down to a long-running gunicorn process, PostgreSQL, Redis and HTTPS. Static and serverless hosts can't run it. A VM with Docker Compose works if you run the databases and TLS; Kubernetes with the Helm chart works if you run the cluster. A container platform with managed Postgres and Redis covers it without either.

## Prerequisites

- **Your `superset_config.py` compiles locally** (skip this if you're starting from the one in step 8):
  ```bash
  python3 -m py_compile superset_config.py && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`** for generating secrets.

Names used below: project `superset`, addons `superset-db` and `superset-cache`, secret group `superset-env`, build service `superset-build`, job `superset-init`, service `superset`.

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
grep -E '^FROM' Dockerfile 2>/dev/null
ls superset_config.py superset-init.sh Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId superset -o json >/dev/null 2>&1 \
  && echo "project: superset exists and will be reused" \
  || echo "project: superset will be created"
```

- If the repo already has a `Dockerfile` that isn't `FROM apache/superset`, it belongs to another app. Stop and use a separate repo (or folder) for Superset, or use a guide for your framework.
- If `superset_config.py` already exists, merge the Redis and proxy settings from step 8 into it rather than replacing it.
- If a `Dockerfile` already exists, compare it with step 8 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services, jobs and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "superset", "region": "europe-west" }'
export PROJECT_ID=superset

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

The init job in step 9 deploys the build service's image, so it needs no access check of its own.

## Step 5: Create PostgreSQL and Redis

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "superset-db", "type": "postgresql", "version": "16-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-50", "storage": 8192, "replicas": 1 },
  "customCredentials": { "dbName": "superset" }
}'
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "superset-cache", "type": "redis", "version": "8-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 4096, "replicas": 1 },
  "typeSpecificSettings": { "redisMaxMemoryPolicy": "allkeys-lru", "redisSentinelEnabled": false }
}'

ALL_OK=1
for ADDON in superset-db superset-cache; do
  OK=0
  for i in $(seq 1 90); do
    A=$(northflank get addon --projectId "$PROJECT_ID" --addonId "$ADDON" --noDefaults -o json | jq -r '(.data // .).status')
    case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo "$ADDON FAILED"; break;; esac
    sleep 10
  done; echo "$ADDON=$A"; [ "$OK" = 1 ] || ALL_OK=0
done
[ "$ALL_OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addons in the UI"; false; }
```

`allkeys-lru` suits a cache: Redis evicts old keys instead of refusing writes. Both addons accept connections only from inside the project.

## Step 6: Create the secret group

```bash
jq -n --arg key "$(openssl rand -base64 42)" --arg pw "$(openssl rand -hex 16)" '{
  name: "superset-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [
    { addonId: "superset-db", keys: [ { keyName: "POSTGRES_URI", aliases: ["SUPERSET__SQLALCHEMY_DATABASE_URI"] } ] },
    { addonId: "superset-cache", keys: [ { keyName: "REDIS_MASTER_URL", aliases: ["REDIS_URL"] } ] }
  ],
  secrets: { variables: {
    SUPERSET_SECRET_KEY: $key, SUPERSET_ENV: "production",
    SUPERSET_ADMIN_USER: "admin", SUPERSET_ADMIN_PASSWORD: $pw
  } }
}' > superset-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file superset-env.json >/dev/null && rm superset-env.json
```

`openssl rand -base64 42` is the generator the Superset config comments suggest. `SUPERSET_ENV=production` turns on the rate limiter, which stores its counters in Redis.

## Step 7: Check the Redis link

```bash
northflank get secret-link --projectId "$PROJECT_ID" --secretId superset-env --addonId superset-cache -o json \
  | jq '(.data // .) | {linked, linkedKeys, availableKeys}'
```

If `linkedKeys` is empty, pick the URL key from `availableKeys` and re-link with `northflank update secret-link … --input '{"keys":[{"keyName":"<KEY>","aliases":["REDIS_URL"]}]}'`.

**Tell the user to copy `SUPERSET_SECRET_KEY` and `SUPERSET_ADMIN_PASSWORD` from the secret group in the UI to their password manager.**

## Step 8: Add the files and build the image

Add three files to the repository. `Dockerfile`, following the Superset docs' pattern for extending the lean image:

```dockerfile
FROM apache/superset:6.1.0
USER root
RUN . /app/.venv/bin/activate && uv pip install psycopg2-binary
COPY --chown=superset:superset superset_config.py /app/pythonpath/superset_config.py
COPY --chown=superset:superset superset-init.sh /app/superset-init.sh
USER superset
```

Add your warehouse drivers to the `uv pip install` line. `superset_config.py` (the image puts `/app/pythonpath` on `PYTHONPATH`):

```python
import os

REDIS_URL = os.environ["REDIS_URL"]

def _redis_cache(prefix, timeout):
    return {"CACHE_TYPE": "RedisCache", "CACHE_DEFAULT_TIMEOUT": timeout,
            "CACHE_KEY_PREFIX": prefix, "CACHE_REDIS_URL": REDIS_URL}

CACHE_CONFIG = _redis_cache("superset_", 86400)
DATA_CACHE_CONFIG = _redis_cache("superset_data_", 86400)
FILTER_STATE_CACHE_CONFIG = _redis_cache("superset_filter_", 7776000)
EXPLORE_FORM_DATA_CACHE_CONFIG = _redis_cache("superset_explore_", 604800)
RATELIMIT_STORAGE_URI = REDIS_URL
ENABLE_PROXY_FIX = True  # trust X-Forwarded-* from the load balancer
```

And `superset-init.sh`, the same commands as Superset's own `docker-init.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
superset db upgrade
if superset fab list-users | grep -qw "$SUPERSET_ADMIN_USER"; then
  echo "admin user already exists"
else
  superset fab create-admin --username "$SUPERSET_ADMIN_USER" --password "$SUPERSET_ADMIN_PASSWORD" \
    --firstname Superset --lastname Admin --email admin@example.com
fi
superset init
```

Northflank builds what's on the remote branch, not your local working tree, so commit and push the files first:

```bash
git add Dockerfile superset_config.py superset-init.sh
git commit -m "Add Northflank deployment files"
git push
```

Write `superset-build.json`:

```json
{
  "name": "superset-build",
  "billing": {},
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github" },
  "buildConfiguration": { "branchRestrictions": ["main"] },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } }
}
```

```bash
northflank put service build --projectId "$PROJECT_ID" --noDefaults -o json --file superset-build.json
northflank start service build --projectId "$PROJECT_ID" --serviceId superset-build --noDefaults -o json \
  --input '{ "branch": "main" }'
OK=0
for i in $(seq 1 90); do
  B=$(northflank get service builds --projectId "$PROJECT_ID" --serviceId superset-build --noDefaults -o json \
      | jq -r '(.data // .).builds | sort_by(.createdAt) | last | .status // "NONE"')
  echo "build=$B"
  case "$B" in SUCCESS) OK=1; break;; FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo BUILD_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "BUILD_NOT_COMPLETE: stop here and read the build logs"; false; }
```

Use the same repository URL, provider and branch as in step 4. If it prints `BUILD_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId superset-build --lineLimit 200`.

## Step 9: Run the init job

Write `superset-init.json`. `put job` without `settings.cron` creates a manual job:

```json
{
  "name": "superset-init",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "settings": { "backoffLimit": 0, "activeDeadlineSeconds": 1800, "runOnSourceChange": "never" },
  "deployment": {
    "internal": { "id": "superset-build", "branch": "main", "buildSHA": "latest" },
    "docker": { "configType": "customCommand", "customCommand": "bash /app/superset-init.sh" }
  }
}
```

```bash
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file superset-init.json
RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId superset-init --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 90); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId superset-init --runId "$RUN_ID" --noDefaults -o json)
  RS=$(echo "$R" | jq -r '(.data // .).status'); echo "run=$RS"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$RS" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId superset-init --runId "$RUN_ID" --lineLimit 200
[ "$OK" = 1 ] || { echo "RUN_NOT_SUCCESSFUL: stop here and read the job logs above"; false; }
```

The run should end `SUCCESS`, and the logs should not mention SQLite. If they do, the database alias didn't reach the job.

## Step 10: Deploy the web service

The image's default command runs gunicorn on `0.0.0.0:8088`. Write `superset.json`:

```json
{
  "name": "superset",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "internal": { "id": "superset-build", "branch": "main", "buildSHA": "latest" },
    "docker": { "configType": "default" }
  },
  "ports": [ { "name": "http", "internalPort": 8088, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "SERVER_WORKER_AMOUNT": "2", "GUNICORN_TIMEOUT": "120" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/health", "port": 8088,
      "initialDelaySeconds": 20, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 6, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file superset.json
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId superset --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"; [ "$D" = "COMPLETED" ] && { OK=1; break; }; [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId superset --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId superset --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId superset --lineLimit 200`.

`SERVER_WORKER_AMOUNT` and `GUNICORN_TIMEOUT` are read by the image's `run-server.sh`. Raise the timeout if slow warehouse queries hit it.

## How to know it worked

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId superset-build --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The init run concluded with `status == "SUCCESS"`.
3. Health and login respond:
   ```bash
   curl -s "https://$HOST/health"                                         # OK
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/login/"        # 200
   ```
4. Superset is on PostgreSQL and Redis. This prints only scheme prefixes (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId superset \
     --shell-cmd 'sh -c' --cmd '"printenv SUPERSET__SQLALCHEMY_DATABASE_URI | cut -c1-11; printenv REDIS_URL | cut -c1-6"'
   ```
   Expect `postgresql:` and `redis:`.
5. **This step needs a human:** sign in at `https://$HOST` as `admin` with the password from the secret group, then add a database connection under **Settings → Database connections**.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the build service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| Logs warn about the default `SECRET_KEY` and refuse to start | `SUPERSET_SECRET_KEY` not inherited | Check the group is unrestricted; restart |
| `No module named 'psycopg2'` | Driver not installed | Check the `uv pip install` line; rebuild |
| Init job creates `superset.db` | Database alias missing | Check `addonDependencies` in `superset-env`; re-run the job |
| `Connection refused` to Redis | `REDIS_URL` empty or wrong key | Redo step 7 |
| Login redirects to `http://` | Proxy headers not trusted | Keep `ENABLE_PROXY_FIX = True`; report if it persists |
| Saved database passwords fail to decrypt | `SECRET_KEY` changed | Restore the old key; see Superset's `superset re-encrypt-secrets` for planned rotation |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the build service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the image was built from the commit you pushed.
- If the step 10 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print `SUPERSET_SECRET_KEY`, the admin password or connection URIs. Print key names and scheme prefixes only.
- Never change `SUPERSET_SECRET_KEY` on a running install.
- Back up the metadata database before running the init job for a new Superset version (see [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data)).
- Don't delete the addons, service or build service without explicit, per-resource confirmation.

## FAQ

### Where should I deploy Apache Superset?
On a host that runs a long-lived gunicorn process next to PostgreSQL and Redis, behind HTTPS. A container platform with managed databases is the least work. A VM with Docker Compose suits small teams that will run backups and TLS themselves.

### Do I need Celery workers?
Not to start. Dashboards and SQL Lab work without them. Alerts and reports, and async queries for long-running SQL, need Celery workers and beat using Redis as the broker; deploy them from the same image with `celery --app=superset.tasks.celery_app:app worker`.

### How do I upgrade Superset?
Change the `FROM` tag, push, wait for the build, run the init job, then `northflank restart service --projectId "$PROJECT_ID" --serviceId superset`.

## Conclusion

You now have Superset on the official image with a fixed secret key, PostgreSQL metadata, Redis caching, a tracked init job and a public HTTPS URL.

**Next:** add a custom domain with [link a domain to a port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port), and connect your warehouse. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy ClickHouse](deploy-clickhouse.md)
- [How to deploy Apache Airflow](deploy-apache-airflow.md)
