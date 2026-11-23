<!--
agent-guide-meta:
  id: NF-STK-CLICKHOUSE
  description: "How to deploy ClickHouse from the official clickhouse-server image with a password, a data volume, private HTTP and native ports, and a scripted query test."
  sources:
    - https://northflank.com/stacks/deploy-clickhouse
    - https://hub.docker.com/r/clickhouse/clickhouse-server
    - https://github.com/ClickHouse/ClickHouse/blob/master/docker/server/entrypoint.sh
    - https://github.com/ClickHouse/ClickHouse/blob/master/docker/server/Dockerfile.ubuntu
    - https://clickhouse.com/docs/interfaces/http
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/forwarding
  related: [NF-STK-KAFKA, NF-STK-ELASTIC, NF-DB-001, NF-DEP-002, NF-STK-MINIO]
-->

# How to deploy ClickHouse

To deploy ClickHouse, run the official `clickhouse/clickhouse-server` image with `CLICKHOUSE_USER`, `CLICKHOUSE_PASSWORD` and `CLICKHOUSE_DB` set, mount a persistent volume at `/var/lib/clickhouse`, and expose the HTTP interface on port 8123 and the native protocol on port 9000 only to your own network. Without a password, the image's `default` user has no network access at all, which is the usual reason a fresh container refuses remote connections.

This guide does it on Northflank with the CLI: a generated password in a secret group, a private service with HTTP and native ports and `/ping` health checks, a data volume, and a client job that creates a table, inserts rows and reads them back over `clickhouse:9000`. You end with a password-protected server your services reach privately. Every step is a copy-pasteable command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end.

## TL;DR: how to deploy ClickHouse

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project and service name, and pin a ClickHouse version, before changing anything.
3. Create the project, a secret group with a generated `CLICKHOUSE_PASSWORD`, and a `clickhouse/clickhouse-server:<version>` deployment service with private ports 8123 (HTTP) and 9000 (TCP), `CLICKHOUSE_DB` and `/ping` probes.
4. Attach a volume at `/var/lib/clickhouse` with `northflank create volume` and wait for `COMPLETED`.
5. Run a `clickhouse-client` test job and verify its logs show the row count, and that `/ping` returns `Ok.` through `northflank forward service`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

ClickHouse is a column-oriented analytics database. It needs a long-running server, fast persistent storage and enough memory for its queries. Serverless and static hosts don't run databases. ClickHouse Cloud is the managed option. A VM gives full control of kernel limits and disks, but you operate it. A container platform runs the official image with a volume and private networking, which suits product analytics, observability backends, and self-hosted tools that need ClickHouse next to them.

What the official image needs on a server:

- **A password.** The Docker Hub page says the `default` user has no network access unless `CLICKHOUSE_USER`, `CLICKHOUSE_PASSWORD` or `CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT` is set. Setting a password opens it to the network, protected by that password. The entrypoint rewrites this user from the environment on every start.
- **A data volume.** `/var/lib/clickhouse/` is the "main folder where ClickHouse stores the data". The entrypoint starts as root and hands the data directory to the `clickhouse` user (uid 101), so volume ownership is handled for you.
- **Two ports.** HTTP on 8123 (the HTTP interface, `/ping`, the `/play` UI) and the native protocol on 9000 (`clickhouse-client` and most drivers). Northflank's public ports carry only HTTP, and neither port should be public anyway, so both stay private.

ClickHouse isn't a Northflank addon type, so it runs as a deployment service. If you'd rather click through it, Northflank has a [ClickHouse stack template](https://northflank.com/stacks/deploy-clickhouse) that deploys a ClickHouse service and a persistent volume from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl` and `curl`**, to generate the password and run the checks.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Confirm where it will deploy

Before changing anything, print the deployment target. Names used below: project `data`, secret group `ch-env`, service `clickhouse`, volume `clickhouse-data`, job `ch-smoke`. If the apps that query ClickHouse already have a project, use its name instead of `data`, because private ports are project-scoped.

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId data -o json >/dev/null 2>&1 \
  && echo "project: data exists and will be reused" \
  || echo "project: data will be created"
northflank get service --projectId data --serviceId clickhouse -o json >/dev/null 2>&1 \
  && echo "service: clickhouse already exists: step 5 would update it" \
  || echo "service: clickhouse is free"
```

Then pin a version. Tags are `<branch>` (`26.8`) or a full release (`26.8.41.14`). `latest` moves between branches, so don't use it. This prints the newest full release on the 26.8 branch:

```bash
CH_VERSION=$(curl -s "https://hub.docker.com/v2/repositories/clickhouse/clickhouse-server/tags?page_size=100&name=26.8." \
  | jq -r '.results[].name' | grep -E '^26\.8\.[0-9]+\.[0-9]+$' | sort -V | tail -n1); echo "$CH_VERSION"
```

- If the service already exists, `put service` in step 5 would change it. Pick another name, or confirm that updating it is intended.
- Start with `nf-compute-200-4` (2 vCPU, 4 GB) and scale up for heavy queries. List plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | "\(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'`.

**If you're an agent:** show the user the team, the project (new or reused), the region (`europe-west` below, unless they choose another), the plan and the ClickHouse version. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "data", "region": "europe-west", "description": "Data services" }'
export PROJECT_ID=data
```

## Step 4: Create the password secret

```bash
CLICKHOUSE_PASSWORD=$(openssl rand -hex 24)
jq -n --arg p "$CLICKHOUSE_PASSWORD" '{
  name: "ch-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  secrets: { variables: { CLICKHOUSE_USER: "default", CLICKHOUSE_PASSWORD: $p } }
}' > ch-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file ch-env.json >/dev/null && rm ch-env.json
unset CLICKHOUSE_PASSWORD
```

The group is unrestricted, so the service and the test job both read it. Nothing prints the password; **tell the user they can read it from the secret group in the UI.**

## Step 5: Deploy ClickHouse

```bash
jq -n --arg v "$CH_VERSION" '{
  name: "clickhouse",
  billing: { deploymentPlan: "nf-compute-200-4" },
  deployment: { instances: 1, external: { imagePath: ("clickhouse/clickhouse-server:" + $v) }, docker: { configType: "default" } },
  ports: [
    { name: "http",   internalPort: 8123, public: false, protocol: "HTTP" },
    { name: "native", internalPort: 9000, public: false, protocol: "TCP" }
  ],
  runtimeEnvironment: { CLICKHOUSE_DB: "analytics", CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT: "1" },
  healthChecks: [
    { protocol: "HTTP", type: "readinessProbe", path: "/ping", port: 8123, initialDelaySeconds: 10, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 6, successThreshold: 1 },
    { protocol: "HTTP", type: "livenessProbe", path: "/ping", port: 8123, initialDelaySeconds: 60, periodSeconds: 30, timeoutSeconds: 5, failureThreshold: 3 }
  ]
}' > clickhouse.json

northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file clickhouse.json
```

REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/deployment`. Notes:

- **`CLICKHOUSE_DB`** creates the `analytics` database on first start. Tables can live in any database.
- **`CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT=1`** lets the `default` user create other users and grant roles with SQL, so you can give each app its own user instead of sharing `default`.
- **`/ping` probes.** The HTTP interface docs describe `/ping` as the endpoint for health checks; it returns `Ok.` without authentication.
- **Private addresses.** Services and jobs in the project reach `http://clickhouse:8123` and `clickhouse:9000`.
- **Kernel limits.** The Docker Hub examples raise `nofile` to 262144 and add `SYS_NICE`, `NET_ADMIN` and `IPC_LOCK` capabilities. The page calls the capabilities optional. This setup doesn't set them; ClickHouse logs warnings and runs.

## Step 6: Attach the data volume and wait

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId clickhouse-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "clickhouse-data",
  "mounts": [ { "containerMountPath": "/var/lib/clickhouse" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "clickhouse", "type": "service" } ]
}'

OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId clickhouse --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

`storageSize` is in MB and can only grow. A `ReadWriteOnce` volume limits the service to one instance, and on restart the old container stops before the new one starts. Attaching the volume redeploys the service.

## Step 7: Query it from a client job

The job runs `clickhouse-client` from the same image, with a custom entrypoint so the server doesn't start. ClickHouse string literals need single quotes, which the job command can't carry safely, so the test uses numbers only: it creates a table, inserts 1,000 generated rows and reads back the count and a sum. The single-quoted `SMOKE` keeps the variables unexpanded until the job runs:

```bash
SMOKE='set -e; C="clickhouse-client --host clickhouse --port 9000 --user $CLICKHOUSE_USER --password $CLICKHOUSE_PASSWORD"; $C --query "SELECT version()"; $C --query "CREATE TABLE IF NOT EXISTS analytics.smoke (id UInt32, n UInt64) ENGINE = MergeTree ORDER BY id"; $C --query "INSERT INTO analytics.smoke SELECT number, number * 2 FROM numbers(1000)"; $C --query "SELECT count(), sum(n) FROM analytics.smoke"'

jq -n --arg v "$CH_VERSION" --arg s "$SMOKE" '{
  name: "ch-smoke",
  billing: { deploymentPlan: "nf-compute-50" },
  settings: { backoffLimit: 0, activeDeadlineSeconds: 300, runOnSourceChange: "never" },
  deployment: {
    external: { imagePath: ("clickhouse/clickhouse-server:" + $v) },
    docker: { configType: "customEntrypointCustomCommand", customEntrypoint: "/bin/bash", customCommand: ("-c " + ($s | @sh)) }
  }
}' > ch-smoke.json
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file ch-smoke.json

RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId ch-smoke --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 40); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId ch-smoke --runId "$RUN_ID" --noDefaults -o json)
  echo "run=$(echo "$R" | jq -r '(.data // .).status')"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$(echo "$R" | jq -r '(.data // .).status')" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId ch-smoke --runId "$RUN_ID" --lineLimit 50
[ "$OK" = 1 ] || { echo "RUN_NOT_SUCCESSFUL: stop here and read the logs above"; false; }
```

`put job` makes a manual job because `settings.cron` is absent (`put job manual` is deprecated in the API). Each run inserts another 1,000 rows, so the count grows by 1,000 per run.

## How to know it worked

You're done when all of these hold:

1. The run concluded with `status` `SUCCESS`, and the logs show the server version, then `1000	999000` on the first run.
2. The HTTP interface answers through a private tunnel. Read the log for the local address rather than assuming one:
   ```bash
   northflank forward service --projectId "$PROJECT_ID" --serviceId clickhouse --skipHostnames > nf-clickhouse.log 2>&1 &
   FWD_PID=$!; sleep 10; cat nf-clickhouse.log
   curl -s http://<LOCAL_ADDR>:<LOCAL_HTTP_PORT>/ping                                  # Ok.
   echo 'SELECT count() FROM analytics.smoke' | curl -s -H 'X-ClickHouse-User: default' \
     -H "X-ClickHouse-Key: <PASSWORD>" http://<LOCAL_ADDR>:<LOCAL_HTTP_PORT>/ -d @-     # 1000
   kill $FWD_PID
   ```
   **This step needs a human** for the password: the user pastes it from the secret group. While the tunnel is up, `http://<LOCAL_ADDR>:<LOCAL_HTTP_PORT>/play` opens ClickHouse's built-in query UI.
3. No port is public:
   ```bash
   northflank get service --projectId "$PROJECT_ID" --serviceId clickhouse --noDefaults -o json \
     | jq -e '[(.data // .).ports[] | select(.public == true)] | length == 0'
   ```
4. Data survives a restart: `northflank restart service --projectId "$PROJECT_ID" --serviceId clickhouse --noDefaults -o json`, wait for `COMPLETED`, and the next job run starts from the old count.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Authentication failed` from the job | Job can't see the secret group, or password changed | Keep `ch-env` unrestricted; restart after changing it |
| `Connection refused` to `clickhouse:9000` | Native port missing or server still starting | Check `ports` with `get service`; wait for `COMPLETED` |
| `Memory limit (total) exceeded` | Query bigger than the plan | Use a bigger plan, or lower `max_memory_usage` per user |
| Server restarts under load | Container out of memory | Pick a plan with more RAM |
| Warnings about `nofile` or capabilities | Kernel limits not raised | Expected here: this guide runs without a raised `nofile` limit or the optional `SYS_NICE`, `NET_ADMIN` and `IPC_LOCK` capabilities, and ClickHouse only warns about it |
| Data gone after restart | Volume not attached at `/var/lib/clickhouse` | Check `attachedObjects` and the mount path |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 6 loop prints `ROLLOUT_NOT_COMPLETE`, or the step 7 job fails, read the logs (`northflank get service logs --projectId "$PROJECT_ID" --serviceId clickhouse`) before changing anything.
- Never print `CLICKHOUSE_PASSWORD`; never set `CLICKHOUSE_SKIP_USER_SETUP=1`, which the Docker Hub page calls insecure.
- Keep both ports private. For a tool that needs browser access, forward the service or put an authenticated app in front.
- Create a separate SQL user per app with `CREATE USER` and `GRANT` rather than sharing `default`.
- Get explicit confirmation before deleting the `clickhouse-data` volume or the service.

## FAQ

### Where should I host ClickHouse?
Next to the services that write to and query it, on a host with fast persistent storage and plenty of RAM. ClickHouse Cloud is the managed route. A VM gives full control for large clusters. A container platform, as here, runs the official image with a volume on a private network, which suits a single server for product analytics, logs or a self-hosted tool such as Langfuse or Sentry.

### Should apps use port 8123 or 9000?
Use whatever your driver expects. HTTP drivers such as `clickhouse-connect` (Python) and `@clickhouse/client` (Node.js) use `http://clickhouse:8123`. Native-protocol drivers and `clickhouse-client` use `clickhouse:9000`.

### Can I run a ClickHouse cluster?
Not by scaling this service: a `ReadWriteOnce` volume limits it to one instance. Replicated tables need ClickHouse Keeper and a service per replica with its own volume and `remote_servers` configuration. One well-sized server handles a lot of data, so scale up first.

### How do I upgrade?
Patch the tag within the same LTS branch first: `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId clickhouse --noDefaults -o json --input '{"deployment":{"instances":1,"external":{"imagePath":"clickhouse/clickhouse-server:<NEW_VERSION>"}}}'`. Read the changelog before moving branches, and back up the volume with [volume backups](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes).

## Conclusion

You now have ClickHouse on the official image, with a password-protected user, its data on a persistent volume, and HTTP and native ports that only your project can reach, checked by a client job and a tunnelled `/ping`.

**Next:** create a user per app, point your ingestion at `clickhouse:9000` or `http://clickhouse:8123`, and schedule volume backups. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy Kafka](deploy-kafka.md)
- [How to deploy Elasticsearch](deploy-elasticsearch.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
