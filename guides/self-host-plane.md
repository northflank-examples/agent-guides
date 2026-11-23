<!--
agent-guide-meta:
  id: NF-STK-PLANE
  description: "How to self-host Plane with its official images: API, workers, web apps and Caddy proxy on managed Postgres, Redis, RabbitMQ and MinIO, plus a migration job."
  sources:
    - https://northflank.com/stacks/deploy-plane
    - https://developers.plane.so/self-hosting/methods/docker-compose
    - https://developers.plane.so/self-hosting/govern/instance-admin
    - https://github.com/makeplane/plane/blob/v1.5.0/deployments/cli/community/docker-compose.yml
    - https://github.com/makeplane/plane/blob/v1.5.0/deployments/cli/community/variables.env
    - https://github.com/makeplane/plane/blob/v1.5.0/apps/proxy/Caddyfile.ce
    - https://github.com/makeplane/plane/blob/v1.5.0/apps/api/plane/settings/common.py
    - https://github.com/makeplane/plane/tree/v1.5.0/apps/api/bin
    - https://hub.docker.com/r/makeplane/plane-backend/tags
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-redis-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-rabbitmq-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-minio-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/secure/upload-secret-files
    - https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/secrets/get-project-secret-addon-link-details
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/project/jobs/get-run-details
  related: [NF-DB-001, NF-DB-003, NF-STK-RABBITMQ, NF-STK-MINIO, NF-STK-MATTERMOST]
-->

# How to self-host Plane

To self-host Plane, run its official images as separate containers: the `plane-backend` image as API, Celery worker, beat scheduler and one-off migrator, plus the web, space, admin and live front ends behind the `plane-proxy` Caddy router. Back them with PostgreSQL, Redis, RabbitMQ and S3-compatible storage, and set `WEB_URL`, `SECRET_KEY` and `LIVE_SERVER_SECRET_KEY`. RabbitMQ isn't optional: Plane's Celery broker is AMQP.

This guide does it on Northflank with the CLI: four managed addons (PostgreSQL, Redis, RabbitMQ, MinIO), two secret groups, the migrator as a job, seven private services and one public proxy. You end with a public HTTPS Plane instance whose API reports itself activated, verified with `curl` and `printenv` checks. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run; a human creates the instance admin once.

## TL;DR: how to self-host Plane

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Pin a Plane release, and confirm the team, project and region before changing anything.
3. Create a project, then PostgreSQL, Redis, RabbitMQ and MinIO, and a secret group linking their credentials plus generated `SECRET_KEY` and `LIVE_SERVER_SECRET_KEY`.
4. Deploy the public `plane-proxy` with a Caddyfile in `runtimeFiles`, store its URL as `WEB_URL`, and run the migrations as a job.
5. Deploy `api`, `worker`, `beat-worker`, `web`, `space`, `admin` and `live`, then create the instance admin at `/god-mode/`.
6. Verify that `/` returns 200 and `/api/instances/` reports `is_activated: true`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Plane is not one container. Its official community `docker-compose.yml` (v1.5.0) runs 12 of them:

| Compose service | Image | What it does | Here |
|---|---|---|---|
| `api` | `plane-backend` | Django API on 8000; registers the instance and creates the bucket on start | service |
| `worker`, `beat-worker` | `plane-backend` | Celery worker and scheduler | services, no ports |
| `migrator` | `plane-backend` | `manage.py migrate`, then exits | job |
| `web`, `space`, `admin`, `live` | `plane-frontend`, `plane-space`, `plane-admin`, `plane-live` | App, public pages, `/god-mode` admin, real-time editor, all on 3000 | services |
| `proxy` | `plane-proxy` | Caddy routing `/api`, `/auth`, `/spaces`, `/god-mode`, `/live` and `/uploads` | public service |
| `plane-db`, `plane-redis`, `plane-mq`, `plane-minio` | Postgres 15, Valkey 7.2, RabbitMQ 3.13, S3-compatible storage (`pgsty/silo`, a MinIO build) | Data stores | addons |

The proxy's Caddyfile hard-codes the upstreams `web:3000`, `api:8000`, `space:3000`, `admin:3000` and `live:3000`. Northflank services in one project reach each other at `[service-name]:[port]`, so naming the services exactly that makes the routing work unchanged. The one exception is MinIO: the Caddyfile points at `plane-minio:9000`, and an addon's hostname isn't its name. Step 6 mounts a copy of the Caddyfile that reads the MinIO endpoint from the environment instead.

Plane's docs ask for at least 2 cores and 4 GB of RAM (8 GB recommended for production), and strongly recommend external database and storage. Where to host Plane comes down to that: several always-on containers that find each other by name, four stateful backing services and one HTTPS entry point. Serverless and static hosts don't fit. A VM with Docker Compose works but keeps every data store on one disk you back up yourself. A container platform with managed addons splits the stateless parts from the stateful ones.

This is a moderately complex deployment: 4 addons, 2 secret groups, 8 services and a job. If you'd rather click through it, Northflank has a [Plane stack template](https://northflank.com/stacks/deploy-plane) that deploys a similar setup from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`**, **`openssl`** and **`base64`**.
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one. This guide uses about 5 vCPU and 8 GB across all containers.

Names used below: project `plane`; addons `plane-db`, `plane-redis`, `plane-mq`, `plane-minio`; secret groups `plane-env` and `plane-urls`; job `plane-migrate`.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Pin a release and confirm where it will deploy

Before changing anything, pin a Plane release and print the deployment target. All six images share one tag. This guide uses `v1.5.0`; the first command shows the newest one:

```bash
curl -s "https://hub.docker.com/v2/repositories/makeplane/plane-backend/tags?page_size=20&ordering=last_updated" \
  | jq -r '.results[].name' | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -n1
export PLANE_TAG=v1.5.0

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId plane -o json >/dev/null 2>&1 \
  && echo "project: plane exists and will be reused" \
  || echo "project: plane will be created"
```

- The guide was written against `v1.5.0`. A newer tag may change the compose layout or the Caddyfile; check Plane's release notes before using it.
- If the project is reused and already has Plane services or addons, stop and ask the user: the steps below change them, and step 5 generates new keys.
- List plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | "\(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'`.

**If you're an agent:** show the user the team, the Plane tag, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create a project

A project holds your services, addons and jobs. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "plane", "region": "europe-west" }'
export PROJECT_ID=plane
```

## Step 4: Create the four addons and wait for them

TLS is off on all four because they're only reachable inside the project, and Plane's `rediss://` mode verifies certificates by default. Versions follow Plane's compose file.

```bash
mk_addon() { # name type version plan storageMB
  northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input "$(jq -n \
    --arg n "$1" --arg t "$2" --arg v "$3" --arg p "$4" --argjson s "$5" '{
      name: $n, type: $t, version: $v, tlsEnabled: false, externalAccessEnabled: false,
      billing: { deploymentPlan: $p, storage: $s, replicas: 1 } }
      + (if $t == "postgresql" then { customCredentials: { dbName: "plane" } } else {} end)')" >/dev/null
}
mk_addon plane-db    postgresql 15-latest   nf-compute-50 6144
mk_addon plane-redis redis      7.2-latest  nf-compute-20 4096
mk_addon plane-mq    rabbitmq   3.13-latest nf-compute-20 4096
mk_addon plane-minio minio      latest      nf-compute-20 12288

OK=1
for a in plane-db plane-redis plane-mq plane-minio; do
  A_OK=0
  for i in $(seq 1 90); do
    A=$(northflank get addon --projectId "$PROJECT_ID" --addonId "$a" --noDefaults -o json | jq -r '(.data // .).status')
    echo "$a=$A"
    case "$A" in running) A_OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
    sleep 10
  done
  [ "$A_OK" = 1 ] || { OK=0; break; }
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon status and logs"; false; }
```

Storage can only be increased. MinIO holds every attachment, so size it for your uploads; MinIO replicas (1 or 4) can't be changed later.

## Step 5: Create the main secret group

Print the credential **key names** (never the values) of each addon:

```bash
for a in plane-db plane-redis plane-mq plane-minio; do
  northflank get addon credentials --projectId "$PROJECT_ID" --addonId "$a" --noDefaults -o json \
    | jq --arg a "$a" '(.data // .) | {addon: $a, secrets: (.secrets | keys), envs: (.envs | keys)}'
done
```

Then create the group. Plane reads `DATABASE_URL`, `REDIS_URL`, `AMQP_URL` and the `AWS_*` variables; with `USE_MINIO=1` it serves files through `WEB_URL/<bucket>`, which the proxy forwards to MinIO.

```bash
jq -n --arg sk "$(openssl rand -hex 32)" --arg lk "$(openssl rand -hex 32)" '{
  name: "plane-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [
    { addonId: "plane-db",    keys: [ { keyName: "POSTGRES_URI", aliases: ["DATABASE_URL"] } ] },
    { addonId: "plane-redis", keys: [ { keyName: "REDIS_MASTER_URL", aliases: ["REDIS_URL"] } ] },
    { addonId: "plane-mq",    keys: [ { keyName: "AMQP_CONNECTION_STRING", aliases: ["AMQP_URL"] } ] },
    { addonId: "plane-minio", keys: [
      { keyName: "MINIO_ENDPOINT", aliases: ["AWS_S3_ENDPOINT_URL"] },
      { keyName: "ACCESS_KEY",     aliases: ["AWS_ACCESS_KEY_ID"] },
      { keyName: "SECRET_KEY",     aliases: ["AWS_SECRET_ACCESS_KEY"] } ] }
  ],
  secrets: { variables: {
    SECRET_KEY: $sk, LIVE_SERVER_SECRET_KEY: $lk,
    USE_MINIO: "1", MINIO_ENDPOINT_SSL: "0", AWS_S3_BUCKET_NAME: "uploads", BUCKET_NAME: "uploads",
    FILE_SIZE_LIMIT: "5242880", GUNICORN_WORKERS: "1", DEBUG: "0", NUM_PROXIES: "2",
    API_BASE_URL: "http://api:8000"
  } }
}' > plane-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file plane-env.json >/dev/null && rm plane-env.json

for a in plane-db plane-redis plane-mq plane-minio; do
  northflank get secret-link --projectId "$PROJECT_ID" --secretId plane-env --addonId "$a" --noDefaults -o json \
    | jq --arg a "$a" '(.data // .) | {addon: $a, linkedKeys, availableKeys}'
done
```

A linked key reaches the containers only under its default name (`NF_<ADDON>_<KEY>`, shown as `defaultKey` in `linkedKeys`) and its aliases, never under the bare key name. So the MinIO key `SECRET_KEY` arrives as `NF_PLANE-MINIO_SECRET_KEY` and `AWS_SECRET_ACCESS_KEY`, and doesn't collide with Plane's Django `SECRET_KEY`, which is a plain variable in the group. If a key is missing from `linkedKeys`, use the exact name from `availableKeys` and re-run the `put secret`. The group is unrestricted, so every service and job in the project inherits it. Don't regenerate `SECRET_KEY` on a running instance: re-running the `jq` command creates new keys, so edit the group in the UI instead.

## Step 6: Deploy the public proxy

The proxy goes first because its hostname becomes `WEB_URL`. This Caddyfile is Plane's `Caddyfile.ce` with two changes: Northflank already terminates TLS, so it listens on `:80` without ACME, and MinIO is reached through `{$AWS_S3_ENDPOINT_URL}` instead of `plane-minio:9000`.

```bash
cat > Caddyfile <<'EOF'
{
	auto_https off
	servers {
		max_header_size 25MB
		client_ip_headers X-Forwarded-For X-Real-IP
		trusted_proxies static private_ranges
	}
}

:80 {
	request_body {
		max_size {$FILE_SIZE_LIMIT}
	}
	redir /spaces /spaces/ permanent
	reverse_proxy /spaces/* space:3000
	redir /god-mode /god-mode/ permanent
	reverse_proxy /god-mode/* admin:3000
	reverse_proxy /live/* live:3000
	reverse_proxy /api/* api:8000
	reverse_proxy /auth/* api:8000
	reverse_proxy /static/* api:8000
	reverse_proxy /{$BUCKET_NAME}/* {$AWS_S3_ENDPOINT_URL}
	reverse_proxy /{$BUCKET_NAME} {$AWS_S3_ENDPOINT_URL}
	reverse_proxy /* web:3000
}
EOF

northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --input "$(jq -n \
  --arg img "makeplane/plane-proxy:$PLANE_TAG" --arg cf "$(base64 < Caddyfile | tr -d '\n')" '{
  name: "proxy",
  billing: { deploymentPlan: "nf-compute-20" },
  deployment: { instances: 1, external: { imagePath: $img }, docker: { configType: "default" } },
  ports: [ { name: "http", internalPort: 80, public: true, protocol: "HTTP" } ],
  runtimeFiles: { "/etc/caddy/Caddyfile": { data: $cf, encoding: "utf-8" } },
  healthChecks: [ { protocol: "TCP", type: "readinessProbe", port: 80,
    initialDelaySeconds: 5, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 6, successThreshold: 1 } ]
}')"

for i in $(seq 1 30); do
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId proxy --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId proxy --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns // empty' | head -n1)
  [ -n "$HOST" ] && break; sleep 10
done
[ -n "$HOST" ] || { echo "NO_PUBLIC_HOST: stop here and check the proxy service"; false; }
echo "https://$HOST"
```

Caddy resolves upstreams per request, so it starts before the other services exist and returns 502 until they're up.

## Step 7: Store the public URL

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input "$(jq -n --arg h "$HOST" '{
  name: "plane-urls", type: "secret", secretType: "environment", priority: 20,
  restrictions: { restricted: false },
  secrets: { variables: { WEB_URL: ("https://" + $h), CORS_ALLOWED_ORIGINS: ("https://" + $h) } }
}')" >/dev/null
```

When you add a custom domain later, update both values and restart `api`, `worker` and `beat-worker`.

## Step 8: Run the migrations as a job

`put job` without `settings.cron` creates a manual job. The command is the compose file's `migrator` entrypoint.

```bash
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --input "$(jq -n --arg img "makeplane/plane-backend:$PLANE_TAG" '{
  name: "plane-migrate",
  billing: { deploymentPlan: "nf-compute-50" },
  settings: { backoffLimit: 0, activeDeadlineSeconds: 1800, runOnSourceChange: "never" },
  deployment: { external: { imagePath: $img },
    docker: { configType: "customCommand", customCommand: "./bin/docker-entrypoint-migrator.sh" } }
}')" >/dev/null

RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId plane-migrate --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 120); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId plane-migrate --runId "$RUN_ID" --noDefaults -o json)
  S=$(echo "$R" | jq -r '(.data // .).status'); echo "run=$S"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$S" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId plane-migrate --runId "$RUN_ID" --lineLimit 50
[ "$OK" = 1 ] || { echo "JOB_NOT_SUCCEEDED: stop here; the logs above show why"; false; }
```

Run status is `RUNNING`, `SUCCESS` or `FAILED`. Re-run this job before starting a new Plane version.

## Step 9: Deploy the Plane services

The API, worker and scheduler wait for migrations before starting. The service names must match the Caddyfile upstreams.

```bash
svc() { # name image command port plan
  northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --input "$(jq -n \
    --arg n "$1" --arg img "makeplane/$2:$PLANE_TAG" --arg cmd "$3" --arg port "$4" --arg plan "$5" '{
    name: $n,
    billing: { deploymentPlan: $plan },
    deployment: { instances: 1, external: { imagePath: $img },
      docker: (if $cmd == "" then { configType: "default" } else { configType: "customCommand", customCommand: $cmd } end) }
  } + (if $port == "" then {} else {
    ports: [ { name: "http", internalPort: ($port | tonumber), public: false, protocol: "HTTP" } ],
    healthChecks: [ { protocol: "TCP", type: "readinessProbe", port: ($port | tonumber),
      initialDelaySeconds: 10, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 30, successThreshold: 1 } ]
  } end)')" >/dev/null
}
svc api         plane-backend  ./bin/docker-entrypoint-api.sh    8000 nf-compute-100-2
svc worker      plane-backend  ./bin/docker-entrypoint-worker.sh ""   nf-compute-50
svc beat-worker plane-backend  ./bin/docker-entrypoint-beat.sh   ""   nf-compute-20
svc web         plane-frontend ""                                3000 nf-compute-50
svc space       plane-space    ""                                3000 nf-compute-50
svc admin       plane-admin    ""                                3000 nf-compute-50
svc live        plane-live     ""                                3000 nf-compute-50

OK=1
for s in api worker beat-worker web space admin live proxy; do
  S_OK=0
  for i in $(seq 1 60); do
    D=$(northflank get service --projectId "$PROJECT_ID" --serviceId "$s" --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
    echo "$s=$D"
    [ "$D" = "COMPLETED" ] && { S_OK=1; break; }
    [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
    sleep 10
  done
  [ "$S_OK" = 1 ] || { OK=0; break; }
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

`COMPLETED` means rolled out and serving. TCP probes are used because the API runs `collectstatic` and instance registration before it binds port 8000.

**This step needs a human:** open `https://$HOST/god-mode/` right away and complete "Let's secure your instance" with an email and password. That account becomes the instance admin; until then the instance isn't set up.

## How to know it worked

You're done when all of these hold:

1. The migrate run concluded with `status == "SUCCESS"` (step 8).
2. The app and admin pages load through the proxy:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"            # 200
   curl -s -o /dev/null -w '%{http_code}\n' -L "https://$HOST/god-mode/" # 200
   ```
3. The API registered the instance (`is_setup_done` turns `true` after the human step):
   ```bash
   curl -s "https://$HOST/api/instances/" | jq '{activated: .config.is_activated, setup: .instance.is_setup_done}'
   ```
4. The API sees the right URL and stores (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId api \
     --shell-cmd 'sh -c' --cmd '"printenv WEB_URL USE_MINIO; printenv DATABASE_URL | cut -c1-11; printenv AMQP_URL | cut -c1-7; printenv AWS_SECRET_ACCESS_KEY >/dev/null && echo S3_KEY_SET"'
   ```
   Expect `https://<HOST>`, `1`, `postgresql:`, `amqp://` and `S3_KEY_SET` (no secret is printed).
5. The bucket path reaches MinIO, not the web app: `curl -s "https://$HOST/uploads/" | head -c 200` returns MinIO's XML (for example `AccessDenied`), not HTML. `NoSuchBucket` means the API hasn't created the bucket yet.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| 502 on every path | Upstream service not `COMPLETED`, or misnamed | Names must be exactly `api`, `web`, `space`, `admin`, `live` |
| API never becomes ready, logs loop on `wait_for_migrations` | Migrate job not run or failed | Re-run step 8 and read its logs |
| Worker logs `Connection refused` to `localhost:5672` | `AMQP_URL` not injected | Check `linkedKeys` for `plane-mq` in step 5; restart the worker |
| Check 4 doesn't print `S3_KEY_SET`, or uploads fail with `InvalidAccessKeyId` | MinIO keys not linked under the `AWS_*` aliases | Check `linkedKeys` for `plane-minio` in step 5; restart `api`, `worker` and `beat-worker` |
| Images and attachments don't load | `/uploads` not routed, or `WEB_URL` wrong | Check check 5 and that `WEB_URL` matches the browser's host |
| Sign-in loops, or links use `http://` | Forwarded proto not trusted | Confirm `trusted_proxies` in the Caddyfile; report it if it persists |
| Editor shows "connecting" forever | `live` down or `LIVE_SERVER_SECRET_KEY` missing | Check `live` logs and the group |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write: team, Plane tag, region and whether the project is new or reused.
- Don't report success until checks 1 to 5 pass.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read that service's logs before changing anything.
- Never print `SECRET_KEY`, `LIVE_SERVER_SECRET_KEY`, connection URLs or credentials responses. Print key names and safe prefixes only.
- Don't re-run the `jq` in step 5 on a live instance: it generates new keys and invalidates sessions.
- Hand the `/god-mode/` step to the user immediately after the first rollout.
- To upgrade, change `PLANE_TAG` for all eight images together, re-run step 8 first, then step 9. Back up `plane-db` before a version change.
- Get explicit confirmation before deleting any addon. MinIO holds every attachment.

## FAQ

### Is RabbitMQ required?
Yes, in v1.5.0. Plane's settings set Celery's broker from `AMQP_URL` (or the `RABBITMQ_*` variables) and the compose file always runs RabbitMQ. Redis is used for caching and the live server, not as the broker.

### Can I use AWS S3 instead of MinIO?
Yes. Set `USE_MINIO=0`, the real bucket in `AWS_S3_BUCKET_NAME`, `AWS_REGION` and access keys, and drop `AWS_S3_ENDPOINT_URL` and the MinIO link. The proxy's bucket routes then go unused, because Plane hands out S3 URLs directly.

### Is self-hosted Plane free?
The Community Edition images used here are open source under AGPL-3.0. Paid Plane editions use different images and a license key, which this guide doesn't cover.

### Where should I host Plane?
Anywhere that can run eight always-on containers that reach each other by name, next to PostgreSQL, Redis, RabbitMQ and S3-compatible storage, behind one HTTPS URL. A single VM with Docker Compose works for a small team if you handle backups and TLS yourself. A container platform with managed data stores, like this setup, keeps the stateful parts out of your containers.

## Conclusion

You now have Plane Community Edition on its official images: API, workers, front ends and the Caddy router as separate services, migrations as a job, and PostgreSQL, Redis, RabbitMQ and MinIO as managed addons, served at a public HTTPS URL.

**Next:** add a custom domain by [linking a domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) on `proxy`, then update `plane-urls` and restart the backend services. Configure SMTP in `/god-mode/` so invitations can be sent. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy RabbitMQ](deploy-rabbitmq.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
- [How to deploy Mattermost](deploy-mattermost.md)
