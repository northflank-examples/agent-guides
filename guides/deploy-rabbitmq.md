<!--
agent-guide-meta:
  id: NF-STK-RABBITMQ
  description: "How to deploy RabbitMQ as a managed, private broker, link its AMQP URL into producer and consumer services, and verify it through the management API."
  sources:
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-rabbitmq-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/configure-addons-for-high-availability
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/databases-and-persistence/access-a-database
    - https://northflank.com/docs/v1/application/databases-and-persistence/migrate-data-to-northflank/migrate-your-rabbitmq-deployment-to-northflank
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/create-addon
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/create-project-secret
    - https://northflank.com/docs/v1/api/project/secrets/get-project-secret-addon-link-details
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/forwarding
    - https://api.northflank.com/v1/addon-types
    - https://www.rabbitmq.com/docs/uri-spec
    - https://www.rabbitmq.com/docs/management
    - https://www.rabbitmq.com/docs/quorum-queues
  related: [NF-JOB-002, NF-DB-003, NF-DEP-006, NF-DEP-004]
-->

# How to deploy RabbitMQ

To deploy RabbitMQ, run the broker with persistent storage so durable queues survive restarts, keep it on a private network reachable only by your producers and consumers, give each service the full `amqp://` (or `amqps://` with TLS) connection URL including the vhost, and choose three replicas up front if you need quorum queues. Clients that rebuild the URL themselves often drop the vhost and fail with `ACCESS_REFUSED`.

This guide does it on Northflank with the CLI: a managed RabbitMQ addon from one JSON definition, its AMQP connection string linked into your producer and consumer through a restricted secret group, and both services restarted. You end with a private broker whose version you read from the management API through a local tunnel, and a message processed end to end, without printing a password. Every step is a command a developer or a coding agent can run as-is.

## TL;DR: how to deploy RabbitMQ

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project and broker name, and check versions with the public `https://api.northflank.com/v1/addon-types`, before changing anything.
3. Create the project, then the broker with `northflank put addon` (`type: "rabbitmq"`), and poll `northflank get addon` until `status` is `running`.
4. Link `AMQP_CONNECTION_STRING` into a secret group restricted to the producer and consumer, aliased as `AMQP_URL`, then restart both.
5. Verify through `northflank forward addon --skipHostnames` by calling the management API's `/api/overview`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Where to deploy RabbitMQ, and why a managed addon

RabbitMQ needs a long-running broker process, persistent storage so durable queues survive restarts, users and a vhost, and a private network path to both your producers and consumers. The usual places to run it are a **managed RabbitMQ** service (the provider operates the broker, storage and credentials), a **self-run container** such as the official `rabbitmq` Docker image on a container platform or Kubernetes, or a **VM** where you install and configure it yourself. Self-hosting gives full control over plugins and config but makes clustering, upgrades and backups your job; serverless hosts don't run a broker, they connect to one.

A RabbitMQ container you run yourself needs a persistent volume, users, a vhost, TLS and a plan for clustering. On Northflank, RabbitMQ is an **addon**: a managed broker inside a project, reachable by default only from services and jobs in that project. It comes with a generated user, a management user for the web UI, an admin account, AMQP and MQTT endpoints, and a 3-replica option for quorum queues.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`curl`**, for the management API check in step 8.

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

Before changing anything, print the deployment target. The broker goes in the project `my-app`; if your services already have a project, use its name instead throughout, so they reach RabbitMQ over the private network.

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId my-app -o json >/dev/null 2>&1 \
  && echo "project: my-app exists and will be reused" \
  || echo "project: my-app will be created"
northflank get addon --projectId my-app --addonId app-broker -o json >/dev/null 2>&1 \
  && echo "addon:   app-broker already exists: pick another name" \
  || echo "addon:   app-broker is free"
curl -s https://api.northflank.com/v1/addon-types \
  | jq '.data.addonTypes[] | select(.type=="rabbitmq") | {versions, majors, storage: .resources.storage.options, replicas: .resources.replicas.options}'
```

- If the addon name is taken, pick another and use it everywhere `app-broker` appears. `put addon` in step 5 would otherwise change the existing one.
- The addon-types endpoint is public. At the time of writing it listed versions `4.3.5`, `4.2.9`, `4.0.9`, `3.13.7`, `3.12.14`, storage from 4096 MB, and replicas `1` or `3`. List plan IDs with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[].id'`.
- Agree on size with whoever owns the bill. Storage can only be increased, and RabbitMQ replicas can't be reduced once increased.
- Find which variable each service reads. Celery reads `CELERY_BROKER_URL` (or `broker_url` in config); most AMQP clients (`amqplib`, `pika`, `aio-pika`, Bunny, Spring AMQP) take one URL. Grep the code for the client and the variable.

**If you're an agent:** show the user the team, the project (new or reused), the region (`europe-west` below, unless they choose another), the broker name, version, plan, storage and replicas. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "my-app", "region": "europe-west" }'
export PROJECT_ID=my-app
```

REST: `POST https://api.northflank.com/v1/projects` with the same body. For REST calls, also export an API token as `NF_API_TOKEN`.

## Step 4: Write the RabbitMQ definition

Save this as `rabbitmq.json`:

```json
{
  "name": "app-broker",
  "description": "RabbitMQ broker for API and worker",
  "type": "rabbitmq",
  "version": "4.3-latest",
  "tlsEnabled": true,
  "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "app" }
}
```

What each choice means:

- `version` accepts `latest`, a `<major>-latest` pin, or an exact version from the list.
- `tlsEnabled` can be changed later for RabbitMQ; public access requires it. With TLS, clients use the TLS ports (AMQP 5671 instead of 5672).
- The docs describe a **custom virtual host name**, otherwise random. `customCredentials.dbName` is the create-addon field for custom names; that it sets the vhost hasn't been confirmed, so check the vhost in the connection string after creation. Leave it out to accept a random vhost.
- `replicas: 3` is required for **quorum queues** (replicated, fault-tolerant queues with automatic leader failover). With 1 replica, a restart makes the broker briefly unavailable. Choose this up front, since replicas can't be reduced.
- `name` is 3–39 characters starting with a letter; the ID is its slug (`app-broker`). `storage` is in MB and holds persisted messages.

## Step 5: Create RabbitMQ and wait until it's running

```bash
northflank put addon --projectId "$PROJECT_ID" --file rabbitmq.json -o json | jq '(.data // .) | {id, status}'
```

REST:

```bash
curl -sS -X PUT "https://api.northflank.com/v1/projects/$PROJECT_ID/addons" \
  -H "Authorization: Bearer $NF_API_TOKEN" -H "Content-Type: application/json" --data @rabbitmq.json
```

Poll every 10 seconds, stopping on a failure state:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get addon --projectId "$PROJECT_ID" --addonId app-broker -o json | jq -r '(.data // .).status')
  echo "status=$S"
  case "$S" in
    running) OK=1; break ;;
    failed|error|errorAllocating|deleting|deleted) echo "provisioning failed"; break ;;
  esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon status and logs"; false; }
```

## Step 6: Inspect the connection details without printing them

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-broker -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

The RabbitMQ docs name these details:

| Detail | Use |
|---|---|
| `AMQP_CONNECTION_STRING`, `AMQP_PORT` | Producers and consumers (AMQP 5672, TLS 5671) |
| MQTT connection string | MQTT clients (1883, TLS 8883) |
| `MANAGEMENT_ENDPOINT` | Management web UI and HTTP API (15672, TLS 15671) |
| `USERNAME`, `PASSWORD` | Regular user; also logs in to the UI as a management user |
| `ADMIN_USERNAME`, `ADMIN_PASSWORD`, `RABBITMQ_ADMIN_CMD`, admin connection strings | Administration only |

STOMP and Streams are disabled by default; the docs say to contact support to enable them.

## Step 7: Link the broker into the producer and consumer

One secret group, restricted to both services, gives them the same URL. Save this as `broker-secrets.json`:

```json
{
  "name": "app-broker-connection",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": {
    "restricted": true,
    "nfObjects": [
      { "id": "<PRODUCER_SERVICE_ID>", "type": "service" },
      { "id": "<CONSUMER_SERVICE_ID>", "type": "service" }
    ]
  },
  "addonDependencies": [
    {
      "addonId": "app-broker",
      "keys": [ { "keyName": "AMQP_CONNECTION_STRING", "aliases": ["AMQP_URL", "CELERY_BROKER_URL"] } ]
    }
  ],
  "secrets": { "variables": {} }
}
```

```bash
northflank put secret --projectId "$PROJECT_ID" --file broker-secrets.json -o json
northflank get secret-link --projectId "$PROJECT_ID" --secretId app-broker-connection --addonId app-broker -o json \
  | jq '(.data // .) | {linked, linkedKeys, availableKeys}'
```

If `linkedKeys` is empty, use the exact name from `availableKeys` and re-link with `northflank update secret-link --projectId "$PROJECT_ID" --secretId app-broker-connection --addonId app-broker --input '{"keys":[{"keyName":"<KEY>","aliases":["AMQP_URL"]}]}'`. Restricting to services that don't exist yet hasn't been confirmed; deploy them first or start unrestricted and tighten later.

Restart each service and wait for `COMPLETED` (rolled out and serving):

```bash
OK=1
for SVC in <PRODUCER_SERVICE_ID> <CONSUMER_SERVICE_ID>; do
  northflank restart service --projectId "$PROJECT_ID" --serviceId $SVC -o json >/dev/null
  SVC_OK=0
  for i in $(seq 1 60); do
    D=$(northflank get service --projectId "$PROJECT_ID" --serviceId $SVC -o json | jq -r '(.data // .).status.deployment.status')
    echo "$SVC deployment=$D"; [ "$D" = "COMPLETED" ] && { SVC_OK=1; break; }; [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }; sleep 10
  done
  [ "$SVC_OK" = 1 ] || { OK=0; break; }
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
```

**In the code**, pass the URL straight to the client; it carries the user, password, host, port and vhost. A minimal Node.js pair with `amqplib`, using a durable queue and manual acks:

```js
// producer
const amqp = require('amqplib');
const conn = await amqp.connect(process.env.AMQP_URL);
const ch = await conn.createChannel();
await ch.assertQueue('jobs', { durable: true });   // add arguments: { 'x-queue-type': 'quorum' } with 3 replicas
ch.sendToQueue('jobs', Buffer.from(JSON.stringify({ id: 1 })), { persistent: true });

// consumer (a separate service with no public port)
const conn = await amqp.connect(process.env.AMQP_URL);
const ch = await conn.createChannel();
await ch.assertQueue('jobs', { durable: true });
ch.prefetch(10);
ch.consume('jobs', async (msg) => { /* do work */ ch.ack(msg); });
```

Python with `pika` is the same idea: `pika.BlockingConnection(pika.URLParameters(os.environ["AMQP_URL"]))`. With TLS on, the image needs CA certificates (`apt-get install -y ca-certificates` or `apk add --no-cache ca-certificates`). The consumer is a worker, so give it no public port.

## Step 8: Prove the broker works

Open a tunnel without sudo and read the local addresses from the log; don't assume the default ports:

```bash
northflank forward addon --projectId "$PROJECT_ID" --addonId app-broker \
  --skipHostnames --portConflict random > /tmp/nf-rabbit-fwd.log 2>&1 &
FWD_PID=$!
sleep 10; cat /tmp/nf-rabbit-fwd.log   # find the local IP:port for 15671 (TLS) or 15672
```

Query the [management HTTP API](https://www.rabbitmq.com/docs/management) with the regular user. The credentials go to curl on stdin, so they're never on the command line or printed:

```bash
CREDS=$(northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-broker -o json)
pick() { echo "$CREDS" | jq -r --arg k "$1" '(.data // .) | (.secrets + .envs) | to_entries[] | select(.key|ascii_downcase==$k) | .value' | head -1; }
printf 'user = "%s:%s"\n' "$(pick username)" "$(pick password)" \
  | curl -sk -K - "https://<LOCAL_IP>:<LOCAL_PORT>/api/overview" | jq '{rabbitmq_version, erlang_version}'
unset CREDS; kill $FWD_PID
```

`-k` skips certificate checks for this loopback test only. Use `http://` if TLS is off. The same tunnel serves the web UI at that address in a browser; log in with `USERNAME`/`PASSWORD` for the management view, or `ADMIN_USERNAME`/`ADMIN_PASSWORD` for full administration.

## How to know it worked

1. `northflank get addon --projectId "$PROJECT_ID" --addonId app-broker -o json | jq -r '(.data // .) | .status, .spec.type'` prints `running` and `rabbitmq`.
2. `/api/overview` returns a `rabbitmq_version` matching the major you asked for.
3. Both services have the URL, checked by scheme only (expect `amqps://` with TLS):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId <CONSUMER_SERVICE_ID> \
     --shell-cmd 'sh -c' --cmd '"printenv AMQP_URL | cut -c1-8"'
   ```
   `exec` drops output without a TTY, hence `script`; on Linux use `script -qefc '…' /dev/null`.
4. A message sent by the producer is processed: the consumer's logs show it, and the queue drains back to zero.
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId <CONSUMER_SERVICE_ID> --duration 300 --textIncludes "ECONNREFUSED"
   ```
   No lines means no connection errors.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ACCESS_REFUSED` / `NOT_ALLOWED` vhost | Client builds its own URL and drops the vhost | Use the linked URL unchanged; URL-encode `/` in a vhost as `%2F` |
| TLS handshake error | Image lacks CA certificates, or client ignores `amqps://` | Install `ca-certificates`; enable TLS in client options |
| `AMQP_URL` empty in a container | Service not in `restrictions`, or no restart | Fix `nfObjects`; restart |
| Quorum queue declaration fails | Fewer than 3 replicas | Create the broker with `replicas: 3` |
| Messages pile up unacked | Consumer crashes or never acks | Ack after work; set `prefetch`; check consumer logs |
| Connections rejected under load | Memory too small for the connection count | Move to a bigger plan; set limits in the admin UI |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 5 loop prints `ADDON_NOT_RUNNING`, or the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Confirm the replica count and storage with the user before step 5. Neither can be reduced.
- Never print `AMQP_URL`, passwords or `RABBITMQ_ADMIN_CMD`. Print key names; check URLs with `cut`.
- Link the regular user into apps. Admin credentials are for the UI and maintenance only.
- Don't purge or delete queues, or run `northflank delete addon`, without explicit confirmation for that specific broker. Messages in flight are lost.

## FAQ

### Where should I deploy my RabbitMQ broker?
Run it in the same region and private network as the services that publish and consume, so messages never cross the public internet. A managed RabbitMQ is the least work for most apps; self-host the `rabbitmq` Docker image on Kubernetes or a container platform, or install it on a VM, when you need specific plugins or configuration and can own clustering, upgrades and backups.

### Can I use RabbitMQ as a Celery broker?
Yes. Link the connection string as `CELERY_BROKER_URL` (as in step 7); Celery accepts `amqp://` and `amqps://` URLs. Keep a result backend separate, such as Redis or Postgres.

### Do I need 3 replicas?
Only if the queue must survive a node failure. Quorum queues need 3+ replicas; declare them with `x-queue-type: quorum`. See [configure addons for high availability](https://northflank.com/docs/v1/application/databases-and-persistence/configure-addons-for-high-availability).

### How do I limit connections per user or vhost?
Log in to the management UI as admin, open Admin, then Limits.

### How do I move an existing broker?
Follow [migrate your RabbitMQ deployment to Northflank](https://northflank.com/docs/v1/application/databases-and-persistence/migrate-data-to-northflank/migrate-your-rabbitmq-deployment-to-northflank). Backups are disk snapshots only.

## Conclusion

You now have a managed RabbitMQ broker, wired into your producer and consumer through one restricted secret group, with a version check through the management API that never printed a password.

**Next:** deploy the consumer as a [background worker](deploy-celery-in-production.md), and add [Redis](deploy-redis.md) if you need a result backend or cache. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a FastAPI app](deploy-fastapi-app.md)
- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
