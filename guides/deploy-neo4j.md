<!--
agent-guide-meta:
  id: NF-STK-NEO4J
  description: "How to deploy Neo4j from the official Docker image with NEO4J_AUTH, a /data volume, private Bolt and HTTP ports, memory settings and a cypher-shell test."
  sources:
    - https://neo4j.com/docs/operations-manual/current/docker/introduction/
    - https://neo4j.com/docs/operations-manual/current/docker/configuration/
    - https://github.com/neo4j/docker-neo4j/blob/master/docker-image-src/calver/coredb/docker-entrypoint.sh
    - https://hub.docker.com/_/neo4j
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
  related: [NF-DB-001, NF-STK-MONGODB, NF-STK-ELASTIC, NF-DEP-002, NF-STK-CLICKHOUSE]
-->

# How to deploy Neo4j

To deploy Neo4j, run the official `neo4j` Docker image with `NEO4J_AUTH=neo4j/<password>` set before the first start, mount a persistent volume at `/data`, keep the Bolt port 7687 and HTTP port 7474 on a private network, and size the heap and page cache to fit the container's memory. `NEO4J_AUTH` is only read when `/data` is empty, so changing it later doesn't change the password.

This guide does it on Northflank with the CLI: a generated password in a secret group, a Community Edition service with private Bolt and HTTP ports and health checks, a `/data` volume, and a client job that runs `cypher-shell` against `bolt://graphdb:7687`. You end with a password-protected graph database your services reach privately, verified by a write and a read. Every step is a copy-pasteable command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end.

## TL;DR: how to deploy Neo4j

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project and service name, and pin a Neo4j version, before changing anything.
3. Create the project, a secret group holding `NEO4J_AUTH`, and a `neo4j:<version>` deployment service with private ports 7687 (TCP) and 7474 (HTTP), heap and page cache settings, and health checks.
4. Attach a volume at `/data` with `northflank create volume` and wait for `COMPLETED`.
5. Run a `cypher-shell` test job and verify its logs show the node count, then open Neo4j Browser through `northflank forward service`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Neo4j is a JVM database with a persistent store. It needs an always-on container, a disk that survives restarts, enough memory for heap plus page cache, and a private network to the apps that query it. Serverless and static hosts don't run databases. Neo4j's managed service, AuraDB, runs it for you. A VM works if you'll handle the JVM, backups and upgrades. A container platform runs the official image with a volume and private networking, which suits a single graph database behind your app or a GraphRAG pipeline.

What the official image needs:

- **An initial password.** The Docker docs say to set `NEO4J_AUTH=neo4j/your_password`, that the username can't be changed from `neo4j`, and that the default minimum password length is 8 characters. If `/data` already holds a database, `NEO4J_AUTH` has no effect.
- **A `/data` volume.** That's where the docs mount persistent data. The entrypoint starts as root and hands mounted folders to the `neo4j` user, so volume ownership is handled.
- **Memory settings.** The defaults are a 512 MB heap and a 512 MB page cache. Settings are passed as `NEO4J_` variables, with `.` written as `_` and `_` written as `__`, so `server.memory.heap.max_size` becomes `NEO4J_server_memory_heap_max__size`.

**Licence.** Tags without a suffix (`neo4j:2026.10.2`) are Community Edition, which this guide uses. Enterprise Edition (`-enterprise` tags) requires `NEO4J_ACCEPT_LICENSE_AGREEMENT=yes`, and Neo4j's notice says using it without a commercial licence is prohibited (`eval` is accepted for evaluation). Check your licence before switching.

Neo4j isn't a Northflank addon type, so it runs as a deployment service.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`**, to generate the password.

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

Before changing anything, print the deployment target. Names used below: project `data`, secret group `neo4j-env`, service `graphdb`, volume `neo4j-data`, job `neo4j-smoke`. If the apps that query Neo4j already have a project, use its name instead of `data`, because private ports are project-scoped.

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId data -o json >/dev/null 2>&1 \
  && echo "project: data exists and will be reused" \
  || echo "project: data will be created"
northflank get service --projectId data --serviceId graphdb -o json >/dev/null 2>&1 \
  && echo "service: graphdb already exists: step 5 would update it" \
  || echo "service: graphdb is free"
```

Then pin a version. Neo4j's docs list calendar versions (current `2026.10`) and `5.26` as the LTS line. This prints the newest calendar tag; set `NEO4J_VERSION` to the one you pick:

```bash
curl -s "https://hub.docker.com/v2/repositories/library/neo4j/tags?page_size=100&ordering=last_updated" \
  | jq -r '.results[].name' | grep -E '^20[0-9]{2}\.[0-9]+\.[0-9]+$' | sort -V | tail -n1
NEO4J_VERSION=2026.10.2
```

- If the service already exists, `put service` in step 5 would change it. Pick another name, or confirm that updating it is intended.
- This guide uses `nf-compute-200-4` (2 vCPU, 4 GB) with a 1 GB heap and a 1 GB page cache. List plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | "\(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'`.

**If you're an agent:** show the user the team, the project (new or reused), the region (`europe-west` below, unless they choose another), the plan and the Neo4j version. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "data", "region": "europe-west", "description": "Data services" }'
export PROJECT_ID=data
```

## Step 4: Create the auth secret

```bash
GRAPH_PASSWORD=$(openssl rand -hex 20)
jq -n --arg p "$GRAPH_PASSWORD" '{
  name: "neo4j-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  secrets: { variables: { NEO4J_AUTH: ("neo4j/" + $p), GRAPH_PASSWORD: $p } }
}' > neo4j-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file neo4j-env.json >/dev/null && rm neo4j-env.json
unset GRAPH_PASSWORD
```

`NEO4J_AUTH` is for the server. `GRAPH_PASSWORD` is the same password for clients, under a name without the `NEO4J_` prefix, because the server turns every other `NEO4J_*` variable into a config setting. **Tell the user they can read the password from the secret group in the UI**; nothing here prints it.

## Step 5: Deploy Neo4j

```bash
jq -n --arg v "$NEO4J_VERSION" '{
  name: "graphdb",
  billing: { deploymentPlan: "nf-compute-200-4" },
  deployment: { instances: 1, external: { imagePath: ("neo4j:" + $v) }, docker: { configType: "default" } },
  ports: [
    { name: "bolt", internalPort: 7687, public: false, protocol: "TCP" },
    { name: "http", internalPort: 7474, public: false, protocol: "HTTP" }
  ],
  runtimeEnvironment: {
    NEO4J_server_memory_heap_initial__size: "1G",
    NEO4J_server_memory_heap_max__size: "1G",
    NEO4J_server_memory_pagecache_size: "1G"
  },
  healthChecks: [
    { protocol: "HTTP", type: "readinessProbe", path: "/", port: 7474, initialDelaySeconds: 20, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 12, successThreshold: 1 },
    { protocol: "TCP", type: "livenessProbe", port: 7687, initialDelaySeconds: 120, periodSeconds: 30, timeoutSeconds: 5, failureThreshold: 3 }
  ]
}' > graphdb.json

northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file graphdb.json
```

REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/deployment`. Notes:

- **Private ports.** Bolt is a binary protocol, so its port is `TCP`; Northflank's public ports only carry HTTP. Services and jobs in the project use `bolt://graphdb:7687`, and the HTTP API is at `http://graphdb:7474`.
- **Service name `graphdb`.** If the platform adds variables named after a service called `neo4j` (such as `NEO4J_PORT`), the entrypoint would turn them into config settings, which can stop the server. A different name avoids that.
- **Memory.** Heap plus page cache (2 GB) leaves the rest of the 4 GB plan for the JVM's other memory and the OS. On a bigger plan, raise the page cache first so more of the graph stays in memory.
- **Listen address.** The entrypoint already defaults `server.default_listen_address` to `0.0.0.0`.

## Step 6: Attach the `/data` volume and wait

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId neo4j-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "neo4j-data",
  "mounts": [ { "containerMountPath": "/data" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "graphdb", "type": "service" } ]
}'

OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId graphdb --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

Attaching redeploys the service onto the empty volume, so Neo4j sets the initial password from `NEO4J_AUTH` there. `storageSize` is in MB and only grows. A `ReadWriteOnce` volume limits the service to one instance.

## Step 7: Write and read with cypher-shell from a client job

The job runs `cypher-shell` from the same image with a custom entrypoint, so the server doesn't start. Cypher string literals would need quotes the job command can't carry safely, so the test uses numbers:

```bash
SMOKE='set -e; CS="/var/lib/neo4j/bin/cypher-shell -a bolt://graphdb:7687 -u neo4j -p $GRAPH_PASSWORD"; $CS "MERGE (n:Smoke {id: 1}) RETURN n.id AS id"; $CS "CALL dbms.components() YIELD name, versions, edition RETURN name, versions, edition"; $CS "MATCH (n:Smoke) RETURN count(n) AS smoke_nodes"'

jq -n --arg v "$NEO4J_VERSION" --arg s "$SMOKE" '{
  name: "neo4j-smoke",
  billing: { deploymentPlan: "nf-compute-50" },
  settings: { backoffLimit: 0, activeDeadlineSeconds: 300, runOnSourceChange: "never" },
  deployment: {
    external: { imagePath: ("neo4j:" + $v) },
    docker: { configType: "customEntrypointCustomCommand", customEntrypoint: "/bin/bash", customCommand: ("-c " + ($s | @sh)) }
  }
}' > neo4j-smoke.json
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file neo4j-smoke.json

RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId neo4j-smoke --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 40); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId neo4j-smoke --runId "$RUN_ID" --noDefaults -o json)
  echo "run=$(echo "$R" | jq -r '(.data // .).status')"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$(echo "$R" | jq -r '(.data // .).status')" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId neo4j-smoke --runId "$RUN_ID" --lineLimit 50
[ "$OK" = 1 ] || { echo "RUN_NOT_SUCCESSFUL: stop here and read the logs above"; false; }
```

`put job` makes a manual job because `settings.cron` is absent (`put job manual` is deprecated in the API). `MERGE` makes the write idempotent, so re-runs keep one `Smoke` node.

## How to know it worked

You're done when all of these hold:

1. The run concluded with `status` `SUCCESS`, and the logs show `id` `1`, an edition of `community`, and `smoke_nodes` `1`.
2. No port is public:
   ```bash
   northflank get service --projectId "$PROJECT_ID" --serviceId graphdb --noDefaults -o json \
     | jq -e '[(.data // .).ports[] | select(.public == true)] | length == 0'
   ```
3. Neo4j Browser works through a tunnel. Forward the service and read the log for the local ports of both 7474 and 7687:
   ```bash
   northflank forward service --projectId "$PROJECT_ID" --serviceId graphdb --skipHostnames > nf-neo4j.log 2>&1 &
   FWD_PID=$!; sleep 10; cat nf-neo4j.log
   ```
   **This step needs a human:** open `http://<LOCAL_ADDR>:<LOCAL_HTTP_PORT>/`, connect to `bolt://<LOCAL_ADDR>:<LOCAL_BOLT_PORT>` as `neo4j` with the password from the secret group, and run `MATCH (n:Smoke) RETURN n`. Then `kill $FWD_PID`.
4. Data survives a restart: `northflank restart service --projectId "$PROJECT_ID" --serviceId graphdb --noDefaults -o json`, wait for `COMPLETED`, run the job again, and `smoke_nodes` is still `1`.

Apps in the project use `bolt://graphdb:7687` with user `neo4j` and `GRAPH_PASSWORD`. Prefer `bolt://` over `neo4j://` here: the routing scheme makes drivers reconnect to the server's advertised address, which this setup doesn't set.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Logs: `Invalid value for password. The minimum password length is 8 characters.` | Password too short | Use the generated 40-character password |
| Job logs an authentication failure | `/data` was initialised with another password | `NEO4J_AUTH` only applies to an empty `/data`; put the existing password in the secret group, or change it in Cypher with `ALTER CURRENT USER SET PASSWORD FROM ... TO ...` |
| Server exits at start citing an unrecognised setting | A stray `NEO4J_*` variable | Remove it from the environment and the secret group |
| Enterprise image exits at once | Licence not accepted | Set `NEO4J_ACCEPT_LICENSE_AGREEMENT` only if your licence allows |
| `OutOfMemoryError` or container killed | Heap plus page cache too large for the plan | Lower them or pick a bigger plan |
| Data gone after restart | Volume not attached at `/data` | Check `attachedObjects` and the mount path |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 6 loop prints `ROLLOUT_NOT_COMPLETE`, or the step 7 job prints `RUN_NOT_SUCCESSFUL`, read the logs (`northflank get service logs --projectId "$PROJECT_ID" --serviceId graphdb`) before changing anything.
- Never print `NEO4J_AUTH` or `GRAPH_PASSWORD`.
- Don't switch to an `-enterprise` tag or set `NEO4J_ACCEPT_LICENSE_AGREEMENT` without the user confirming they hold a licence.
- Keep both ports private. For browser access, use `northflank forward service`.
- Get explicit confirmation before deleting the `neo4j-data` volume or the service.

## FAQ

### Where should I host Neo4j?
On a host with a persistent disk, enough memory for heap plus page cache, and a private network to your app. AuraDB is the managed option. A VM gives you full control for clusters. A container platform, as in this guide, runs the official image with a volume and private Bolt and HTTP ports, which suits a single Community Edition database.

### Is Neo4j free to self-host?
Community Edition, the default image tag, is free and open source. Enterprise Edition features, including clustering, need a commercial licence, which the image enforces with `NEO4J_ACCEPT_LICENSE_AGREEMENT`.

### Can I run a Neo4j cluster?
Clustering is an Enterprise Edition feature. It needs one service per server, each with its own volume and advertised addresses, plus a licence. This guide covers a single server.

### How do I back up and upgrade?
Back up with [volume backups](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes) or Neo4j's own dump tools. To upgrade, read Neo4j's upgrade notes, then patch the tag: `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId graphdb --noDefaults -o json --input '{"deployment":{"instances":1,"external":{"imagePath":"neo4j:<NEW_VERSION>"}}}'`.

## Conclusion

You now have Neo4j Community Edition on the official image, with a password set at first start, the graph on a persistent `/data` volume, and Bolt and HTTP ports that only your project can reach, verified by `cypher-shell`.

**Next:** connect your app with an official Neo4j driver at `bolt://graphdb:7687`, schedule volume backups, and raise the page cache as the graph grows. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy MongoDB](deploy-mongodb.md)
- [How to deploy Elasticsearch](deploy-elasticsearch.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy ClickHouse](deploy-clickhouse.md)
