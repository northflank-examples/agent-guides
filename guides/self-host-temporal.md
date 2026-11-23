<!--
agent-guide-meta:
  id: NF-STK-TEMPORAL
  description: "How to self-host Temporal in production with the temporalio/server image, PostgreSQL persistence, schema setup as a job, private gRPC on 7233 and the UI."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/stacks/deploy-temporal
    - https://github.com/temporalio/temporal/blob/v1.32.2/common/config/config_template_embedded.yaml
    - https://github.com/temporalio/temporal/blob/v1.32.2/docker/scripts/sh/entrypoint.sh
    - https://github.com/temporalio/temporal/blob/v1.32.2/docker/targets/admin-tools.Dockerfile
    - https://github.com/temporalio/temporal/blob/v1.32.2/tools/sql/main.go
    - https://github.com/temporalio/samples-server/blob/main/compose/docker-compose-postgres.yml
    - https://github.com/temporalio/samples-server/blob/main/compose/scripts/setup-postgres.sh
    - https://github.com/temporalio/temporal/blob/v1.32.2/common/dynamicconfig/file_based_client.go
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://github.com/temporalio/docker-builds/blob/main/docker/auto-setup.sh
    - https://hub.docker.com/r/temporalio/auto-setup
    - https://docs.temporal.io/cli/operator
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/project/jobs/run-job
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
    - https://northflank.com/docs/v1/application/network/networking-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
  related: [NF-DB-001, NF-DEP-001, NF-DEP-002, NF-STK-AIRFLOW]
-->

# How to self-host Temporal

To self-host Temporal in production, run the `temporalio/server` image against PostgreSQL with two databases (`temporal` and `temporal_visibility`), create their schemas with `temporal-sql-tool` from the `temporalio/admin-tools` image before the server starts, keep the gRPC frontend on port 7233 private to your workers, and put the Temporal UI behind authentication. The old `auto-setup` image is deprecated, so schema setup is now your job.

This guide does it on Northflank with the CLI: a managed PostgreSQL addon, a schema job and a namespace job from `admin-tools`, the server built from a two-file repo, private gRPC at `temporal:7233`, and the UI on a public HTTPS URL with basic auth. You end with a healthy cluster and a `default` namespace. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human links Git once.

## TL;DR: how to self-host Temporal

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check the repo, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Create PostgreSQL with `northflank put addon`, a secret group that links its host, port, user and password as the server's `POSTGRES_*` and the tool's `SQL_*` variables, and run `temporal-sql-tool` as a schema job.
4. Push a `Dockerfile` (`FROM temporalio/server`) with a dynamic config file and deploy it with `northflank put service combined` on private TCP port 7233.
5. Create the `default` namespace with a second job, and deploy `temporalio/ui` behind basic auth.
6. Check that the live build is your pushed commit, `cluster health` reports `SERVING` and the UI answers on its public URL.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

A self-hosted Temporal Service has four pieces:

- **Persistence.** PostgreSQL holds two databases: `temporal` for workflow state and `temporal_visibility` for search. Each has its own versioned schema.
- **Schema tooling.** `temporal-sql-tool` ships in `temporalio/admin-tools`, together with the schema files under `/etc/temporal/schema`. The server image doesn't create tables.
- **The server.** `temporalio/server` runs the frontend, history, matching and worker services in one process. It builds its config from environment variables (`DB`, `POSTGRES_SEEDS`, `POSTGRES_USER`…) and refuses to start without a dynamic config file at `/etc/temporal/config/dynamicconfig/docker.yaml`, which the image doesn't include.
- **The UI**, a separate `temporalio/ui` image that talks to the frontend over gRPC. It has no login of its own.

Your workers and clients connect to the frontend on port 7233. Keep it private: in this guide, workers in the same project reach it at `temporal:7233`.

Where to self-host Temporal comes down to long-running processes, a reliable SQL database and private networking between server and workers. Serverless platforms can't hold gRPC connections open. Kubernetes with the Temporal Helm chart works if you run the cluster; a container platform with managed Postgres and private networking runs the same images with less to operate.

If you'd rather click through it, Northflank has a [Temporal stack template](https://northflank.com/stacks/deploy-temporal) that deploys the server, UI, admin tools and PostgreSQL from the UI.

## Prerequisites

- **Optional: Docker, to check the server image builds locally** after step 8 adds its files:
  ```bash
  docker build -t temporal-local . && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`** for generating the UI password. Optionally, the [Temporal CLI](https://docs.temporal.io/cli) for the local check.

Pin matching versions: server and admin-tools `1.32.2`, UI `2.58.1` at the time of writing. Names used below: project `temporal`, addon `temporal-db`, secret group `temporal-db-env`, jobs `temporal-schema` and `temporal-namespace`, services `temporal` and `temporal-ui`.

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
ls Dockerfile dynamicconfig.yaml 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId temporal -o json >/dev/null 2>&1 \
  && echo "project: temporal exists and will be reused" \
  || echo "project: temporal will be created"
```

- This guide builds only the Temporal server from Git: a two-file repo holding a `Dockerfile` and `dynamicconfig.yaml`. If the repo already has a `Dockerfile` that isn't `FROM temporalio/server`, it belongs to another app. Stop and use a separate repo (or folder) for the server, or use a guide for your framework.
- If `dynamicconfig.yaml` already exists, keep its settings and make sure step 8's `COPY` line points at it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services, jobs and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "temporal", "region": "europe-west" }'
export PROJECT_ID=temporal

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

The schema and namespace jobs and the UI use public images, so they need no access check.

## Step 5: Create PostgreSQL

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "temporal-db", "type": "postgresql", "version": "16-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-100-2", "storage": 16384, "replicas": 1 },
  "customCredentials": { "dbName": "temporal" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId temporal-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

Temporal's server config keeps `SQL_TLS_ENABLED=false` by default, which matches `tlsEnabled: false` on a private database.

## Step 6: Create the database secret group

Print the credential **key names** first, never the values:

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId temporal-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

If the output casing differs from `HOST`, `PORT`, `USERNAME`, `PASSWORD`, change the `keyName` values below. Each key gets two aliases: one for the server's config template and one for `temporal-sql-tool`, which reads `SQL_*` variables.

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "temporal-db-env", "type": "secret", "secretType": "environment", "priority": 10,
  "restrictions": { "restricted": true, "nfObjects": [
    { "id": "temporal", "type": "service" }, { "id": "temporal-schema", "type": "job" } ] },
  "addonDependencies": [ { "addonId": "temporal-db", "keys": [
    { "keyName": "HOST",     "aliases": ["POSTGRES_SEEDS", "SQL_HOST"] },
    { "keyName": "PORT",     "aliases": ["DB_PORT", "SQL_PORT"] },
    { "keyName": "USERNAME", "aliases": ["POSTGRES_USER", "SQL_USER"] },
    { "keyName": "PASSWORD", "aliases": ["POSTGRES_PWD", "SQL_PASSWORD"] } ] } ],
  "secrets": { "variables": {
    "DB": "postgres12", "SQL_PLUGIN": "postgres12",
    "DBNAME": "temporal", "VISIBILITY_DBNAME": "temporal_visibility" } }
}'
```

Only the server and the schema job can read the group; the UI never sees database credentials.

## Step 7: Create the schemas

The job runs `temporal-sql-tool` from the admin-tools image: initial setup and update to the latest version for `temporal`, then the same for `temporal_visibility` after creating it. The image has no entrypoint, so the job uses `sh -c`:

```bash
SCHEMA='S=/etc/temporal/schema/postgresql/v12; temporal-sql-tool --db temporal setup-schema -v 0.0 && temporal-sql-tool --db temporal update-schema -d $S/temporal/versioned && (temporal-sql-tool --db temporal_visibility create-database --defaultdb temporal || echo visibility database exists) && temporal-sql-tool --db temporal_visibility setup-schema -v 0.0 && temporal-sql-tool --db temporal_visibility update-schema -d $S/visibility/versioned'
jq -n --arg s "$SCHEMA" '{
  name: "temporal-schema",
  billing: { deploymentPlan: "nf-compute-20" },
  settings: { backoffLimit: 0, activeDeadlineSeconds: 900, runOnSourceChange: "never" },
  deployment: {
    external: { imagePath: "temporalio/admin-tools:1.32.2" },
    docker: { configType: "customEntrypointCustomCommand", customEntrypoint: "sh -c", customCommand: ("\"" + $s + "\"") } }
}' > temporal-schema.json
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file temporal-schema.json

RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId temporal-schema --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 60); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId temporal-schema --runId "$RUN_ID" --noDefaults -o json)
  RS=$(echo "$R" | jq -r '(.data // .).status'); echo "run=$RS"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$RS" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId temporal-schema --runId "$RUN_ID" --lineLimit 200
[ "$OK" = 1 ] || { echo "RUN_NOT_SUCCESSFUL: stop here and read the job logs above"; false; }
```

The run must end `SUCCESS`. Re-run this job with the new admin-tools tag before every server upgrade, because `update-schema` applies any new versions.

## Step 8: Deploy the server

Add two files to the repository. `Dockerfile`:

```dockerfile
FROM temporalio/server:1.32.2
COPY dynamicconfig.yaml /etc/temporal/config/dynamicconfig/docker.yaml
```

And `dynamicconfig.yaml`, from Temporal's own Compose setup:

```yaml
limit.maxIDLength:
  - value: 255
    constraints: {}
```

Northflank builds what's on the remote branch, not your local working tree, so commit and push the files first:

```bash
git add Dockerfile dynamicconfig.yaml
git commit -m "Add Northflank deployment files"
git push
```

Write `temporal.json`. `NUM_HISTORY_SHARDS` **can't be changed after the cluster first starts**, so pick it now; 512 is a common production starting point, and the image default is 4.

```json
{
  "name": "temporal",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "grpc", "internalPort": 7233, "public": false, "protocol": "TCP" } ],
  "runtimeEnvironment": { "BIND_ON_IP": "0.0.0.0", "NUM_HISTORY_SHARDS": "512", "LOG_LEVEL": "info" },
  "healthChecks": [
    { "protocol": "TCP", "type": "readinessProbe", "port": 7233,
      "initialDelaySeconds": 15, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 6, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file temporal.json
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId temporal --noDefaults -o json)
  B=$(echo "$S" | jq -r '(.data // .).status.build.status // "NONE"'); D=$(echo "$S" | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "build=$B deployment=$D"
  case "$B" in FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo BUILD_FAILED; break;; esac
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  [ "$B" = "SUCCESS" ] && [ "$D" = "COMPLETED" ] && { OK=1; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
```

Use the same repository URL, provider and branch as in step 4. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId temporal --lineLimit 200`.

With `BIND_ON_IP=0.0.0.0`, the image's entrypoint advertises the pod IP to the cluster's membership ring and listens on all interfaces, so `temporal:7233` reaches the frontend. If no build appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId temporal --noDefaults -o json --input '{}'`.

## Step 9: Create the `default` namespace

```bash
NS='temporal operator cluster health --address temporal:7233 && (temporal operator namespace describe --namespace default --address temporal:7233 || temporal operator namespace create --namespace default --retention 72h --address temporal:7233)'
jq -n --arg s "$NS" '{
  name: "temporal-namespace",
  billing: { deploymentPlan: "nf-compute-20" },
  settings: { backoffLimit: 0, activeDeadlineSeconds: 300, runOnSourceChange: "never" },
  deployment: {
    external: { imagePath: "temporalio/admin-tools:1.32.2" },
    docker: { configType: "customEntrypointCustomCommand", customEntrypoint: "sh -c", customCommand: ("\"" + $s + "\"") } }
}' > temporal-namespace.json
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file temporal-namespace.json
RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId temporal-namespace --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
```

Poll it as in step 7. Keep the job: run it again any time as a health check, or copy it for more namespaces.

## Step 10: Deploy the UI with basic auth

The UI has no login, and it can start, cancel and terminate workflows, so the public port gets Northflank's basic-auth security rule:

```bash
UI_PW=$(openssl rand -hex 16)
jq -n --arg pw "$UI_PW" '{
  name: "temporal-ui",
  billing: { deploymentPlan: "nf-compute-20" },
  deployment: { instances: 1, external: { imagePath: "temporalio/ui:2.58.1" }, docker: { configType: "default" } },
  ports: [ { name: "http", internalPort: 8080, public: true, protocol: "HTTP",
             security: { credentials: [ { type: "basic-auth", username: "admin", password: $pw } ] } } ],
  runtimeEnvironment: { TEMPORAL_ADDRESS: "temporal:7233" },
  healthChecks: [ { protocol: "TCP", type: "readinessProbe", port: 8080, initialDelaySeconds: 5,
                    periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 3, successThreshold: 1 } ]
}' > temporal-ui.json
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file temporal-ui.json >/dev/null && rm temporal-ui.json
```

**Hand `UI_PW` to the user through a password manager**, and `unset UI_PW` after the checks below. For SSO or IP allowlists instead, see [add security policies for ports](https://northflank.com/docs/v1/application/network/add-security-policies-for-ports).

## How to know it worked

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId temporal --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The `temporal-schema` run concluded `SUCCESS`, and the `temporal-namespace` logs show `SERVING` from `cluster health`.
3. The UI is up and protected. Once `northflank get service --projectId "$PROJECT_ID" --serviceId temporal-ui` shows deployment `COMPLETED`:
   ```bash
   HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId temporal-ui --noDefaults -o json \
     | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
   # Older API responses only list the host on the deprecated ports endpoint
   [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId temporal-ui --noDefaults -o json \
     | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"                       # 401 without credentials
   curl -s -o /dev/null -w '%{http_code}\n' -u "admin:$UI_PW" "https://$HOST/"     # 200
   ```
4. From your machine, forward the private port and query it with the Temporal CLI:
   ```bash
   northflank forward service --projectId "$PROJECT_ID" --serviceId temporal &
   temporal operator namespace describe --namespace default --address temporal:7233
   ```
   `forward` maps the service's hostname locally unless you pass `--skipHostnames`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| Server exits with `unable to create dynamic config client: … dynamic config file: … no such file or directory` | Image built without `dynamicconfig.yaml` | Check the `COPY` path; rebuild |
| `sql schema version compatibility check failed` | Schema job not run, or older than the server | Re-run step 7 with matching admin-tools tag |
| `permission denied to create database` | Addon user can't create databases | Create a second addon with `dbName: temporal_visibility`, link it as `VISIBILITY_POSTGRES_SEEDS`, `VISIBILITY_POSTGRES_USER`, `VISIBILITY_POSTGRES_PWD`, `VISIBILITY_DB_PORT` |
| Workers get `connection refused` on `temporal:7233` | Worker in another project, or server not ready | Deploy workers in this project; check readiness |
| UI loads but shows `Unavailable` | Wrong `TEMPORAL_ADDRESS` | Use `temporal:7233`; restart the UI |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the server until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print database passwords or the UI password. Print credential key names only.
- Never make port 7233 public without mTLS and an authorizer configured; anyone who can reach it can run workflows.
- Treat `NUM_HISTORY_SHARDS` as permanent. Ask before changing it, and never change it on a cluster with data.
- Back up the database before a server upgrade, and don't delete the addon or services without explicit, per-resource confirmation.

## FAQ

### Where should I self-host Temporal?
On infrastructure with a managed SQL database and private networking between the server and your workers, so port 7233 never faces the internet. A container platform with managed Postgres fits; so does Kubernetes with the Helm chart if you already run a cluster.

### Is self-hosted Temporal free?
The server is MIT-licensed open source. You pay for the compute and database it runs on, and you operate upgrades, schema migrations and backups yourself.

### How do my workers connect?
Deploy each worker as a service in the same project with its SDK client pointed at `temporal:7233` and namespace `default`. Workers need no public port.

### How do I upgrade Temporal?
Back up the database, re-run the schema job with the new admin-tools tag, then bump the `FROM` tag in your `Dockerfile` and push. Upgrade one minor version at a time, as Temporal's release notes advise.

## Conclusion

You now have a self-hosted Temporal Service on PostgreSQL with versioned schemas, a private gRPC frontend at `temporal:7233`, a `default` namespace and a UI behind basic auth.

**Next:** deploy your workers next to the server, and [schedule database backups](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy Apache Airflow](deploy-apache-airflow.md)
