<!--
agent-guide-meta:
  id: NF-STK-DAGSTER
  description: "How to deploy Dagster open source in production with a webserver, daemon and gRPC code location server, PostgreSQL storage in dagster.yaml and a protected UI."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://dagster.io/docs/deployment/oss/deployment-options/docker
    - https://dagster.io/docs/deployment/oss/dagster-yaml
    - https://github.com/dagster-io/dagster/tree/master/examples/deploy_docker
    - https://github.com/dagster-io/dagster/blob/master/python_modules/dagster/dagster/_core/storage/config.py
    - https://github.com/dagster-io/dagster/blob/1.13.31/python_modules/libraries/dagster-postgres/dagster_postgres/utils.py
    - https://dagster.io/docs/deployment/execution/run-launchers
    - https://pypi.org/project/dagster/
    - https://northflank.com/docs/v1/application/release/manage-ci-cd
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-build-service
    - https://northflank.com/docs/v1/api/project/services/start-service-build
    - https://northflank.com/docs/v1/api/project/services/list-service-builds
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
  related: [NF-DB-001, NF-DEP-001, NF-STK-AIRFLOW, NF-STK-MINIO]
-->

# How to deploy Dagster

To deploy Dagster in production, run three processes from your project's image: a gRPC code location server (`dagster api grpc`) that loads your definitions, the `dagster-webserver` UI pointed at it through `workspace.yaml`, and `dagster-daemon` for schedules, sensors and the run queue. All three share one `dagster.yaml` that puts run, event log and schedule storage in PostgreSQL. Without the daemon, schedules never fire and queued runs never start.

This guide does it on Northflank with the CLI: managed PostgreSQL linked as `DAGSTER_PG_URL`, one build service for your project, three services on private networking, and basic auth on the UI because Dagster open source has no login. You end with the UI on a public HTTPS URL and a launched job that succeeds. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human links Git once.

## TL;DR: how to deploy Dagster

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a Dagster project, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Create PostgreSQL with `northflank put addon` and link `POSTGRES_URI` as `DAGSTER_PG_URL` in a secret group with `northflank put secret`.
4. Push a `Dockerfile`, `dagster.yaml` and `workspace.yaml`, and build the image with `northflank put service build`.
5. Deploy `dagster-code`, `dagster-webserver` and `dagster-daemon` with `northflank put service deployment`, the webserver behind basic auth.
6. Check that the image was built from your pushed commit, the code location loads and a launched job ends in `SUCCESS`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Dagster's own Docker deployment example splits the work the same way:

- **Code location server.** Loads your assets and jobs and serves them over gRPC on port 4000. The webserver and daemon never import your code; redeploying it doesn't restart the UI.
- **`dagster-webserver`.** The UI and GraphQL API on port 3000. It finds code locations through `workspace.yaml`.
- **`dagster-daemon`.** Evaluates schedules and sensors and dequeues runs from the `QueuedRunCoordinator`.
- **`dagster.yaml`** (the instance config in `$DAGSTER_HOME`). Every process must read the same one, or they won't see each other's runs.

The upstream example uses the `DockerRunLauncher`, which needs the Docker socket. On a container platform you keep the default `DefaultRunLauncher`, which, per the Dagster docs, "spawns a new process per run on the same node as the job's code location". Runs execute inside the `dagster-code` container, so size that service for your workloads.

Where to deploy Dagster comes down to three long-running processes, PostgreSQL and private networking between them. Serverless hosts can't run the daemon. A VM with Docker Compose works if you run Postgres and TLS; Kubernetes with the Dagster Helm chart works if you run the cluster. A container platform with managed Postgres runs the same three processes with less to operate.

## Prerequisites

- **Your Dagster definitions load locally** (skip this if you're starting from the test `definitions.py` in step 7):
  ```bash
  dagster definitions validate -f definitions.py && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`** for generating the UI password.

Names used below: project `dagster`, addon `dagster-db`, secret group `dagster-env`, build service `dagster-build`, services `dagster-code`, `dagster-webserver`, `dagster-daemon`.

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
grep -hiE '^\s*"?dagster' requirements.txt pyproject.toml setup.py 2>/dev/null || echo "dagster dependency: not listed"
grep -lE 'Definitions\(' $(git ls-files '*.py') 2>/dev/null
ls Dockerfile dagster.yaml workspace.yaml 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId dagster -o json >/dev/null 2>&1 \
  && echo "project: dagster exists and will be reused" \
  || echo "project: dagster will be created"
```

- If no `dagster` dependency and no file with `Definitions(` print, this isn't a Dagster project yet. That's fine if you're starting from the test `definitions.py` in step 7; if the repo is a different app, stop and use a guide for your framework.
- The file with `Definitions(` is the one the code server loads (`-f definitions.py` in step 8). Pin the same `dagster` version you use locally in the `Dockerfile`.
- If `dagster.yaml` or `workspace.yaml` already exists, compare it with step 7: storage must come from `DAGSTER_PG_URL` and the gRPC host must be `dagster-code`.
- If a `Dockerfile` already exists, compare it with step 7 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "dagster", "region": "europe-west" }'
export PROJECT_ID=dagster

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Create PostgreSQL

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "dagster-db", "type": "postgresql", "version": "16-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-50", "storage": 8192, "replicas": 1 },
  "customCredentials": { "dbName": "dagster" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId dagster-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

Dagster creates its tables on first connection, so no separate migrate step is needed for a new install.

## Step 6: Link the database into a secret group

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "dagster-env", "type": "secret", "secretType": "environment", "priority": 10,
  "restrictions": { "restricted": false },
  "addonDependencies": [ { "addonId": "dagster-db", "keys": [ { "keyName": "POSTGRES_URI", "aliases": ["DAGSTER_PG_URL"] } ] } ]
}'
```

`postgres_url` in `dagster.yaml` reads it at startup. Dagster rewrites a `postgresql://` URL to use psycopg2, which `dagster-postgres` installs.

## Step 7: Add the files and build the image

Add these files at the repository root. `Dockerfile` (one image for all three processes; pin versions that match):

```dockerfile
FROM python:3.12-slim
RUN pip install --no-cache-dir dagster==1.13.31 dagster-webserver==1.13.31 dagster-postgres==0.29.31
ENV DAGSTER_HOME=/opt/dagster/dagster_home
RUN mkdir -p $DAGSTER_HOME
COPY dagster.yaml $DAGSTER_HOME/
WORKDIR /opt/dagster/app
COPY workspace.yaml definitions.py ./
```

`dagster.yaml`:

```yaml
storage:
  postgres:
    postgres_url:
      env: DAGSTER_PG_URL
run_coordinator:
  module: dagster.core.run_coordinator
  class: QueuedRunCoordinator
  config:
    max_concurrent_runs: 5
scheduler:
  module: dagster.core.scheduler
  class: DagsterDaemonScheduler
telemetry:
  enabled: false
```

`workspace.yaml` (the host is the code service's name):

```yaml
load_from:
  - grpc_server:
      host: dagster-code
      port: 4000
      location_name: "my_code"
```

And `definitions.py`, a test asset and job:

```python
import dagster as dg

@dg.asset
def hello() -> str:
    return "hello from dagster"

hello_job = dg.define_asset_job("hello_job", selection=[hello])
defs = dg.Definitions(assets=[hello], jobs=[hello_job])
```

Add your own packages to the `pip install` line or a `requirements.txt`.

Northflank builds what's on the remote branch, not your local working tree, so commit and push the files first:

```bash
git add Dockerfile dagster.yaml workspace.yaml definitions.py
git commit -m "Add Northflank deployment files"
git push
```

Write `dagster-build.json`:

```json
{
  "name": "dagster-build",
  "billing": {},
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github" },
  "buildConfiguration": { "branchRestrictions": ["main"] },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } }
}
```

```bash
northflank put service build --projectId "$PROJECT_ID" --noDefaults -o json --file dagster-build.json
northflank start service build --projectId "$PROJECT_ID" --serviceId dagster-build --noDefaults -o json \
  --input '{ "branch": "main" }'
OK=0
for i in $(seq 1 90); do
  B=$(northflank get service builds --projectId "$PROJECT_ID" --serviceId dagster-build --noDefaults -o json \
      | jq -r '(.data // .).builds | sort_by(.createdAt) | last | .status // "NONE"')
  echo "build=$B"
  case "$B" in SUCCESS) OK=1; break;; FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo BUILD_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "BUILD_NOT_COMPLETE: stop here and read the build logs"; false; }
```

Use the same repository URL, provider and branch as in step 4. If it prints `BUILD_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId dagster-build --lineLimit 200`.

## Step 8: Deploy the code location server

Write `dagster-code.json`. Port 4000 stays private; only the webserver and daemon call it:

```json
{
  "name": "dagster-code",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "internal": { "id": "dagster-build", "branch": "main", "buildSHA": "latest" },
    "docker": { "configType": "customCommand", "customCommand": "dagster api grpc -h 0.0.0.0 -p 4000 -f definitions.py" }
  },
  "ports": [ { "name": "grpc", "internalPort": 4000, "public": false, "protocol": "TCP" } ],
  "healthChecks": [
    { "protocol": "TCP", "type": "readinessProbe", "port": 4000,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 6, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file dagster-code.json
```

## Step 9: Deploy the webserver and daemon

The webserver gets a public port with basic auth. The daemon has no ports:

```bash
UI_PW=$(openssl rand -hex 16)
jq -n --arg pw "$UI_PW" '{
  name: "dagster-webserver",
  billing: { deploymentPlan: "nf-compute-100-1" },
  deployment: { instances: 1,
    internal: { id: "dagster-build", branch: "main", buildSHA: "latest" },
    docker: { configType: "customCommand", customCommand: "dagster-webserver -h 0.0.0.0 -p 3000 -w workspace.yaml" } },
  ports: [ { name: "http", internalPort: 3000, public: true, protocol: "HTTP",
             security: { credentials: [ { type: "basic-auth", username: "admin", password: $pw } ] } } ],
  healthChecks: [ { protocol: "TCP", type: "readinessProbe", port: 3000, initialDelaySeconds: 10,
                    periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 6, successThreshold: 1 } ]
}' > dagster-webserver.json
jq -n '{
  name: "dagster-daemon",
  billing: { deploymentPlan: "nf-compute-50" },
  deployment: { instances: 1,
    internal: { id: "dagster-build", branch: "main", buildSHA: "latest" },
    docker: { configType: "customCommand", customCommand: "dagster-daemon run" } }
}' > dagster-daemon.json
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file dagster-webserver.json >/dev/null && rm dagster-webserver.json
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file dagster-daemon.json

ALL_OK=1
for S in dagster-code dagster-webserver dagster-daemon; do
  OK=0
  for i in $(seq 1 60); do
    D=$(northflank get service --projectId "$PROJECT_ID" --serviceId "$S" --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
    [ "$D" = "COMPLETED" ] && { OK=1; break; }; [ "$D" = "FAILED" ] && { echo "$S DEPLOY_FAILED"; break; }; sleep 10
  done; echo "$S=$D"; [ "$OK" = 1 ] || ALL_OK=0
done
[ "$ALL_OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$ALL_OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId dagster-webserver --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId dagster-webserver --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs of the service that didn't complete: `northflank get service logs --projectId "$PROJECT_ID" --serviceId <id> --lineLimit 200`.

Run only one daemon instance. `dagster-daemon run` finds `workspace.yaml` in the working directory. **Hand `UI_PW` to the user through a password manager**, and `unset UI_PW` after the checks below.

## How to know it worked

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId dagster-build --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The UI answers behind basic auth:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"                     # 401
   curl -s -o /dev/null -w '%{http_code}\n' -u "admin:$UI_PW" "https://$HOST/"   # 200
   ```
3. The code location loaded:
   ```bash
   curl -s -u "admin:$UI_PW" "https://$HOST/graphql" -H 'Content-Type: application/json' \
     -d '{"query":"{ workspaceOrError { ... on Workspace { locationEntries { name loadStatus } } } }"}' | jq .
   ```
   Expect `my_code` with `LOADED`.
4. A queued run is picked up by the daemon and succeeds. `exec` needs `script` without a TTY (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId dagster-webserver \
     --shell-cmd 'sh -c' --cmd '"dagster job launch -w workspace.yaml -l my_code -j hello_job"'
   sleep 60
   curl -s -u "admin:$UI_PW" "https://$HOST/graphql" -H 'Content-Type: application/json' \
     -d '{"query":"{ runsOrError(limit: 1) { ... on Runs { results { jobName status } } } }"}' | jq .
   ```
   The newest `hello_job` run shows `SUCCESS`. If it stays `QUEUED`, the daemon isn't running.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the build service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| UI shows `my_code` with a connection error | Code server not ready, or wrong host | Check `dagster-code` is `COMPLETED`; host must be `dagster-code` |
| `DagsterInvalidConfigError` on `postgres_url` | `DAGSTER_PG_URL` empty | Check `dagster-env` links `POSTGRES_URI`; restart |
| Runs stay `QUEUED` | Daemon not running | `northflank get service logs --projectId "$PROJECT_ID" --serviceId dagster-daemon` |
| Run step logs (stdout/stderr) empty in the UI | Compute logs stored on the code container's disk | Use `S3ComputeLogManager` from `dagster-aws` with S3 or [MinIO](deploy-minio.md) |
| Runs killed or out of memory | Runs execute in `dagster-code` | Give `dagster-code` a bigger plan |
| New code not visible after push | Services still on the old image | Restart `dagster-code`, then reload the location in the UI |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the build service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the image was built from the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print `DAGSTER_PG_URL` or the UI password.
- Never remove the basic-auth rule from the webserver port without the user's agreement; Dagster open source has no login and can launch any job.
- Keep `dagster-daemon` at one instance.
- Before upgrading Dagster, back up the database and run `dagster instance migrate` from one container. Don't delete the addon or services without explicit, per-resource confirmation.

## FAQ

### Where should I deploy Dagster?
Somewhere that runs three long-lived processes with private networking between them and a PostgreSQL database. A container platform with managed Postgres fits; Kubernetes with the Dagster Helm chart fits teams that already run a cluster. Dagster+ is the managed alternative if you don't want to operate it.

### How do I ship new code?
Push to the branch. The build service builds a new image; then restart `dagster-code` with `northflank restart service --projectId "$PROJECT_ID" --serviceId dagster-code`. Restart the webserver and daemon only when `dagster.yaml`, `workspace.yaml` or the Dagster version changes.

### Why one image for all three processes?
It keeps one build and one set of pinned versions. Dagster's example uses two images so the webserver and daemon don't carry your code's dependencies; split them the same way, with a second build service, once your project's dependencies get heavy or you deploy code far more often than Dagster itself.

### Can I run several code locations?
Yes. Deploy another code service from its own build on a different name and port, and add a `grpc_server` entry for it to `workspace.yaml`.

### How do I upgrade Dagster?
Bump all pinned versions together, push, wait for the build, run `dagster instance migrate` with `northflank exec service` on `dagster-webserver`, then restart all three services.

## Conclusion

You now have Dagster open source with a gRPC code location, webserver and daemon as separate services, run and event storage in managed PostgreSQL, and a UI behind basic auth.

**Next:** add a custom domain with [link a domain to a port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port), and move compute logs to object storage. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
- [How to deploy Apache Airflow](deploy-apache-airflow.md)
- [How to self-host Prefect](self-host-prefect.md)
