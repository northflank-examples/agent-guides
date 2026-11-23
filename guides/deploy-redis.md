<!--
agent-guide-meta:
  id: NF-DB-003
  description: "How to deploy Redis as a managed instance with the right eviction policy and TLS, link REDIS_URL into your services, and verify that it answers PING."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/create-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/create-project-secret
    - https://northflank.com/docs/v1/api/project/secrets/get-project-secret-addon-link-details
    - https://northflank.com/docs/v1/api/forwarding
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-redis-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/databases-and-persistence/configure-addons-for-high-availability
    - https://northflank.com/docs/v1/application/databases-and-persistence/access-a-database
  related: [NF-DB-001, NF-JOB-002, NF-DEP-008, NF-DEP-012]
-->

# How to deploy Redis

To deploy Redis for production, run it on a private network next to your app, size its memory for your data set, choose the eviction policy for its job (`allkeys-lru` for a cache, `noeviction` for Sidekiq, BullMQ or Celery queues), enable TLS, and pass the app a `rediss://` connection URL as `REDIS_URL`. An evicting policy on a queue silently drops jobs.

This guide does it on Northflank with the CLI: a managed Redis addon from one JSON definition, with the eviction policy and Sentinel choice fixed up front, its URL linked into your services through a restricted secret group, and a restart. You end with Redis answering `PING` with `PONG` through a local tunnel and `REDIS_URL` present in your service, without printing a password. Every step is a non-interactive command a developer or a coding agent can run as-is.

## TL;DR: how to deploy Redis

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project and addon name, and check versions with the public `https://api.northflank.com/v1/addon-types`, before changing anything.
3. Create the project, then the addon from a definition with `type: "redis"`, `tlsEnabled: true` and a `typeSpecificSettings.redisMaxMemoryPolicy` chosen for your workload, using `northflank create addon --file redis.json`. Poll until `status` is `running`.
4. Link `REDIS_MASTER_URL` into a restricted secret group under the alias `REDIS_URL`, restart the service and confirm `REDIS_URL` starts with `rediss://`.
5. Verify with `northflank forward addon --skipHostnames` and `redis-cli PING`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Where to deploy Redis, and why script it

Redis needs a long-running process with enough memory for your data set, a private network path to your app, and, if it holds queue data, persistence and failover. The usual places to run it are a **managed Redis** service (the provider runs the server, TLS and credentials), a **self-run container** such as the official `redis` Docker image on a container platform or Kubernetes, or a **VM** where you install and configure Redis yourself. Self-running gives you every config knob but leaves persistence, Sentinel and upgrades to you; static and serverless hosts don't run Redis, they connect to one of these. This guide uses managed Redis.

Redis looks simple, but two settings decide whether it behaves like a cache or a queue, and on Northflank both are fixed at creation: the **eviction policy** and **Sentinel**. Getting them right in a reviewed JSON file beats discovering them after a queue silently drops jobs. Scripting also gives you a clear "ready" signal and keeps the password out of your terminal history.

On Northflank, managed Redis is an **addon** of type `redis` inside a project. By default it has only an internal address, reachable from services and jobs in the same project.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`redis-cli`**, for the PING test in step 8.

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

Before changing anything, print the deployment target. Redis goes in the project `my-app`; if your app already has a project, use its name instead throughout, so the app reaches Redis over the private network.

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId my-app -o json >/dev/null 2>&1 \
  && echo "project: my-app exists and will be reused" \
  || echo "project: my-app will be created"
northflank get addon --projectId my-app --addonId app-cache -o json >/dev/null 2>&1 \
  && echo "addon:   app-cache already exists: pick another name" \
  || echo "addon:   app-cache is free"
curl -s https://api.northflank.com/v1/addon-types \
  | jq '.data.addonTypes[] | select(.type=="redis") | {versions, storage: .resources.storage, replicas: .resources.replicas}'
```

- If the addon name is taken, pick another and use it everywhere `app-cache` appears.
- The addon-types endpoint is public, so this shows Redis versions and allowed storage sizes without a token. Run `northflank list plans -o json` to pick a plan. Redis keeps data in memory, so the plan's memory is your `maxmemory` limit. The docs recommend storage of at least the memory size, because Redis runs with AOF persistence.
- Find what Redis is for. Grep the code for the client library and for the variable it reads (`REDIS_URL`, `REDIS_HOST`…).

**If you're an agent:** show the user the team, the project (new or reused), the region (`europe-west` below, unless they choose another), the addon name, version, plan and storage. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "my-app", "region": "europe-west" }'
export PROJECT_ID=my-app
```

For REST calls, also export an API token as `NF_API_TOKEN`.

## Step 4: Choose the eviction policy

`redisMaxMemoryPolicy` controls what happens when memory is full. **It can't be changed after creation.** The default is `noeviction`.

| Use case | Policy | Why |
|---|---|---|
| Cache only | `allkeys-lru` (or `allkeys-lfu`) | Old keys are removed to make room |
| Sessions with TTLs mixed with permanent keys | `volatile-lru` | Only keys with an expiry are evicted |
| Job queues (Sidekiq, BullMQ, Celery broker) | `noeviction` | Evicting a queue key loses jobs; writes fail instead |

If one Redis will serve both a cache and a queue, use `noeviction`, or create two addons. Ask the user before choosing.

## Step 5: Write the addon definition

Save this as `redis.json`:

```json
{
  "name": "app-cache",
  "description": "Redis cache for the API",
  "type": "redis",
  "version": "latest",
  "tlsEnabled": true,
  "externalAccessEnabled": false,
  "billing": {
    "deploymentPlan": "nf-compute-20",
    "storage": 6144,
    "replicas": 1
  },
  "typeSpecificSettings": {
    "redisMaxMemoryPolicy": "allkeys-lru",
    "redisSentinelEnabled": false
  }
}
```

Notes from the create-addon schema and the Redis docs:

- `name` is 3–39 characters and starts with a letter. The ID becomes `app-cache`.
- `storage` is in megabytes and must be one of the addon-types storage options, which start at 4096 for Redis. `replicas` can be 1, 2 or 3.
- `tlsEnabled: true` makes internal connections use TLS, and the connection URL then uses the `rediss://` scheme. For Redis, the docs say TLS can be changed later; public access requires it.
- `redisSentinelEnabled` deploys Sentinel for automated failover. It needs at least 3 replicas, adds compute cost, removes the `MASTER`/`SLAVE` connection details in favour of `HOST_SENTINEL` and `PORT_SENTINEL`, needs a Sentinel-aware client, and **can't be changed after creation**.
- Without Sentinel, extra replicas are read replicas. If the primary goes down, writes are blocked until it returns, which the docs put at 1–2 minutes.
- Redis has no custom database name and no dump backups. Backups are disk snapshots only.

To pin a version instead of `latest`, the schema accepts a major with `-latest` appended (for example `8-latest`). Pick a major from the addon-types `majors` list (`8`, `7`, `6` at the time of writing), or an exact version from `versions`.

## Step 6: Create Redis and wait until it's running

```bash
northflank create addon --projectId "$PROJECT_ID" --file redis.json -o json | jq '(.data // .) | {id, status}'
```

REST:

```bash
curl -sS -X POST "https://api.northflank.com/v1/projects/$PROJECT_ID/addons" \
  -H "Authorization: Bearer $NF_API_TOKEN" -H "Content-Type: application/json" \
  --data @redis.json
```

For re-runnable scripts, `northflank put addon --projectId "$PROJECT_ID" --file redis.json` creates or updates. `create` returns 409 if the addon exists.

Poll until it's `running`, stopping on a failure state:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get addon --projectId "$PROJECT_ID" --addonId app-cache -o json | jq -r '(.data // .).status')
  echo "status=$S"
  case "$S" in
    running) OK=1; break ;;
    failed|error|errorAllocating|deleting|deleted) echo "provisioning failed"; break ;;
  esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon status and logs"; false; }
```

## Step 7: Link the connection URL into your service

The Redis docs name `REDIS_MASTER_URL` (format `[redis|rediss]://[user:password@][host][:port]`) as the connection string, plus `REDIS_CONNECT_COMMAND` for command-line clients. First, list credential **key names only**:

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-cache -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

Save this as `cache-secrets.json`, restricted to the services that need Redis:

```json
{
  "name": "app-cache-connection",
  "description": "Redis connection for the API and worker",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": {
    "restricted": true,
    "nfObjects": [
      { "id": "<SERVICE_ID>", "type": "service" },
      { "id": "<WORKER_ID>", "type": "service" }
    ]
  },
  "addonDependencies": [
    {
      "addonId": "app-cache",
      "keys": [ { "keyName": "REDIS_MASTER_URL", "aliases": ["REDIS_URL"] } ]
    }
  ],
  "secrets": { "variables": {} }
}
```

```bash
northflank create secret --projectId "$PROJECT_ID" --file cache-secrets.json -o json
northflank get secret-link --projectId "$PROJECT_ID" --secretId app-cache-connection --addonId app-cache -o json \
  | jq '(.data // .) | {linked, linkedKeys, availableKeys}'
```

If `linkedKeys` is empty or create rejected the key, pick the URL key from `availableKeys` and re-link with `northflank update secret-link … --input '{"keys":[{"keyName":"<KEY>","aliases":["REDIS_URL"]}]}'`. See [connect database secrets to workloads](https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads) for the full secret-group pattern.

Then restart and wait for `COMPLETED`, which means rolled out and serving:

```bash
northflank restart service --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> -o json
OK=0
for i in $(seq 1 60); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> -o json | jq -r '(.data // .).status.deployment.status')
  echo "deployment=$S"; [ "$S" = "COMPLETED" ] && { OK=1; break; }; [ "$S" = "FAILED" ] && { echo DEPLOY_FAILED; break; }; sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
```

**TLS in the app.** With TLS on, the URL starts with `rediss://`. Most clients (ioredis, node-redis, redis-py, the Ruby `redis` gem) switch to TLS from the scheme. If the handshake fails, the container image probably lacks CA certificates. Install `ca-certificates` in the image, as the access docs describe.

## Step 8: Prove Redis answers PING

Tunnel to the addon without sudo:

```bash
northflank forward addon --projectId "$PROJECT_ID" --addonId app-cache \
  --skipHostnames --portConflict random > /tmp/nf-redis-fwd.log 2>&1 &
FWD_PID=$!
sleep 10; cat /tmp/nf-redis-fwd.log   # read the local IP:port for 6379
```

Read the address from the log; don't assume `127.0.0.1:6379`. Pass the password through `REDISCLI_AUTH` so it isn't on the command line. The jq filter matches the key name case-insensitively and never prints the value:

```bash
export REDISCLI_AUTH=$(northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-cache -o json \
  | jq -r '(.data // .) | (.secrets + .envs) | to_entries[] | select(.key|ascii_downcase=="password") | .value' | head -1)
redis-cli --tls -h <LOCAL_IP> -p <LOCAL_PORT> PING
unset REDISCLI_AUTH; kill $FWD_PID
```

Drop `--tls` if you created the addon with `tlsEnabled: false`. If certificate verification fails because you're connecting to a loopback IP rather than the certificate's hostname, add `--insecure` for this local check only. Never disable verification in the app.

## How to know it worked

1. `northflank get addon --projectId "$PROJECT_ID" --addonId app-cache -o json | jq -r '(.data // .).status'` prints `running`, and `.spec.type` is `redis`.
2. `redis-cli … PING` prints `PONG`.
3. Inside the service, the URL has the right scheme. Print only the first characters, wrapped in `script` because `exec` drops output without a TTY:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> \
     --shell-cmd 'sh -c' --cmd '"printenv REDIS_URL | cut -c1-9"'
   ```
   Expect `rediss://` (or `redis://` without TLS). On Linux use `script -qefc '…' /dev/null`.
4. The app logs no Redis connection errors after the restart, and a cache hit appears once you request the same resource twice:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> --duration 300 --textIncludes ECONNREFUSED -o json
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `OOM command not allowed when used memory > 'maxmemory'` | `noeviction` policy and memory full | Move to a larger plan; for a pure cache, recreate with `allkeys-lru` (policy is fixed) |
| Queue jobs disappear | Eviction policy evicts queue keys | Use a separate Redis with `noeviction` for queues |
| App TLS handshake error | Image lacks CA certs, or client ignores `rediss://` | Install `ca-certificates`; enable TLS in the client options |
| `NOAUTH Authentication required` | Password not passed | Use the full `REDIS_URL`, or set the client's password option |
| `REDIS_URL` empty in container | Service not in `restrictions.nfObjects`, or no restart | Fix the restriction, restart |
| Writes fail for a minute or two | Primary replica restarting, no Sentinel | Expected without Sentinel; plan for retries or recreate with Sentinel |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 6 loop prints `ADDON_NOT_RUNNING`, or the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Confirm the eviction policy and Sentinel choice with the user before creating. Both are permanent. Storage can't be reduced, and with Sentinel a replica increase is permanent. Without Sentinel the docs conflict (the scale-a-database page says Redis replicas can be reduced; the Redis page says they can't), so treat any replica increase as possibly permanent.
- Never print `REDIS_URL`, passwords or `REDIS_CONNECT_COMMAND`. List key names; use `cut` for scheme checks.
- Keep `externalAccessEnabled: false` unless asked. If it's enabled, add `ipPolicies`.
- Don't run `FLUSHALL`, `FLUSHDB` or `northflank delete addon` without explicit confirmation for that specific instance.

## FAQ

### Where should I deploy my Redis server?
Put Redis as close to your app as possible: same region, same private network, because every cache hit pays the network round trip. A managed Redis is the simplest choice for caches, sessions and queues; run the `redis` Docker image on Kubernetes or a container platform, or install it on a VM, when you need configuration or modules a managed service doesn't expose and you're ready to own persistence and failover.

### Should I use `redis://` or `rediss://`?
Use whatever the linked `REDIS_MASTER_URL` contains. With `tlsEnabled: true` it's `rediss://`, and your client must have TLS support and CA certificates.

### Can I change the eviction policy later?
No. The docs say the policy can't be changed after creation. Create a new addon and move the data, or switch the app over if it's only a cache.

### How many connections can Redis take?
The docs list a default maximum of 10,000 concurrent connections. A larger plan handles more.

### Can I use Redis for both Sidekiq and caching?
Yes, but use `noeviction` so queue data is never evicted, or run two addons: one `allkeys-lru` cache and one `noeviction` queue. See [how to deploy Celery in production](deploy-celery-in-production.md).

## Conclusion

You now have a managed Redis instance with an eviction policy that matches its job, a TLS connection URL injected into only the services that need it, and a `PING`/`PONG` check that never printed a password. The same steps work from a laptop, CI or a coding agent.

**Next:** [schedule snapshot backups](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data) if Redis holds anything you can't rebuild, and [deploy your worker](deploy-celery-in-production.md) against the same `REDIS_URL`. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy a Rails app](deploy-rails-app.md)
- [How to deploy a full stack app](deploy-full-stack-app.md)
