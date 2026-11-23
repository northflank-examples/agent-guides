<!--
agent-guide-meta:
  id: NF-STK-AIRFLOW
  description: "How to deploy Apache Airflow 3 in production with the official image, managed PostgreSQL, DAGs built from Git, db migrate as a job and LocalExecutor or Celery."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml
    - https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html
    - https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/logging-monitoring/check-health.html
    - https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html
    - https://airflow.apache.org/docs/docker-stack/entrypoint.html
    - https://airflow.apache.org/docs/docker-stack/build.html
    - https://pypi.org/project/apache-airflow-providers-postgres/
    - https://github.com/apache/airflow/blob/3.4.0/Dockerfile
    - https://airflow.apache.org/docs/apache-airflow-providers-celery/stable/configurations-ref.html
    - https://airflow.apache.org/docs/apache-airflow-providers-fab/stable/cli-ref.html
    - https://github.com/apache/airflow/blob/main/scripts/docker/entrypoint_prod.sh
    - https://northflank.com/docs/v1/application/release/manage-ci-cd
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-build-service
    - https://northflank.com/docs/v1/api/project/services/start-service-build
    - https://northflank.com/docs/v1/api/project/services/list-service-builds
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/project/jobs/run-job
    - https://northflank.com/docs/v1/api/project/jobs/get-run-details
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
  related: [NF-DB-001, NF-DB-003, NF-JOB-002, NF-DEP-001]
-->

# How to deploy Apache Airflow

To deploy Apache Airflow 3 in production, build an image from the official `apache/airflow` image with your DAGs copied in from Git, use PostgreSQL as the metadata database, run `airflow db migrate` once as a separate job, and run the API server, scheduler, DAG processor and triggerer as separate processes sharing one Fernet key and JWT secret. Mismatched secrets or an unmigrated database are the usual reasons components fail.

This guide does it on Northflank with the CLI: managed PostgreSQL, a build service that bakes your DAGs into the image, a migrate job with a clear pass or fail, and four LocalExecutor services, with optional Celery and Redis. You end with a public HTTPS Airflow UI whose health endpoint reports every component healthy. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human links Git once.

## TL;DR: how to deploy Apache Airflow

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check the repo's DAGs and `Dockerfile`, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Create PostgreSQL with `northflank put addon` and a secret group that links `POSTGRES_URI` as `AIRFLOW__DATABASE__SQL_ALCHEMY_CONN`, plus generated Fernet, API and JWT secrets.
4. Push a `Dockerfile` (`FROM apache/airflow:3.4.0`) with your DAGs, build it once with `northflank put service build`, and run `airflow db migrate` as a manual job.
5. Deploy `api-server`, `scheduler`, `dag-processor` and `triggerer` with `northflank put service deployment`.
6. Check that the image was built from your pushed commit, `/api/v2/monitor/health` reports every component healthy and a triggered DAG run succeeds.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Airflow 3 split the old webserver and scheduler into more processes. The official Docker Compose file runs them as separate containers:

- **`api-server`** serves the UI and REST API on port 8080, plus the Execution API that running tasks report to.
- **`scheduler`** decides what runs and, with the LocalExecutor, runs the tasks itself.
- **`dag-processor`** parses DAG files into the database. In Airflow 3 it no longer runs inside the scheduler.
- **`triggerer`** runs deferred tasks.
- **`airflow db migrate`** runs once per upgrade, so it belongs in a job, not in a container start command.

**Executor choice.** The LocalExecutor runs tasks as subprocesses of the scheduler. It needs no extra infrastructure and suits most teams to start with; you scale it by giving the scheduler a bigger plan. The CeleryExecutor adds Redis as a broker and separate worker services that you can scale out. Start with Local and switch when one machine's worth of tasks isn't enough (see the optional step).

**DAG delivery.** Every component that parses or runs DAGs needs the same files. Baking them into the image from Git means one build, one version, and a redeploy per change, with no shared volume. Volumes on Northflank are `ReadWriteOnce`, so a shared DAG folder across services isn't an option anyway.

Where to deploy Airflow comes down to what it needs: several long-running processes, PostgreSQL and a public HTTPS URL. Serverless functions can't host a scheduler. A VM or Kubernetes with the Helm chart works if you run the database, TLS and cluster yourself; a container platform with managed databases runs the same image without that work.

## Prerequisites

- **Your DAG files compile locally** (skip this if you're starting from the test DAG in step 7):
  ```bash
  python3 -m py_compile dags/*.py && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`** for generating secrets.

Names used below: project `airflow`, addon `airflow-db`, secret group `airflow-env`, build service `airflow-build`, job `airflow-migrate`, services `airflow-api`, `airflow-scheduler`, `airflow-dag-processor`, `airflow-triggerer`.

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
ls -d dags 2>/dev/null && ls dags/*.py
grep -E '^FROM' Dockerfile 2>/dev/null
grep -hiE '^apache-airflow' requirements.txt 2>/dev/null
ls scripts/airflow-init.sh requirements.txt 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId airflow -o json >/dev/null 2>&1 \
  && echo "project: airflow exists and will be reused" \
  || echo "project: airflow will be created"
```

- If the repo already has a `Dockerfile` that isn't `FROM apache/airflow`, it belongs to another app. Stop and use a separate repo (or folder) for Airflow, or use a guide for your framework.
- If there's no `dags/` folder, step 7 creates one with a test DAG. A `requirements.txt` means extra Python packages: install them as step 7 shows, pinned to the same `apache-airflow` version.
- If a `Dockerfile` already exists, compare it with step 7 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services, jobs and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "airflow", "region": "europe-west" }'
export PROJECT_ID=airflow

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

The migrate job in step 8 deploys the build service's image, so it needs no access check of its own.

## Step 5: Create PostgreSQL and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "airflow-db", "type": "postgresql", "version": "16-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-50", "storage": 8192, "replicas": 1 },
  "customCredentials": { "dbName": "airflow" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId airflow-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

The database only accepts connections from inside the project. Storage can only be increased later.

## Step 6: Create the shared secret group

Every Airflow component must share the same Fernet key (encrypts connection passwords), API secret key and JWT secret. Generate them without echoing and put them in one unrestricted group, so the services and the job all inherit it:

```bash
FERNET=$(openssl rand -base64 32 | tr '+/' '-_')
jq -n --arg fernet "$FERNET" --arg api "$(openssl rand -hex 32)" --arg jwt "$(openssl rand -hex 32)" \
      --arg pw "$(openssl rand -hex 16)" '{
  name: "airflow-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "airflow-db", keys: [
    { keyName: "POSTGRES_URI", aliases: ["AIRFLOW__DATABASE__SQL_ALCHEMY_CONN"] } ] } ],
  secrets: { variables: {
    AIRFLOW__CORE__EXECUTOR: "LocalExecutor",
    AIRFLOW__CORE__AUTH_MANAGER: "airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager",
    AIRFLOW__CORE__EXECUTION_API_SERVER_URL: "http://airflow-api:8080/execution/",
    AIRFLOW__CORE__HOSTNAME_CALLABLE: "airflow.utils.net.get_host_ip_address",
    AIRFLOW__CORE__LOAD_EXAMPLES: "false",
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: "true",
    AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: "true",
    AIRFLOW__CORE__FERNET_KEY: $fernet,
    AIRFLOW__API__SECRET_KEY: $api,
    AIRFLOW__API_AUTH__JWT_SECRET: $jwt,
    AIRFLOW_ADMIN_USER: "admin",
    AIRFLOW_ADMIN_PASSWORD: $pw
  } }
}' > airflow-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file airflow-env.json >/dev/null && rm airflow-env.json
unset FERNET
```

Notes:

- `AIRFLOW__CORE__EXECUTION_API_SERVER_URL` uses the private address `[service-name]:[port]`, so tasks reach the API server inside the project. Without it, task processes try `localhost` and fail.
- The FAB auth manager gives you username and password logins and `airflow users create`, as in the upstream Compose file. The default Simple auth manager is meant for development.
- `hostname_callable` makes each component register its pod IP, so the UI can fetch task logs from the scheduler's log server on port 8793.

**Tell the user to copy `AIRFLOW_ADMIN_PASSWORD` and `AIRFLOW__CORE__FERNET_KEY` from the secret group in the UI to their password manager.** Losing the Fernet key makes stored connection passwords unreadable.

## Step 7: Add the files and build the image from Git

Add these files to the repository, with `Dockerfile` at the root:

```dockerfile
FROM apache/airflow:3.4.0
COPY --chown=airflow:root dags/ /opt/airflow/dags/
COPY --chown=airflow:root scripts/ /opt/airflow/scripts/
```

`scripts/airflow-init.sh` migrates the database and creates the first admin user once:

```bash
#!/usr/bin/env bash
set -euo pipefail
airflow db migrate
if airflow users list | grep -qw "$AIRFLOW_ADMIN_USER"; then
  echo "admin user already exists"
else
  airflow users create --username "$AIRFLOW_ADMIN_USER" --password "$AIRFLOW_ADMIN_PASSWORD" \
    --firstname Airflow --lastname Admin --role Admin --email admin@example.com
fi
```

And `dags/hello.py` is a test DAG using the Airflow 3 SDK:

```python
from datetime import datetime
from airflow.sdk import dag, task

@dag(schedule=None, start_date=datetime(2025, 1, 1), catchup=False)
def hello():
    @task
    def say_hi():
        print("hello from airflow")
    say_hi()

hello()
```

For extra Python packages, add `RUN pip install --no-cache-dir "apache-airflow==3.4.0" -r requirements.txt` after a `COPY requirements.txt .`, as the Airflow image docs recommend. Don't use `_PIP_ADDITIONAL_REQUIREMENTS` in production.

Northflank builds what's on the remote branch, not your local working tree, so commit and push the files first:

```bash
git add Dockerfile dags/ scripts/
git commit -m "Add Northflank deployment files"
git push
```

A build service builds the image once; every component and the job deploy that same image. Write `airflow-build.json`:

```json
{
  "name": "airflow-build",
  "billing": {},
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github" },
  "buildConfiguration": { "branchRestrictions": ["main"] },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } }
}
```

```bash
northflank put service build --projectId "$PROJECT_ID" --noDefaults -o json --file airflow-build.json
northflank start service build --projectId "$PROJECT_ID" --serviceId airflow-build --noDefaults -o json \
  --input '{ "branch": "main" }'

OK=0
for i in $(seq 1 90); do
  B=$(northflank get service builds --projectId "$PROJECT_ID" --serviceId airflow-build --noDefaults -o json \
      | jq -r '(.data // .).builds | sort_by(.createdAt) | last | .status // "NONE"')
  echo "build=$B"
  case "$B" in SUCCESS) OK=1; break;; FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo BUILD_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "BUILD_NOT_COMPLETE: stop here and read the build logs"; false; }
```

If it prints `BUILD_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId airflow-build --lineLimit 200`.

Use the same repository URL, provider and branch as in step 4: `projectType` can be `github`, `gitlab` or `bitbucket`. Pushes to the branch rule build new images automatically.

## Step 8: Migrate the database and create the admin user

Write `airflow-migrate.json`. `put job` without `settings.cron` creates a manual job:

```json
{
  "name": "airflow-migrate",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "settings": { "backoffLimit": 0, "activeDeadlineSeconds": 1800, "runOnSourceChange": "never" },
  "deployment": {
    "internal": { "id": "airflow-build", "branch": "main", "buildSHA": "latest" },
    "docker": { "configType": "customCommand", "customCommand": "bash /opt/airflow/scripts/airflow-init.sh" }
  }
}
```

```bash
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file airflow-migrate.json
RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId airflow-migrate --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 90); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId airflow-migrate --runId "$RUN_ID" --noDefaults -o json)
  RS=$(echo "$R" | jq -r '(.data // .).status'); echo "run=$RS"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$RS" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId airflow-migrate --runId "$RUN_ID" --lineLimit 200
[ "$OK" = 1 ] || { echo "RUN_NOT_SUCCESSFUL: stop here and read the job logs above"; false; }
```

The image's entrypoint waits for the database (`airflow db check`) and then passes `bash …` through. `airflow db migrate` also migrates the FAB tables, because Airflow 3 discovers provider database managers automatically. Run status is `RUNNING`, `SUCCESS` or `FAILED`. Re-run this job before every Airflow upgrade.

## Step 9: Deploy the API server and scheduler

Write `airflow-api.json`:

```json
{
  "name": "airflow-api",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "internal": { "id": "airflow-build", "branch": "main", "buildSHA": "latest" },
    "docker": { "configType": "customCommand", "customCommand": "api-server" }
  },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/api/v2/monitor/health", "port": 8080,
      "initialDelaySeconds": 20, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 6, "successThreshold": 1 }
  ]
}
```

And `airflow-scheduler.json`. Port 8793 is the log server, 8974 the scheduler health server; both stay private:

```json
{
  "name": "airflow-scheduler",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "internal": { "id": "airflow-build", "branch": "main", "buildSHA": "latest" },
    "docker": { "configType": "customCommand", "customCommand": "scheduler" }
  },
  "ports": [
    { "name": "logs", "internalPort": 8793, "public": false, "protocol": "HTTP" },
    { "name": "health", "internalPort": 8974, "public": false, "protocol": "HTTP" }
  ],
  "healthChecks": [
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/health", "port": 8974,
      "initialDelaySeconds": 60, "periodSeconds": 30, "timeoutSeconds": 10, "failureThreshold": 5 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file airflow-api.json
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file airflow-scheduler.json
```

## Step 10: Deploy the DAG processor and triggerer

Both have no ports; only the command differs:

```bash
for C in dag-processor triggerer; do
  jq -n --arg c "$C" '{
    name: ("airflow-" + $c),
    billing: { deploymentPlan: "nf-compute-100-1" },
    deployment: { instances: 1,
      internal: { id: "airflow-build", branch: "main", buildSHA: "latest" },
      docker: { configType: "customCommand", customCommand: $c } }
  }' > "airflow-$C.json"
  northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file "airflow-$C.json"
done
```

## Step 11: Wait for the rollout and read the URL

```bash
ALL_OK=1
for S in airflow-api airflow-scheduler airflow-dag-processor airflow-triggerer; do
  OK=0
  for i in $(seq 1 60); do
    D=$(northflank get service --projectId "$PROJECT_ID" --serviceId "$S" --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
    [ "$D" = "COMPLETED" ] && { OK=1; break; }
    [ "$D" = "FAILED" ] && { echo "$S DEPLOY_FAILED"; break; }
    sleep 10
  done; echo "$S=$D"; [ "$OK" = 1 ] || ALL_OK=0
done
[ "$ALL_OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$ALL_OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId airflow-api --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId airflow-api --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs of the service that didn't complete: `northflank get service logs --projectId "$PROJECT_ID" --serviceId <id> --lineLimit 200`.

`COMPLETED` means rolled out and serving. To ship new DAGs, push to the branch, wait for the build as in step 7, then run `northflank restart service --projectId "$PROJECT_ID" --serviceId <id>` for each of the four services so they pick up the new image.

## How to know it worked

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId airflow-build --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The migrate run concluded with `status == "SUCCESS"`.
3. Every component reports healthy:
   ```bash
   curl -s "https://$HOST/api/v2/monitor/health" | jq '{db: .metadatabase.status, scheduler: .scheduler.status, triggerer: .triggerer.status, dag_processor: .dag_processor.status}'
   ```
   Expect `"healthy"` for each.
4. The test DAG is parsed and a run succeeds. `exec` needs `script` to keep output without a TTY (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId airflow-scheduler \
     --shell-cmd 'bash -c' --cmd '"airflow dags unpause hello && airflow dags trigger hello"'
   sleep 60
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId airflow-scheduler \
     --shell-cmd 'bash -c' --cmd '"airflow dags list-runs hello"'
   ```
   The newest run shows `success`.
5. **This step needs a human:** sign in at `https://$HOST` as `admin` with the password from the secret group, and open the run's task log.

## Optional: switch to the CeleryExecutor with Redis

1. Create a Redis addon (`"type": "redis"`, `"tlsEnabled": false`) the same way as step 5.
2. Link it into `airflow-env` with `{ "addonId": "<redis-id>", "keys": [ { "keyName": "REDIS_MASTER_URL", "aliases": ["AIRFLOW__CELERY__BROKER_URL"] } ] }`, and set `AIRFLOW__CORE__EXECUTOR` to `CeleryExecutor`. Check the key name with `northflank get secret-link --projectId "$PROJECT_ID" --secretId airflow-env --addonId <redis-id> -o json | jq '(.data // .) | {linkedKeys, availableKeys}'`; if `linkedKeys` is empty, re-link with the right key from `availableKeys`. The result backend defaults to the metadata database with a `db+` prefix.
3. Deploy `airflow-worker` like step 10 with `customCommand: "celery worker"`, private port 8793 named `logs`, and runtime variable `DUMB_INIT_SETSID=0` for warm shutdown, as in the upstream Compose file. Add instances to scale out.
4. Restart the other four services so they pick up the new executor.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the build service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| Migrate job fails with `could not translate host name` | Group not inherited, or link missing | Check `airflow-env` has `addonDependencies`; re-run |
| `ModuleNotFoundError: psycopg2` | Driver missing for `postgresql://` | Add `RUN pip install --no-cache-dir "apache-airflow==3.4.0" psycopg2-binary` to the `Dockerfile` |
| Tasks fail with connection refused to `localhost:8080` | `EXECUTION_API_SERVER_URL` unset | Set it as in step 6; restart all services |
| `Invalid token` / `Signature verification failed` in task logs | JWT or API secret differs between components | Keep them only in the shared group; restart all |
| UI shows "Could not read served logs" | Log server unreachable at pod IP | Check port 8793 on the scheduler, or configure [remote logging](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html) to S3 or [MinIO](deploy-minio.md) |
| DAG missing in the UI | DAG processor not running or import error | `northflank get service logs --projectId "$PROJECT_ID" --serviceId airflow-dag-processor` |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the build service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the image was built from the commit you pushed.
- If the step 11 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print the Fernet key, JWT secret, API secret, admin password or database URI. Build JSON with `jq --arg` and delete temp files.
- Never regenerate `AIRFLOW__CORE__FERNET_KEY` on a running install; stored connections become unreadable.
- Run the migrate job before restarting services on a new Airflow version, and take a database backup first (see [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data)).
- Don't delete the addon, services or build service without explicit, per-resource confirmation.

## FAQ

### Where should I deploy Apache Airflow?
Somewhere that runs four or five long-lived processes from one image, provides PostgreSQL and serves the UI over HTTPS. A container platform with managed Postgres covers all of it. A VM with Docker Compose works for small installs if you handle TLS, backups and restarts; Kubernetes with the official Helm chart suits teams that already run a cluster.

### LocalExecutor or CeleryExecutor?
Start with LocalExecutor: no broker, fewer services, and tasks run on the scheduler. Move to Celery when tasks need more CPU or memory than one scheduler plan can give, or when you want workers to scale independently.

### Can I sync DAGs from Git without rebuilding?
Airflow 3 supports DAG bundles, including a Git bundle configured through `[dag_processor] dag_bundle_config_list`. It needs a Git connection; the official image already includes the Git provider. Baking DAGs into the image, as here, is simpler and keeps code and dependencies on the same version.

### How do I upgrade Airflow?
Change the `FROM` tag, push, wait for the build, run the migrate job, then restart the four services.

## Conclusion

You now have Airflow 3 on the official image, with a managed PostgreSQL metadata database, DAGs built from Git into one image, a tracked migrate job, and the API server, scheduler, DAG processor and triggerer each running as their own service.

**Next:** add a custom domain with [link a domain to a port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port), and switch to Celery when one scheduler is no longer enough. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy Dagster](deploy-dagster.md)
- [How to self-host Prefect](self-host-prefect.md)
