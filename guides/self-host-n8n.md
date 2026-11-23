<!--
agent-guide-meta:
  id: NF-STK-N8N
  description: "How to self-host n8n in production with the official Docker image, managed PostgreSQL, a fixed encryption key, a .n8n volume and public HTTPS webhook URLs."
  sources:
    - https://northflank.com/stacks/deploy-n8n
    - https://docs.n8n.io/hosting/installation/docker/
    - https://docs.n8n.io/hosting/configuration/environment-variables/database/
    - https://docs.n8n.io/hosting/configuration/environment-variables/deployment/
    - https://docs.n8n.io/hosting/configuration/environment-variables/endpoints/
    - https://docs.n8n.io/hosting/configuration/configuration-examples/webhook-url/
    - https://docs.n8n.io/hosting/logging-monitoring/monitoring/
    - https://docs.n8n.io/hosting/scaling/queue-mode/
    - https://docs.n8n.io/hosting/configuration/environment-variables/queue-mode/
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DB-001, NF-DB-003, NF-DEP-002, NF-JOB-002]
-->

# How to self-host n8n

To self-host n8n in production, run the official n8n Docker image against PostgreSQL (`DB_TYPE=postgresdb`), set a fixed `N8N_ENCRYPTION_KEY`, persist `/home/node/.n8n` on a volume, and set `N8N_WEBHOOK_URL` and `N8N_PROXY_HOPS=1` so n8n uses its public HTTPS address. Without the webhook URL, n8n shows `http://localhost:5678` links behind a proxy, and a lost encryption key makes every stored credential unreadable.

This guide does it on Northflank with the CLI: a managed PostgreSQL addon, a secret group holding the database credentials and a generated encryption key, an `n8nio/n8n` service and a volume for `.n8n`. You end with a public HTTPS editor whose webhook URLs point at that address, verified with `curl` and `printenv` checks. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human creates the owner account once.

## TL;DR: how to self-host n8n

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, the project and the n8n version before changing anything.
3. Create a project, a PostgreSQL addon, and a secret group with the database credentials and a generated `N8N_ENCRYPTION_KEY`.
4. Run `n8nio/n8n` with `northflank put service deployment` on port 5678 and attach a volume at `/home/node/.n8n`.
5. Wait for the rollout, set `N8N_HOST`, `N8N_EDITOR_BASE_URL` and `N8N_WEBHOOK_URL` to the public host, restart, and create the owner account.
6. Verify that `/healthz/readiness` returns 200, `DB_TYPE` is `postgresdb` and `N8N_WEBHOOK_URL` is your `https://` host.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

n8n's defaults suit a laptop, not a server. By default it uses SQLite and generates a random encryption key on first launch, and it builds webhook URLs from `N8N_PROTOCOL`, `N8N_HOST` and `N8N_PORT`. Behind a TLS-terminating proxy, the n8n docs say that "won't work". So this setup changes three things:

- **PostgreSQL** (`DB_TYPE=postgresdb`) holds workflows, credentials and executions.
- **A fixed `N8N_ENCRYPTION_KEY`** keeps stored credentials readable across restarts and future workers.
- **`N8N_WEBHOOK_URL` and `N8N_PROXY_HOPS=1`** make n8n show and register the public HTTPS URL.

The n8n Docker docs still recommend persisting `/home/node/.n8n` with PostgreSQL, so a volume goes there.

Where to host n8n comes down to what it needs: a process that stays running (schedules and webhooks fire at any hour), a PostgreSQL database, persistent storage and a stable public HTTPS URL. Serverless and static hosts are built for short request-driven functions, so they don't fit a long-running n8n process. A raw VM or your own Kubernetes cluster works, but you manage the proxy, TLS, database backups and upgrades yourself. A container platform with managed databases sits in between: you run the same Docker image and the platform handles the rest.

If you'd rather click through it, Northflank has an [n8n stack template](https://northflank.com/stacks/deploy-n8n) that deploys the same pieces from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`** to generate the encryption key (`openssl version`).

Names used below: project `n8n`, addon `n8n-db`, secret group `n8n-env`, service `n8n`, volume `n8n-data`.

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

Before changing anything, print the deployment target and pin an n8n version. The image path must include a tag, and this picks the newest plain version tag from Docker Hub:

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId n8n -o json >/dev/null 2>&1 \
  && echo "project: n8n exists and will be reused" \
  || echo "project: n8n will be created"
N8N_VERSION=$(curl -s "https://hub.docker.com/v2/repositories/n8nio/n8n/tags?page_size=100&ordering=last_updated" \
  | jq -r '.results[].name' | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -n1); echo "n8n:     $N8N_VERSION"
```

n8n needs more than the smallest plan; this guide uses `nf-compute-100-2` (1 vCPU, 2 GB). List the others with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[].id'`.

**If you're an agent:** show the user the team, the region (`europe-west` below, unless they choose another), whether the project is new or reused, and the n8n version. Wait for them to confirm before you create anything.

## Step 3: Create a project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "n8n", "region": "europe-west" }'
export PROJECT_ID=n8n
```

## Step 4: Create PostgreSQL and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "n8n-db", "type": "postgresql", "version": "16-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "n8n" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId n8n-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon status and logs"; false; }
```

The database only accepts traffic from inside the project, so this guide turns off TLS to keep n8n's default `DB_POSTGRESDB_SSL_ENABLED=false` working. If you want TLS, set `tlsEnabled: true` and `DB_POSTGRESDB_SSL_ENABLED=true`. `dbName` and the TLS setting are fixed at creation, and storage can only be increased.

## Step 5: Create the secret group

n8n reads separate `DB_POSTGRESDB_*` variables rather than a connection URL. First print the credential **key names** (never the values):

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId n8n-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

The PostgreSQL docs list `HOST`, `PORT`, `USERNAME`, `PASSWORD` and `DATABASE`. If the output uses different casing, change the `keyName` values below to match. Then generate the encryption key and create the group without echoing the key:

```bash
N8N_ENCRYPTION_KEY=$(openssl rand -hex 32)
jq -n --arg key "$N8N_ENCRYPTION_KEY" '{
  name: "n8n-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "n8n-db", keys: [
    { keyName: "HOST",     aliases: ["DB_POSTGRESDB_HOST"] },
    { keyName: "PORT",     aliases: ["DB_POSTGRESDB_PORT"] },
    { keyName: "DATABASE", aliases: ["DB_POSTGRESDB_DATABASE"] },
    { keyName: "USERNAME", aliases: ["DB_POSTGRESDB_USER"] },
    { keyName: "PASSWORD", aliases: ["DB_POSTGRESDB_PASSWORD"] }
  ] } ],
  secrets: { variables: { DB_TYPE: "postgresdb", N8N_ENCRYPTION_KEY: $key } }
}' > n8n-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file n8n-env.json >/dev/null && rm n8n-env.json
unset N8N_ENCRYPTION_KEY
```

The secret group is now the only copy of the encryption key. **Tell the user to copy it to their password manager from the secret group in the UI.** If it's lost, n8n can't decrypt any stored credentials.

## Step 6: Deploy the n8n service

Write `n8n.json`. Replace `<N8N_VERSION>` with the tag from step 2.

```json
{
  "name": "n8n",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "n8nio/n8n:<N8N_VERSION>" },
    "docker": { "configType": "default" }
  },
  "ports": [ { "name": "http", "internalPort": 5678, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "N8N_PORT": "5678", "N8N_PROXY_HOPS": "1", "GENERIC_TIMEZONE": "UTC", "TZ": "UTC" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/healthz/readiness", "port": 5678,
      "initialDelaySeconds": 15, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 6, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/healthz", "port": 5678,
      "initialDelaySeconds": 60, "periodSeconds": 20, "timeoutSeconds": 3, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file n8n.json
```

According to the n8n monitoring docs, `/healthz/readiness` returns 200 only once the database is connected and migrated. `/healthz` only shows the process is up, so the liveness probe uses that one.

## Step 7: Attach a volume for `/home/node/.n8n`

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId n8n-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "n8n-data",
  "mounts": [ { "containerMountPath": "/home/node/.n8n" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 6144 },
  "attachedObjects": [ { "id": "n8n", "type": "service" } ]
}'
```

`create volume` returns 409 if the volume exists, hence the `get` guard. Attaching redeploys the service. With a single read/write volume the service is limited to one instance, and on restart the old container stops before the new one starts, so expect a few seconds of downtime.

## Step 8: Wait for the rollout and point n8n at its public URL

Wait for the first rollout and read the hostname Northflank assigned to the public port:

```bash
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId n8n --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId n8n --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId n8n --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId n8n`. The troubleshooting table covers the common causes.

Then send the complete `runtimeEnvironment` map, because the patch replaces the map, and restart:

```bash
northflank patch service deployment --projectId "$PROJECT_ID" --serviceId n8n --noDefaults -o json --input "$(jq -n --arg h "$HOST" '{
  runtimeEnvironment: {
    N8N_PORT: "5678", N8N_PROXY_HOPS: "1", GENERIC_TIMEZONE: "UTC", TZ: "UTC",
    N8N_HOST: $h, N8N_EDITOR_BASE_URL: ("https://" + $h + "/"),
    N8N_WEBHOOK_URL: ("https://" + $h + "/"), WEBHOOK_URL: ("https://" + $h + "/")
  }
}')"
northflank restart service --projectId "$PROJECT_ID" --serviceId n8n --noDefaults -o json
```

`N8N_WEBHOOK_URL` replaces `WEBHOOK_URL`, which is deprecated from n8n 2.35.0. Older versions only read `WEBHOOK_URL`, so this sets both. On 2.35.0 and later, drop `WEBHOOK_URL` to silence the deprecation warning. When you add a custom domain, repeat this step with that hostname.

## Step 9: Wait for the restart and create the owner account

Give the restart about 15 seconds to begin, then run the step 8 wait loop again (the lines up to `ROLLOUT_NOT_COMPLETE`) until it ends with `deployment=COMPLETED`. `COMPLETED` means rolled out and serving.

**This step needs a human:** open `https://$HOST` right away and create the owner account. Until someone does, anyone who finds the URL can claim the instance.

## How to know it worked

You're done when all of these hold:

1. Readiness (database connected and migrated) returns 200:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/healthz/readiness"
   ```
2. n8n is using PostgreSQL and the public webhook URL. These print only safe prefixes (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId n8n \
     --shell-cmd 'sh -c' --cmd '"printenv DB_TYPE N8N_WEBHOOK_URL; printenv DB_POSTGRESDB_HOST | cut -c1-6"'
   ```
   Expect `postgresdb`, `https://<HOST>/` and the start of a hostname, not `localhost`.
3. The volume is attached to `n8n`:
   ```bash
   northflank get volume --projectId "$PROJECT_ID" --volumeId n8n-data --noDefaults -o json | jq '(.data // .) | {status, attachedObjects}'
   ```
4. After the owner account exists, a workflow with a Webhook node shows a production URL that starts with `https://<HOST>/webhook/`.

## Optional: queue mode with Redis and workers

For heavier workloads, n8n's queue mode has the main instance put executions on a Redis queue and separate workers run them. To set it up:

1. Create a Redis addon (`"type": "redis"`) the same way as step 4.
2. Add `EXECUTIONS_MODE=queue` to the secret group. Then link Redis's host, port and password as `QUEUE_BULL_REDIS_HOST`, `QUEUE_BULL_REDIS_PORT` and `QUEUE_BULL_REDIS_PASSWORD`, printing key names first as in step 5. If Redis has TLS on, also set `QUEUE_BULL_REDIS_TLS=true`.
3. Deploy a second deployment service from the same image, with no public port, `"docker": { "configType": "customCommand", "customCommand": "worker" }` and the same secret group. The n8n docs start workers with `docker run … n8nio/n8n worker`.

Workers share the encryption key and database through the same secret group. In queue mode n8n stores binary data in the database by default. This option hasn't been run end to end here.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Readiness never passes, logs show `ECONNREFUSED 127.0.0.1:5432` | `DB_POSTGRESDB_*` not injected | Check key names from step 5; restart |
| Logs show a TLS/SSL error to Postgres | Addon created with TLS, n8n SSL off | Set `DB_POSTGRESDB_SSL_ENABLED=true` in the group |
| Webhook URLs show `http://localhost:5678` | Step 8 not applied, or service not restarted | Re-run step 8 |
| `Credentials could not be decrypted` | `N8N_ENCRYPTION_KEY` changed | Restore the original key; never regenerate it |
| `EACCES` on `/home/node/.n8n` | Volume owned by another user | Restart once; if it persists, report it. See [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) |
| Can't scale to 2 instances | Single read/write volume | Use queue mode workers instead |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Don't report success until checks 1 and 2 pass: readiness returns 200 and `N8N_WEBHOOK_URL` is the public `https://` host.
- Never print `N8N_ENCRYPTION_KEY`, database passwords or the credentials response. Print key names only.
- Never regenerate or overwrite `N8N_ENCRYPTION_KEY` on an instance that already has credentials.
- Hand the owner-account step to the user immediately after the first rollout.
- Get explicit confirmation before deleting the addon, the volume or the service. Prefer pausing. Back up the database before an n8n major-version upgrade.

## FAQ

### Do I need the volume if I use PostgreSQL?
The n8n Docker docs say the `.n8n` folder still holds things like the encryption key file and instance logs, and recommend persisting it even with PostgreSQL. Setting `N8N_ENCRYPTION_KEY` covers the most important part, and the volume covers the rest.

### Can I use MySQL instead?
No. MySQL and MariaDB support was deprecated in n8n 1.0, and the current database docs list only `sqlite` and `postgresdb` for `DB_TYPE`. Use PostgreSQL.

### How do I upgrade n8n?
Back up the database, then change the tag with `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId n8n --input '{"deployment":{"instances":1,"external":{"imagePath":"n8nio/n8n:<NEW_VERSION>"}}}'` and wait for `COMPLETED`. n8n runs its database migrations on start.

### Where should I host n8n?
Anywhere that can run the n8n Docker image as an always-on container next to PostgreSQL and a persistent volume, behind a public HTTPS URL. That rules out serverless functions and static hosts. A VM or Kubernetes cluster works if you're happy to run the proxy, TLS and database yourself; a container platform with managed PostgreSQL, like the setup in this guide, takes those off your hands.

## Conclusion

You now have n8n on the official image, backed by managed PostgreSQL, with a fixed encryption key, a persistent `.n8n` folder, and webhook URLs that point at the public HTTPS address.

**Next:** add a custom domain by [linking a domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) (then repeat step 8 with that hostname), and add Redis for queue mode when you outgrow one process. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
