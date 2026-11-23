<!--
agent-guide-meta:
  id: NF-STK-KAFKA
  description: "How to deploy Kafka in KRaft mode from the official apache/kafka image on a private network with a volume, health checks and a scripted produce/consume test."
  sources:
    - https://northflank.com/stacks/deploy-kafka
    - https://hub.docker.com/r/apache/kafka
    - https://github.com/apache/kafka/blob/trunk/docker/examples/README.md
    - https://github.com/apache/kafka/blob/trunk/docker/examples/docker-compose-files/single-node/plaintext/docker-compose.yml
    - https://github.com/apache/kafka/blob/trunk/docker/jvm/Dockerfile
    - https://github.com/apache/kafka/blob/trunk/docker/resources/common-scripts/configure
    - https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/docker/KafkaDockerWrapper.scala
    - https://kafka.apache.org/documentation/#kraft
    - https://northflank.com/docs/v1/application/network/networking-on-northflank
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/forwarding
  related: [NF-STK-RABBITMQ, NF-DB-003, NF-DEP-002, NF-STK-CLICKHOUSE, NF-STK-ELASTIC]
-->

# How to deploy Kafka

To deploy Kafka, run the official `apache/kafka` image as a single KRaft node that acts as both broker and controller, advertise a listener on a hostname your clients can resolve, keep the log directory on a persistent volume, and set internal topic replication factors to 1. Most failed Kafka deploys come from advertised listeners that point at `localhost` or an address clients can't reach.

This guide does it on Northflank with the CLI: an `apache/kafka` service on a private TCP port with TCP health checks, a volume for the log directory, and a one-off job that creates a topic, produces a message and consumes it back over the private address `broker:9092`. Every step is a copy-pasteable command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end.

## TL;DR: how to deploy Kafka

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project and service name, and pin a Kafka version, before changing anything.
3. Create the project, generate a `CLUSTER_ID`, and run `apache/kafka:<version>` with `northflank put service deployment`: KRaft `broker,controller` roles, listener `PLAINTEXT://:9092` advertised as `broker:9092`, a private TCP port and TCP probes.
4. Attach a volume at `/var/lib/kafka/data` with `northflank create volume` and wait for `COMPLETED`.
5. Run a test job from the same image and verify it concluded with `SUCCESS` and its logs show the message you produced.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Kafka is a long-running, stateful TCP server. It needs a process that never scales to zero, a disk that survives restarts, and a network path that clients reach by a stable name. Serverless and static hosts can't run it. A VM works if you'll manage the JVM, disks and upgrades yourself; managed Kafka services take that away at a price; a container platform with volumes and private networking runs the official image with little configuration.

Three things in the official image need setting on a server:

- **KRaft roles.** Kafka 4.x has no ZooKeeper. One node with `KAFKA_PROCESS_ROLES=broker,controller` runs both roles, as in the upstream single-node example.
- **Listeners.** Clients connect to the bootstrap address, then reconnect to whatever the broker *advertises*. Advertising `broker:9092`, the private address other services in the project use, is what makes clients work.
- **Replication factors.** Internal topics default to a replication factor of 3, which a single broker can't satisfy. The upstream example sets them to 1.

Kafka isn't a Northflank addon type, so it runs as a deployment service. The image converts every `KAFKA_*` environment variable into a broker property (`KAFKA_LOG_DIRS` becomes `log.dirs`), except JVM and logging variables such as `KAFKA_HEAP_OPTS` and `KAFKA_OPTS`; configuration is just environment variables. Once you set any of them, the image builds `server.properties` from the environment alone instead of its default file, so the full KRaft set below is required.

If you'd rather click through it, Northflank has a [Kafka stack template](https://northflank.com/stacks/deploy-kafka) that deploys the official image in KRaft mode with persistent storage from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`**, to generate the cluster ID.

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

Before changing anything, print the deployment target. Names used below: project `data`, service `broker`, volume `kafka-data`, job `kafka-smoke`. If the apps that use Kafka already have a project, use its name instead of `data`, because private ports are project-scoped.

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId data -o json >/dev/null 2>&1 \
  && echo "project: data exists and will be reused" \
  || echo "project: data will be created"
northflank get service --projectId data --serviceId broker -o json >/dev/null 2>&1 \
  && echo "service: broker already exists: step 5 would update it" \
  || echo "service: broker is free"
```

Then pin a Kafka version. This prints the newest plain release tag (no `-rc`):

```bash
KAFKA_VERSION=$(curl -s "https://hub.docker.com/v2/repositories/apache/kafka/tags?page_size=100&ordering=last_updated" \
  | jq -r '.results[].name' | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -n1); echo "$KAFKA_VERSION"   # e.g. 4.4.0
```

- If the service already exists, `put service` in step 5 would change it. Pick another name, or confirm that updating it is intended.
- The broker JVM uses a 1 GB heap here, so `nf-compute-100-2` (1 vCPU, 2 GB) is the smallest sensible plan. List plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | "\(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'`.

**If you're an agent:** show the user the team, the project (new or reused), the region (`europe-west` below, unless they choose another), the plan and the Kafka version. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "data", "region": "europe-west", "description": "Data services" }'
export PROJECT_ID=data
```

## Step 4: Generate a cluster ID

```bash
CLUSTER_ID=$(openssl rand 16 | base64 | tr '+/' '-_' | tr -d '='); echo "$CLUSTER_ID"
```

KRaft formats the log directory with the cluster ID on first start, so **keep this value**: changing it later makes the broker refuse the existing data. The image falls back to a built-in default ID if you don't set one, which works but is shared by every default install.

## Step 5: Deploy the broker

Write `broker.json` with `jq`, so the version and cluster ID are filled in:

```bash
jq -n --arg v "$KAFKA_VERSION" --arg cid "$CLUSTER_ID" '{
  name: "broker",
  billing: { deploymentPlan: "nf-compute-100-2" },
  deployment: { instances: 1, external: { imagePath: ("apache/kafka:" + $v) }, docker: { configType: "default" } },
  ports: [ { name: "kafka", internalPort: 9092, public: false, protocol: "TCP" } ],
  runtimeEnvironment: {
    KAFKA_NODE_ID: "1",
    KAFKA_PROCESS_ROLES: "broker,controller",
    KAFKA_LISTENERS: "PLAINTEXT://:9092,CONTROLLER://:9093",
    KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://broker:9092",
    KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT",
    KAFKA_CONTROLLER_LISTENER_NAMES: "CONTROLLER",
    KAFKA_INTER_BROKER_LISTENER_NAME: "PLAINTEXT",
    KAFKA_CONTROLLER_QUORUM_VOTERS: "1@localhost:9093",
    CLUSTER_ID: $cid,
    KAFKA_LOG_DIRS: "/var/lib/kafka/data/logs",
    KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: "1",
    KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: "1",
    KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: "1",
    KAFKA_SHARE_COORDINATOR_STATE_TOPIC_REPLICATION_FACTOR: "1",
    KAFKA_SHARE_COORDINATOR_STATE_TOPIC_MIN_ISR: "1",
    KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: "0",
    KAFKA_HEAP_OPTS: "-Xms1g -Xmx1g"
  },
  healthChecks: [
    { protocol: "TCP", type: "readinessProbe", port: 9092, initialDelaySeconds: 15, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 6, successThreshold: 1 },
    { protocol: "TCP", type: "livenessProbe", port: 9092, initialDelaySeconds: 60, periodSeconds: 20, timeoutSeconds: 3, failureThreshold: 3 }
  ]
}' > broker.json

northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file broker.json
```

REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/deployment`. What the settings do:

- **`public: false`, `protocol: TCP`.** Kafka speaks its own binary protocol, and Northflank's public ports only carry HTTP and HTTP/2. A private TCP port is reachable from every service and job in the project at `broker:9092` (the docs' `[service-name]:[port-number]` form).
- **`KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://broker:9092`.** This is the address clients reconnect to after bootstrapping, so it must be the private name, never `localhost`.
- **`KAFKA_CONTROLLER_QUORUM_VOTERS=1@localhost:9093`.** The controller talks to itself over the loopback. Pointing it at `broker:9093` would route through the service address, which only sends traffic to *ready* containers, and the broker isn't ready until the controller is up.
- **`KAFKA_LOG_DIRS` in a subdirectory.** A fresh volume can contain a `lost+found` directory at its root, which Kafka would treat as a bad log directory.
- **The service is named `broker`.** It avoids any `KAFKA_*`-prefixed variable the platform might add for a service called `kafka`, which the image would turn into a broker property.
- **TCP probes on 9092.** The listener only opens once the node has started. Kafka has no HTTP health endpoint.

`PLAINTEXT` means no authentication or encryption, so only workloads inside the project can reach it. For SASL or TLS, see the FAQ.

## Step 6: Attach the volume and wait for the rollout

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId kafka-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "kafka-data",
  "mounts": [ { "containerMountPath": "/var/lib/kafka/data" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "broker", "type": "service" } ]
}'

OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId broker --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

`/var/lib/kafka/data` is the data volume declared in the image's Dockerfile. `storageSize` is in MB and can only grow, so size it for your retention. Attaching redeploys the service, and a `ReadWriteOnce` volume limits it to one instance; on restart the old container stops before the new one starts. `create volume` returns 409 if the volume exists, hence the `get` guard.

## Step 7: Run a produce/consume test from a client job

A manual job from the same image acts as a Kafka client inside the project. `put job` makes a manual job when `settings.cron` is absent (`put job manual` is deprecated in the API):

```bash
SMOKE="cd /opt/kafka/bin && ./kafka-topics.sh --bootstrap-server broker:9092 --create --if-not-exists --topic smoke --partitions 1 --replication-factor 1 && echo hello-from-northflank | ./kafka-console-producer.sh --bootstrap-server broker:9092 --topic smoke && ./kafka-console-consumer.sh --bootstrap-server broker:9092 --topic smoke --from-beginning --max-messages 1 --timeout-ms 30000"

jq -n --arg v "$KAFKA_VERSION" --arg s "$SMOKE" '{
  name: "kafka-smoke",
  billing: { deploymentPlan: "nf-compute-50" },
  settings: { backoffLimit: 0, activeDeadlineSeconds: 600, runOnSourceChange: "never" },
  deployment: {
    external: { imagePath: ("apache/kafka:" + $v) },
    docker: { configType: "customEntrypointCustomCommand", customEntrypoint: "/bin/bash", customCommand: ("-c " + ($s | @sh)) }
  }
}' > kafka-smoke.json
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file kafka-smoke.json

RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId kafka-smoke --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 60); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId kafka-smoke --runId "$RUN_ID" --noDefaults -o json)
  echo "run=$(echo "$R" | jq -r '(.data // .).status')"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$(echo "$R" | jq -r '(.data // .).status')" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId kafka-smoke --runId "$RUN_ID" --lineLimit 100
[ "$OK" = 1 ] || { echo "RUN_NOT_SUCCESSFUL: stop here and read the logs above"; false; }
```

The scripts live in `/opt/kafka/bin`, as in the image's quick start. The custom entrypoint replaces the image's broker start command, so the job only runs the client tools. Re-run it any time with `northflank start job run`.

## How to know it worked

You're done when all of these hold:

1. The run concluded with `status` `SUCCESS`, and the job logs contain `hello-from-northflank` and `Processed a total of 1 messages`.
2. The broker's deployment status is `COMPLETED`, and it has no public port:
   ```bash
   northflank get service --projectId "$PROJECT_ID" --serviceId broker --noDefaults -o json \
     | jq -e '[(.data // .).ports[] | select(.public == true)] | length == 0'
   ```
3. The data survives a restart. Run `northflank restart service --projectId "$PROJECT_ID" --serviceId broker --noDefaults -o json`, wait for `COMPLETED`, then run the job again: the consumer reads from the beginning, so `smoke` still holds the first message.
4. The broker logs show KRaft mode and no listener errors:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId broker --lineLimit 200 | grep -iE 'kraft|error' | tail -n 20
   ```

Other services in the project connect with `bootstrap.servers=broker:9092`.

## Multi-broker caveats

One node is fine for development, queues and modest event streams, but it has no fault tolerance. For production replication:

- **Run each node as its own service** (`kafka-0`, `kafka-1`, `kafka-2`), each with its own volume, a unique `KAFKA_NODE_ID`, and an advertised listener using its own service name. Don't scale one service to three instances: a `ReadWriteOnce` volume limits a service to one instance, and Kafka clients must reach specific brokers rather than a load-balanced address.
- **Use three controllers.** A KRaft quorum of three tolerates one failure. List all three as `KAFKA_CONTROLLER_QUORUM_VOTERS` (`1@kafka-0:9093,2@kafka-1:9093,3@kafka-2:9093`), add private ports for 9093, and be aware the readiness-gated service address issue above then applies to controller traffic. That setup hasn't been tested here.
- **Raise the replication factors** back to 3 and set `min.insync.replicas` to 2.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Logs: `AccessDeniedException` on `/var/lib/kafka/data/logs` | Volume root not writable by `appuser` | Report it; see [volume permissions](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) |
| Client hangs after connecting, or tries `localhost:9092` | Wrong advertised listener | Set `KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://broker:9092` and redeploy |
| Broker never becomes ready, logs show controller connection retries | Quorum voters point at the service address | Use `1@localhost:9093` |
| Logs report a cluster ID mismatch after a redeploy | `CLUSTER_ID` changed | Restore the original ID |
| `Replication factor: 3 larger than available brokers: 1` | Internal topic factors left at default | Keep the `*_REPLICATION_FACTOR` variables at 1 |
| Container restarts with `OutOfMemoryError` or is killed | Heap too large for the plan | Lower `KAFKA_HEAP_OPTS` or pick a bigger plan |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 6 loop prints `ROLLOUT_NOT_COMPLETE`, or the step 7 job prints `RUN_NOT_SUCCESSFUL`, read the logs (`northflank get service logs --projectId "$PROJECT_ID" --serviceId broker`) before changing anything.
- Keep the port private. Northflank's public ports only carry HTTP, and a `PLAINTEXT` Kafka listener has no authentication.
- Never change `CLUSTER_ID` or `KAFKA_NODE_ID` on a broker that already has data.
- Send the complete `runtimeEnvironment` map when you patch the service; partial maps can drop keys.
- Get explicit confirmation before deleting the `kafka-data` volume or the service. Deleting the volume deletes every topic.

## FAQ

### Where should I deploy Kafka?
On a host that runs a long-lived JVM container with a persistent disk and a private network your producers and consumers share. A managed Kafka service suits large clusters if you're happy to pay for it. A VM works if you'll handle disks, upgrades and the JVM yourself. A container platform, as in this guide, runs the official image with a volume and a private address for your other services.

### Can I reach Kafka from my laptop?
Not simply. `northflank forward service` can tunnel the port, but clients then follow the advertised `broker:9092` address. Without `--skipHostnames` the CLI configures local hostnames (it may ask for sudo), which may make that name resolve locally; this hasn't been tested. Run clients inside the project, as the test job does.

### How do I add authentication or TLS?
The image supports SASL and SSL listeners through environment variables, with key stores in `/etc/kafka/secrets` and a JAAS file passed in `KAFKA_OPTS`. See the [Apache Kafka Docker examples](https://github.com/apache/kafka/blob/trunk/docker/examples/README.md). Mount the files with [secret files](https://northflank.com/docs/v1/application/secure/upload-secret-files).

### How do I upgrade Kafka?
Patch the image tag and wait for `COMPLETED`: `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId broker --noDefaults -o json --input '{"deployment":{"instances":1,"external":{"imagePath":"apache/kafka:<NEW_VERSION>"}}}'`. Read the Kafka upgrade notes first, and back up the volume with [volume backups](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes).

## Conclusion

You now have a single-node KRaft Kafka broker on the official image, with its log directory on a persistent volume, reachable only inside your project at `broker:9092`, and a test job that proves produce and consume work.

**Next:** point your producers and consumers at `broker:9092`, add a volume backup schedule, and plan the three-node layout above before you rely on it for critical data. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy RabbitMQ](deploy-rabbitmq.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy ClickHouse](deploy-clickhouse.md)
- [How to deploy Elasticsearch](deploy-elasticsearch.md)
