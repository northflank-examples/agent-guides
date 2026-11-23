<!--
agent-guide-meta:
  id: NF-STK-ELASTIC
  description: "How to deploy Elasticsearch as a secured single node from the official image, with a set password, a data volume, private networking and a scripted index test."
  sources:
    - https://northflank.com/stacks/deploy-elasticsearch
    - https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-docker-basic
    - https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-docker-prod
    - https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-docker-configure
    - https://www.elastic.co/docs/deploy-manage/deploy/self-managed/bootstrap-checks
    - https://github.com/elastic/elasticsearch/blob/main/distribution/docker/src/docker/bin/docker-entrypoint.sh
    - https://hub.docker.com/r/curlimages/curl
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
  related: [NF-STK-KAFKA, NF-STK-CLICKHOUSE, NF-DB-001, NF-DEP-002, NF-STK-MONGODB]
-->

# How to deploy Elasticsearch

To deploy Elasticsearch, run the official `docker.elastic.co/elasticsearch/elasticsearch` image with `discovery.type=single-node`, keep security on and set the `elastic` user's password through `ELASTIC_PASSWORD`, mount a persistent volume at `/usr/share/elasticsearch/data`, and give the container enough memory for its automatically sized heap. Most failed installs come from too little memory or a host-level `vm.max_map_count` check.

This guide does it on Northflank with the CLI: a generated `elastic` password in a secret group, a private service on port 9200 with TCP health checks, a data volume, and a client job that indexes and searches a document at `http://elasticsearch:9200`. You end with a secured node that rejects anonymous requests, verified from the job logs. Every step is a copy-pasteable command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end.

## TL;DR: how to deploy Elasticsearch

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project, service name and Elasticsearch version before changing anything.
3. Create the project, a secret group holding a generated `ELASTIC_PASSWORD`, and a `docker.elastic.co/elasticsearch/elasticsearch:<version>` deployment service: single-node discovery, security on, a private port 9200 and TCP probes, on a 4 GB plan.
4. Attach a volume at `/usr/share/elasticsearch/data` with `northflank create volume` and wait for `COMPLETED`.
5. Run a `curl` client job and verify its logs show cluster health, one search hit and `anonymous=401`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Elasticsearch is a memory-hungry JVM with a persistent index on disk. It needs an always-on container, a volume, several gigabytes of RAM and a private network path from the apps that query it. Serverless and static hosts can't run it. A VM gives you full control of kernel settings like `vm.max_map_count`, but you own the OS, upgrades and backups. Elastic Cloud runs it for you. A container platform runs the official image with a volume and private networking, within the limits of a shared kernel.

The choices in this guide, and why:

- **Single-node discovery.** `discovery.type=single-node` stops the node looking for peers. Elastic's bootstrap checks page says a node using single-node discovery is in development mode, where failed bootstrap checks are logged as warnings instead of stopping startup, which matters here because `vm.max_map_count` is a host kernel setting you can't normally change from a container.
- **Security on, with a known password.** The image sets the `elastic` user's bootstrap password from `ELASTIC_PASSWORD`. Setting `xpack.security.enabled` explicitly skips Elastic's first-start security auto-configuration, so no password is generated or printed: without `ELASTIC_PASSWORD` you'd have to reset it before you can log in.
- **HTTP TLS off, network private.** The port is reachable only inside the project, so this guide turns off HTTP TLS to keep clients simple and protects the API with the password. Turn TLS on if your security policy requires encryption inside the project (see the FAQ).
- **Memory.** Elastic recommends automatic heap sizing, which reads the container's memory. The plan's memory *is* the setting.

Settings are passed in Elastic's `ES_SETTING_` style (uppercase, `.` becomes `_`, `_` becomes `__`), which the Docker docs offer for platforms that don't allow dots in variable names.

If you'd rather click through it, Northflank has an [Elasticsearch with Kibana stack template](https://northflank.com/stacks/deploy-elasticsearch) that deploys Elasticsearch with a persistent volume and Kibana from the UI.

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

Before changing anything, print the deployment target. Names used below: project `data`, secret group `es-env`, service `elasticsearch`, volume `es-data`, job `es-smoke`. If the apps that query Elasticsearch already have a project, use its name instead of `data`, because private ports are project-scoped.

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId data -o json >/dev/null 2>&1 \
  && echo "project: data exists and will be reused" \
  || echo "project: data will be created"
northflank get service --projectId data --serviceId elasticsearch -o json >/dev/null 2>&1 \
  && echo "service: elasticsearch already exists: step 5 would update it" \
  || echo "service: elasticsearch is free"
ES_VERSION=9.5.6
```

- If the service already exists, `put service` in step 5 would change it. Pick another name, or confirm that updating it is intended.
- The examples use `9.5.6`, the latest release in Elastic's Docker docs on 2026-11-20. Check the [Docker install page](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-docker-basic) for a newer one and set `ES_VERSION` to it.
- Elastic's quick start uses a 1 GB container, which is a lab minimum. Use `nf-compute-200-4` (2 vCPU, 4 GB) or larger. List plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | "\(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'`.

**If you're an agent:** show the user the team, the project (new or reused), the region (`europe-west` below, unless they choose another), the plan and the Elasticsearch version. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "data", "region": "europe-west", "description": "Data services" }'
export PROJECT_ID=data
```

## Step 4: Create the password secret

```bash
ELASTIC_PASSWORD=$(openssl rand -hex 24)
jq -n --arg p "$ELASTIC_PASSWORD" '{
  name: "es-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  secrets: { variables: { ELASTIC_PASSWORD: $p } }
}' > es-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file es-env.json >/dev/null && rm es-env.json
unset ELASTIC_PASSWORD
```

The password is hex, so it's safe in shell scripts and URLs. The group is unrestricted, so both the service and the client job read it. **Tell the user they can read it from the secret group in the UI**; nothing here prints it.

## Step 5: Deploy Elasticsearch

```bash
jq -n --arg v "$ES_VERSION" '{
  name: "elasticsearch",
  billing: { deploymentPlan: "nf-compute-200-4" },
  deployment: { instances: 1, external: { imagePath: ("docker.elastic.co/elasticsearch/elasticsearch:" + $v) }, docker: { configType: "default" } },
  ports: [ { name: "http", internalPort: 9200, public: false, protocol: "HTTP" } ],
  runtimeEnvironment: {
    ES_SETTING_DISCOVERY_TYPE: "single-node",
    ES_SETTING_CLUSTER_NAME: "search",
    ES_SETTING_XPACK_SECURITY_ENABLED: "true",
    ES_SETTING_XPACK_SECURITY_HTTP_SSL_ENABLED: "false"
  },
  healthChecks: [
    { protocol: "TCP", type: "readinessProbe", port: 9200, initialDelaySeconds: 30, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 18, successThreshold: 1 },
    { protocol: "TCP", type: "livenessProbe", port: 9200, initialDelaySeconds: 180, periodSeconds: 30, timeoutSeconds: 5, failureThreshold: 4 }
  ]
}' > elasticsearch.json

northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file elasticsearch.json
```

REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/deployment`. Notes:

- **Private port.** `public: false` means only workloads in the project reach `http://elasticsearch:9200`. Port 9300 (transport) isn't needed for a single node.
- **TCP probes.** With security on, an unauthenticated `GET /` returns 401, which an HTTP probe treats as failure. A TCP probe passes once the REST layer listens. Startup takes a minute or more, hence the long readiness window.
- **Heap vs plan memory.** Elastic's production docs say the node "automatically sizes JVM heap based on" its roles and "the total memory available to the node's container", and recommend that default. Pick the plan for the memory you need. The rest of the memory serves the filesystem cache, which Lucene relies on. Elastic says `ES_JAVA_OPTS` (for example `-Xms2g -Xmx2g`) "overrides all other JVM options" and isn't recommended in production; use it only if logs show a heap far from what the plan allows.
- **`vm.max_map_count`.** Elastic's production page says it "must be set to `1048576`" on the host. It's a kernel setting, and single-node discovery makes the check a warning rather than a startup failure. If the log warns about it, that's expected. If you ever see mmap errors under load, disabling mmap with `ES_SETTING_NODE_STORE_ALLOW__MMAP=false` is the documented way around the check, at some cost to performance.

## Step 6: Attach the data volume and wait

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId es-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "es-data",
  "mounts": [ { "containerMountPath": "/usr/share/elasticsearch/data" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "elasticsearch", "type": "service" } ]
}'

OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId elasticsearch --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

Elastic says to bind a volume to `/usr/share/elasticsearch/data` so data survives restarts. `storageSize` is in MB and can only grow. Elasticsearch stops writing to indices when the disk is nearly full (its disk watermarks), so leave headroom. A `ReadWriteOnce` volume limits the service to one instance, and attaching it redeploys the service.

## Step 7: Index and search from a client job

The job uses the small `curlimages/curl` image (pin a tag from [Docker Hub](https://hub.docker.com/r/curlimages/curl); `8.23.0` below) and reads `ELASTIC_PASSWORD` from the secret group. The single-quoted `SMOKE` keeps `$ELASTIC_PASSWORD` unexpanded until it runs in the job:

```bash
SMOKE='set -e; U="elastic:$ELASTIC_PASSWORD"; ES=http://elasticsearch:9200;
curl -sf -u "$U" "$ES/_cluster/health?wait_for_status=yellow&timeout=60s"; echo;
curl -sf -u "$U" -X PUT "$ES/smoke/_doc/1?refresh=true" -H "Content-Type: application/json" -d "{\"msg\":\"hello\"}"; echo;
curl -sf -u "$U" "$ES/smoke/_search?q=msg:hello"; echo;
curl -s -o /dev/null -w "anonymous=%{http_code}\n" "$ES/"'

jq -n --arg s "$SMOKE" '{
  name: "es-smoke",
  billing: { deploymentPlan: "nf-compute-10" },
  settings: { backoffLimit: 0, activeDeadlineSeconds: 300, runOnSourceChange: "never" },
  deployment: {
    external: { imagePath: "curlimages/curl:8.23.0" },
    docker: { configType: "customEntrypointCustomCommand", customEntrypoint: "/bin/sh", customCommand: ("-c " + ($s | gsub("\n"; " ") | @sh)) }
  }
}' > es-smoke.json
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file es-smoke.json

RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId es-smoke --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 40); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId es-smoke --runId "$RUN_ID" --noDefaults -o json)
  echo "run=$(echo "$R" | jq -r '(.data // .).status')"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$(echo "$R" | jq -r '(.data // .).status')" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId es-smoke --runId "$RUN_ID" --lineLimit 50
[ "$OK" = 1 ] || { echo "RUN_NOT_SUCCESSFUL: stop here and read the logs above"; false; }
```

`put job` makes a manual job because `settings.cron` is absent (`put job manual` is deprecated in the API). With `set -e` and `curl -f`, any 4xx or 5xx fails the run. The `gsub` joins the script onto one line, since each command already ends with `;`.

## How to know it worked

You're done when all of these hold:

1. The run concluded with `status` `SUCCESS`.
2. The logs show `"status":"yellow"` or `"green"` (a new index has a replica that can't be placed on one node, so yellow is normal), a search response with `"total":{"value":1`, and `anonymous=401`, which proves security is on.
3. The service has no public port:
   ```bash
   northflank get service --projectId "$PROJECT_ID" --serviceId elasticsearch --noDefaults -o json \
     | jq -e '[(.data // .).ports[] | select(.public == true)] | length == 0'
   ```
4. Data survives a restart: `northflank restart service --projectId "$PROJECT_ID" --serviceId elasticsearch --noDefaults -o json`, wait for `COMPLETED`, run the job again, and the search still finds the document.

Apps in the project connect to `http://elasticsearch:9200` with user `elastic` and `ELASTIC_PASSWORD` from the secret group. For apps, create a dedicated user or API key instead of sharing `elastic`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Container killed, `OutOfMemoryError`, or restart loop | Plan too small for heap plus page cache | Use `nf-compute-200-4` or larger |
| `AccessDeniedException` on `/usr/share/elasticsearch/data` | Volume not writable by uid 1000 / gid 0 | Report it; see [volume permissions](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) |
| Startup fails with `bootstrap checks failed` and `max virtual memory areas vm.max_map_count` | Single-node discovery not applied | Check `ES_SETTING_DISCOVERY_TYPE=single-node` with `get service` |
| Job gets `401` on every call | Wrong or missing `ELASTIC_PASSWORD`, or data volume created with an older password | The bootstrap password only applies to a new node. Reset it with `/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic` (Elastic's Docker docs) via `exec`, then update the secret group |
| Clients get an SSL error | Client uses `https://` | Use `http://elasticsearch:9200` or enable HTTP TLS |
| Readiness never passes, logs fine | Slow first start | Wait; raise `failureThreshold` |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 6 loop prints `ROLLOUT_NOT_COMPLETE`, or the step 7 job prints `RUN_NOT_SUCCESSFUL`, read the logs (`northflank get service logs --projectId "$PROJECT_ID" --serviceId elasticsearch`) before changing anything.
- Never print `ELASTIC_PASSWORD`. Read it into a variable and pass it only to the secret group.
- Keep port 9200 private. If the user needs browser access, use `northflank forward service` or deploy Kibana in the project rather than making Elasticsearch public.
- Don't use `ES_JAVA_OPTS` to squeeze Elasticsearch onto a small plan; pick a bigger plan.
- Get explicit confirmation before deleting the `es-data` volume or the service.

## FAQ

### Where should I deploy Elasticsearch?
On a host with plenty of RAM, a persistent disk, and a private network to the apps that query it. Elastic Cloud is the managed option. A VM gives you full kernel control for large clusters. A container platform, as here, runs the official image with a volume and keeps it private, which suits single-node search for an app, logs or a RAG pipeline.

### How do I reach it from my laptop?
Run `northflank forward service --projectId "$PROJECT_ID" --serviceId elasticsearch --skipHostnames` in the background, read the printed local address, and use `curl -u elastic:<password> http://<LOCAL_ADDR>:<LOCAL_PORT>/`. See [forwarding](https://northflank.com/docs/v1/api/forwarding).

### Can I run a multi-node cluster?
Yes, but not by scaling this service. Each node needs its own service and volume, `discovery.seed_hosts` and `cluster.initial_master_nodes`, transport TLS on port 9300, and the full bootstrap checks, including `vm.max_map_count`. Confirm the host allows that setting before you plan it. Use at least three master-eligible nodes.

### How do I turn on HTTPS?
Remove `ES_SETTING_XPACK_SECURITY_HTTP_SSL_ENABLED`, provide certificates (mounted with [secret files](https://northflank.com/docs/v1/application/secure/upload-secret-files)) and point `xpack.security.http.ssl.*` at them, as Elastic's Docker Compose example does. Clients then use `https://` and trust your CA.

## Conclusion

You now have a single Elasticsearch node on the official image, with security on, a password you control, its index on a persistent volume, and a private address your services use.

**Next:** add an API key per app, schedule [volume backups](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes) or Elasticsearch snapshots, and add Kibana if you need a UI. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy ClickHouse](deploy-clickhouse.md)
- [How to deploy Kafka](deploy-kafka.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy MongoDB](deploy-mongodb.md)
