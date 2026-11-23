<!--
agent-guide-meta:
  id: NF-STK-DIFY
  description: "How to deploy Dify from its docker-compose layout: API, worker, beat, web, sandbox, plugin daemon, Weaviate and nginx, with managed PostgreSQL, Redis and MinIO."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/stacks/deploy-dify
    - https://docs.dify.ai/en/self-host/quick-start/docker-compose
    - https://docs.dify.ai/en/self-host/configuration/environments
    - https://docs.dify.ai/en/self-host/deploy/troubleshooting/docker-issues
    - https://github.com/langgenius/dify/blob/1.20.0/docker/docker-compose-template.yaml
    - https://github.com/langgenius/dify/blob/1.20.0/docker/docker-compose.yaml
    - https://github.com/langgenius/dify/blob/1.20.0/docker/.env.example
    - https://github.com/langgenius/dify/tree/1.20.0/docker/envs
    - https://github.com/langgenius/dify/blob/1.20.0/docker/nginx/conf.d/default.conf.template
    - https://github.com/langgenius/dify/blob/1.20.0/api/extensions/storage/aws_s3_storage.py
    - https://github.com/langgenius/dify-plugin-daemon/blob/main/internal/db/pg/pg.go
    - https://github.com/langgenius/dify-sandbox
    - https://northflank.com/docs/v1/application/secure/inject-secrets
    - https://northflank.com/docs/v1/application/secure/upload-secret-files
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
  related: [NF-STK-LANGFLOW, NF-STK-FLOWISE, NF-STK-MINIO, NF-DB-001, NF-DB-003]
-->

# How to deploy Dify

To deploy Dify, follow its `docker/docker-compose.yaml`: run `langgenius/dify-api` three times (`MODE=api`, `worker` and `beat`), plus `dify-web`, `dify-sandbox`, `dify-plugin-daemon` and a vector store such as Weaviate, behind an nginx that routes `/console/api`, `/api`, `/v1` and `/files` to the API and everything else to the web app. Back it with PostgreSQL, Redis and shared object storage, and set your own `SECRET_KEY` and internal keys instead of the defaults.

This guide does it on Northflank with the CLI: managed PostgreSQL, Redis and MinIO, a shared secret group, eight services from Dify's official images, and nginx configured from a secret file. You end with the Dify setup page on a public HTTPS URL, verified by API and web checks. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run; a human creates the admin account. Unmapped containers are listed below.

## TL;DR: how to deploy Dify

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project and cost of eight services and three addons, then create the `dify` project.
3. Create PostgreSQL, Redis and MinIO with `northflank put addon`, then the `dify-env` secret group with their linked credentials and generated keys (`SECRET_KEY`, `PLUGIN_DAEMON_KEY`, `INNER_API_KEY_FOR_PLUGIN`, `CODE_EXECUTION_API_KEY`, `WEAVIATE_API_KEY`).
4. Deploy `weaviate`, `dify-sandbox`, `dify-plugin`, `dify-api`, `dify-worker`, `dify-beat` and `dify-web`, then `dify-nginx` with its config from a secret file.
5. Wait for the rollout and set the public URLs on the API and worker.
6. Verify that `/console/api/setup` returns `not_started` and `/install` returns 200, then have a human create the admin account.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## How Dify's Compose file maps to services

Dify 1.20.0's Compose file starts these by default, and this guide maps them as follows:

| Compose service | Here |
|---|---|
| `db_postgres`, `redis` | PostgreSQL and Redis addons |
| `./volumes/app/storage` (shared by `api` and `worker`) | MinIO addon, `STORAGE_TYPE=s3`. A volume can only attach to one service, and Dify keeps per-workspace encryption keys in storage, so API and worker must share it |
| `api`, `worker`, `worker_beat` | `dify-api`, `dify-worker`, `dify-beat` (one image, `MODE` differs) |
| `web` | `dify-web` |
| `sandbox`, `plugin_daemon` | `dify-sandbox`, `dify-plugin` (volume at `/app/storage`) |
| `weaviate` (default `COMPOSE_PROFILES`) | `weaviate` with a volume |
| `nginx` | `dify-nginx`, routes copied from `nginx/conf.d/default.conf.template` |
| `init_permissions`, `certbot` | Not needed: no shared local disk; Northflank terminates TLS |

**Not mapped:**

- **`ssrf_proxy`** (Squid). Its config is generated by an entrypoint script from three templates. Without it, the API's outbound HTTP calls go direct, so workflow HTTP nodes can reach private addresses inside the project. This guide sets the sandbox's `ENABLE_NETWORK=false` so user code can't. Treat workspace members as trusted, or port the Squid config from `docker/ssrf_proxy` yourself.
- **`agent_backend`, `local_sandbox`, `agent_ssrf_proxy`** (Agent v2 shell workspaces). They need two isolated Docker networks that private project networking doesn't reproduce. `NEXT_PUBLIC_ENABLE_AGENT_V2=false` hides the feature.
- **`api_websocket`** (the `collaboration` profile for real-time co-editing, enabled in `.env.example`'s default `COMPOSE_PROFILES`, and nginx's `/socket.io/` route to it). Workflows still save without it.

Where to host Dify follows from that list: about ten long-running processes, PostgreSQL, Redis, S3-compatible storage, two persistent disks and one HTTPS entry point. A single VM running Compose is the simplest place; you then own TLS, backups and upgrades. Serverless platforms can't host it. A container platform with managed databases runs each image as a service and keeps the data stores managed.

If you'd rather click through it, Northflank has a [Dify stack template](https://northflank.com/stacks/deploy-dify) that deploys a similar set of services from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl` and `base64`.**
- **Agreement to the cost.** This is eight services and three addons; step 2 shows where to see the prices.

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

Before changing anything, print the deployment target:

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId dify -o json >/dev/null 2>&1 \
  && echo "project: dify exists and will be reused" \
  || echo "project: dify will be created"
```

This guide creates eight services and three addons. Show the plans used below next to their prices from the public plan list (`curl -s https://api.northflank.com/v1/plans`).

**If you're an agent:** show the user the team, the services, addons and plans with their cost, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "dify", "region": "europe-west", "description": "Dify" }'
export PROJECT_ID=dify
```

## Step 4: Create PostgreSQL, Redis and MinIO

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{"name": "dify-db", "type": "postgresql", "version": "16",
  "tlsEnabled": false, "externalAccessEnabled": false, "customCredentials": { "dbName": "dify" },
  "billing": { "deploymentPlan": "nf-compute-50", "storage": 10240, "replicas": 1 }}'
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{"name": "dify-redis", "type": "redis", "version": "7.2.16",
  "tlsEnabled": false, "externalAccessEnabled": false, "billing": { "deploymentPlan": "nf-compute-20", "storage": 4096, "replicas": 1 }}'
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{"name": "dify-storage", "type": "minio", "version": "2025.10.15",
  "tlsEnabled": false, "externalAccessEnabled": false, "billing": { "deploymentPlan": "nf-compute-20", "storage": 10240, "replicas": 1 }}'

NOT_READY=""
for a in dify-db dify-redis dify-storage; do
  OK=0
  for i in $(seq 1 90); do
    S=$(northflank get addon --projectId "$PROJECT_ID" --addonId "$a" --noDefaults -o json | jq -r '(.data // .).status')
    echo "$a=$S"
    case "$S" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
    sleep 10
  done
  [ "$OK" = 1 ] || NOT_READY="$NOT_READY $a"
done
[ -z "$NOT_READY" ] || { echo "ADDON_NOT_READY:$NOT_READY: stop here and check these addons in the UI"; false; }
```

TLS is off because Dify's defaults (`DB_SSL_MODE=disable`, `REDIS_USE_SSL=false`, an `http://` S3 endpoint) expect plain connections; all three are reachable only inside the project.

## Step 5: Check credential key names

Print key names only, never values:

```bash
for a in dify-db dify-redis dify-storage; do
  northflank get addon credentials --projectId "$PROJECT_ID" --addonId "$a" --noDefaults -o json \
    | jq --arg a "$a" '(.data // .) | {addon: $a, keys: ((.secrets + .envs) | keys)}'
done
```

If a name or its casing differs from the `keyName` values in step 6, change them there.

## Step 6: Create the shared secret group

Generate the internal keys once and keep them in this shell; the sandbox and Weaviate definitions need two of them:

```bash
export SBX_KEY=$(openssl rand -hex 24) WV_KEY=$(openssl rand -hex 24)
jq -n --arg sk "$(openssl rand -base64 42)" --arg pd "$(openssl rand -hex 32)" --arg inner "$(openssl rand -hex 32)" \
      --arg init "$(openssl rand -hex 12)" --arg sbx "$SBX_KEY" --arg wv "$WV_KEY" '{
  name: "dify-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ "dify-api", "dify-worker", "dify-beat", "dify-plugin" | { id: ., type: "service" } ] },
  addonDependencies: [
    { addonId: "dify-db", keys: [ { keyName: "HOST", aliases: ["DB_HOST"] }, { keyName: "PORT", aliases: ["DB_PORT"] },
      { keyName: "USERNAME", aliases: ["DB_USERNAME"] }, { keyName: "PASSWORD", aliases: ["DB_PASSWORD"] }, { keyName: "DATABASE", aliases: ["DB_DATABASE"] } ] },
    { addonId: "dify-redis", keys: [ { keyName: "HOST", aliases: ["REDIS_HOST"] }, { keyName: "PORT", aliases: ["REDIS_PORT"] },
      { keyName: "PASSWORD", aliases: ["REDIS_PASSWORD"] } ] },
    { addonId: "dify-storage", keys: [ { keyName: "MINIO_ENDPOINT", aliases: ["S3_ENDPOINT"] },
      { keyName: "ACCESS_KEY", aliases: ["S3_ACCESS_KEY"] }, { keyName: "SECRET_KEY", aliases: ["S3_SECRET_KEY"] } ] } ],
  secrets: { variables: {
    SECRET_KEY: $sk, INIT_PASSWORD: $init, DB_TYPE: "postgresql", REDIS_USE_SSL: "false",
    CELERY_BROKER_URL: "redis://:${REDIS_PASSWORD}@${REDIS_HOST}:${REDIS_PORT}/1",
    STORAGE_TYPE: "s3", S3_BUCKET_NAME: "dify", S3_REGION: "us-east-1", S3_ADDRESS_STYLE: "path",
    VECTOR_STORE: "weaviate", WEAVIATE_ENDPOINT: "http://weaviate:8080", WEAVIATE_GRPC_ENDPOINT: "grpc://weaviate:50051", WEAVIATE_API_KEY: $wv,
    CODE_EXECUTION_ENDPOINT: "http://dify-sandbox:8194", CODE_EXECUTION_API_KEY: $sbx,
    PLUGIN_DAEMON_URL: "http://dify-plugin:5002", PLUGIN_DAEMON_KEY: $pd, INNER_API_KEY_FOR_PLUGIN: $inner,
    MIGRATION_ENABLED: "true"
  } }
}' > dify-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file dify-env.json >/dev/null && rm dify-env.json
```

Dify creates the `dify` bucket on first use if it's missing. `INIT_PASSWORD` protects the first-run page; **tell the user to read it from the `dify-env` secret group in the UI**. If the restricted create is rejected, re-run with `restrictions: { restricted: false }`.

## Step 7: Deploy the backing services

A small helper writes and applies each definition:

```bash
svc() { # name image plan env-json ports-json probes-json
  jq -n --arg n "$1" --arg i "$2" --arg p "$3" --argjson env "$4" --argjson ports "$5" --argjson hc "$6" '{
    name: $n, billing: { deploymentPlan: $p }, ports: $ports, runtimeEnvironment: $env, healthChecks: $hc,
    deployment: { instances: 1, external: { imagePath: $i }, docker: { configType: "default" } } }' > "$1.json"
  northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file "$1.json" >/dev/null && rm "$1.json" && echo "put $1"
}
port() { jq -nc --arg n "$1" --argjson p "$2" --arg pr "${3:-HTTP}" '[{name: $n, internalPort: $p, public: false, protocol: $pr}]'; }
tcp()  { jq -nc --argjson p "$1" '[{protocol: "TCP", type: "readinessProbe", port: $p, initialDelaySeconds: 10, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 6, successThreshold: 1}]'; }

svc weaviate cr.weaviate.io/semitechnologies/weaviate:1.39.13 nf-compute-100-2 "$(jq -nc --arg k "$WV_KEY" '{
  PERSISTENCE_DATA_PATH: "/var/lib/weaviate", QUERY_DEFAULTS_LIMIT: "25", DEFAULT_VECTORIZER_MODULE: "none", CLUSTER_HOSTNAME: "node1",
  AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: "false", AUTHENTICATION_APIKEY_ENABLED: "true", AUTHENTICATION_APIKEY_ALLOWED_KEYS: $k,
  AUTHENTICATION_APIKEY_USERS: "hello@dify.ai", AUTHORIZATION_ADMINLIST_ENABLED: "true", AUTHORIZATION_ADMINLIST_USERS: "hello@dify.ai"}')" \
  "$(jq -nc --argjson a "$(port http 8080)" --argjson b "$(port grpc 50051 TCP)" '$a + $b')" "$(tcp 8080)"

svc dify-sandbox langgenius/dify-sandbox:0.2.15 nf-compute-100-1 "$(jq -nc --arg k "$SBX_KEY" '{
  API_KEY: $k, GIN_MODE: "release", WORKER_TIMEOUT: "15", ENABLE_NETWORK: "false", SANDBOX_PORT: "8194"}')" "$(port http 8194)" "$(tcp 8194)"

svc dify-plugin langgenius/dify-plugin-daemon:0.6.15-local nf-compute-100-2 '{
  "DB_DATABASE": "dify_plugin", "DB_DEFAULT_DATABASE": "dify", "DB_SSL_MODE": "disable", "SERVER_PORT": "5002",
  "SERVER_KEY": "${PLUGIN_DAEMON_KEY}", "DIFY_INNER_API_URL": "http://dify-api:5001", "DIFY_INNER_API_KEY": "${INNER_API_KEY_FOR_PLUGIN}",
  "PLUGIN_STORAGE_TYPE": "local", "PLUGIN_STORAGE_LOCAL_ROOT": "/app/storage", "PLUGIN_WORKING_PATH": "/app/storage/cwd",
  "PLUGIN_INSTALLED_PATH": "plugin", "PLUGIN_PACKAGE_CACHE_PATH": "plugin_packages", "PLUGIN_MEDIA_CACHE_PATH": "assets",
  "FORCE_VERIFYING_SIGNATURE": "true", "PLUGIN_REMOTE_INSTALLING_HOST": "0.0.0.0", "PLUGIN_REMOTE_INSTALLING_PORT": "5003"}' \
  "$(port http 5002)" "$(tcp 5002)"

vol() { northflank get volume --projectId "$PROJECT_ID" --volumeId "$1" --noDefaults -o json >/dev/null 2>&1 || \
  northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input "$(jq -nc --arg v "$1" --arg m "$2" --arg s "$3" '{
    name: $v, mounts: [{containerMountPath: $m}], spec: {accessMode: "ReadWriteOnce", storageSize: 10240}, attachedObjects: [{id: $s, type: "service"}]}')" >/dev/null; }
vol weaviate-data /var/lib/weaviate weaviate
vol plugin-data /app/storage dify-plugin
```

The plugin daemon's variable names differ from the API's, so `SERVER_KEY` and `DIFY_INNER_API_KEY` reference the group's values with `${…}` templating. `DB_DATABASE` set on the service overrides the linked one, and the daemon creates `dify_plugin` on first start. The images are the tags pinned in Dify 1.20.0's Compose file.

## Step 8: Deploy the API, worker, beat and web

```bash
svc dify-api langgenius/dify-api:1.20.0 nf-compute-200-4 '{"MODE": "api"}' "$(port http 5001)" \
  '[{"protocol": "HTTP", "type": "startupProbe", "path": "/health", "port": 5001, "initialDelaySeconds": 20, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 60, "successThreshold": 1}]'
svc dify-worker langgenius/dify-api:1.20.0 nf-compute-200-4 '{"MODE": "worker"}' '[]' '[]'
svc dify-beat langgenius/dify-api:1.20.0 nf-compute-50 '{"MODE": "beat"}' '[]' '[]'
svc dify-web langgenius/dify-web:1.20.0 nf-compute-100-1 '{"SERVER_CONSOLE_API_URL": "http://dify-api:5001",
  "NEXT_PUBLIC_ENABLE_AGENT_V2": "false", "NEXT_TELEMETRY_DISABLED": "1"}' "$(port http 3000)" "$(tcp 3000)"
```

The API runs database migrations on start (`MIGRATION_ENABLED=true`), which is why its startup probe allows up to ten minutes.

## Step 9: Deploy nginx with Dify's routes

```bash
cat > default.conf <<'EOF'
server {
  listen 80;
  client_max_body_size 100M;
  proxy_http_version 1.1;
  proxy_buffering off;
  proxy_read_timeout 3600s;
  proxy_set_header Host $host;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto https;
  location /console/api { proxy_pass http://dify-api:5001; }
  location /api         { proxy_pass http://dify-api:5001; }
  location /v1          { proxy_pass http://dify-api:5001; }
  location /files       { proxy_pass http://dify-api:5001; }
  location /openapi     { proxy_pass http://dify-api:5001; }
  location /mcp         { proxy_pass http://dify-api:5001; }
  location /triggers    { proxy_pass http://dify-api:5001; }
  location /callbacks/human-input/v2/im/ { proxy_pass http://dify-api:5001; }
  location /e/ {
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Dify-Hook-Url https://$host$request_uri;
    proxy_pass http://dify-plugin:5002;
  }
  location / { proxy_pass http://dify-web:3000; }
}
EOF
jq -n --arg d "$(base64 < default.conf | tr -d '\n')" '{
  name: "dify-nginx-conf", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "dify-nginx", type: "service" } ] },
  secrets: { files: { "/etc/nginx/conf.d/default.conf": { data: $d, encoding: "utf-8" } } }
}' > nginx-conf.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file nginx-conf.json >/dev/null && rm nginx-conf.json default.conf

svc dify-nginx nginx:1.30.7-alpine nf-compute-20 '{}' \
  '[{"name": "http", "internalPort": 80, "public": true, "protocol": "HTTP"}]' "$(tcp 80)"
```

The config avoids `${…}`, because Northflank templates secret files. A location that sets its own headers doesn't inherit the server-level ones, so `/e/` repeats them.

## Step 10: Set the public URLs

```bash
NOT_READY=""
for s in dify-api dify-worker dify-beat dify-web dify-plugin dify-nginx; do
  OK=0
  for i in $(seq 1 90); do
    D=$(northflank get service --projectId "$PROJECT_ID" --serviceId "$s" --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
    [ "$D" = "COMPLETED" ] && { echo "$s ready"; OK=1; break; }
    [ "$D" = "FAILED" ] && { echo "$s FAILED"; break; }
    sleep 10
  done
  [ "$OK" = 1 ] || NOT_READY="$NOT_READY $s"
done
[ -z "$NOT_READY" ] || { echo "ROLLOUT_NOT_COMPLETE:$NOT_READY: stop here and read those services' logs"; false; }
OK=0; [ -z "$NOT_READY" ] && OK=1
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId dify-nginx --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId dify-nginx --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi

[ -n "$HOST" ] && for s in api worker; do
  northflank patch service deployment --projectId "$PROJECT_ID" --serviceId "dify-$s" --noDefaults -o json --input "$(jq -n --arg m "$s" --arg u "https://$HOST" '{
    runtimeEnvironment: { MODE: $m, CONSOLE_API_URL: $u, CONSOLE_WEB_URL: $u, SERVICE_API_URL: $u, APP_WEB_URL: $u, FILES_URL: $u,
      TRIGGER_URL: $u, ENDPOINT_URL_TEMPLATE: ($u + "/e/{hook_id}") } }')" >/dev/null
  northflank restart service --projectId "$PROJECT_ID" --serviceId "dify-$s" --noDefaults -o json >/dev/null
done
```

If it prints `ROLLOUT_NOT_COMPLETE`, the patch doesn't run: read the named services' logs with `northflank get service logs --projectId "$PROJECT_ID" --serviceId <service>`. The web app calls the API on the same origin through nginx, so it needs no public URL. `FILES_URL` gives model providers absolute links to uploaded images, and `TRIGGER_URL` is the base for webhook trigger URLs (it defaults to `http://localhost:5001`).

**This step needs a human:** open `https://$HOST/install`, enter `INIT_PASSWORD`, and create the admin account.

## How to know it worked

You're done when all of these hold:

1. nginx routes to the API, and setup hasn't run yet:
   ```bash
   curl -s "https://$HOST/console/api/setup" | jq '.'                               # {"step": "not_started"}
   ```
2. nginx routes to the web app: `curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/install"` prints `200`.
3. Migrations ran: `northflank get service logs --projectId "$PROJECT_ID" --serviceId dify-api --lineLimit 300 | grep -i migrat`.
4. The API reaches the plugin daemon:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId dify-api \
     --cmd 'curl -s -o /dev/null -w %{http_code} http://dify-plugin:5002/health/check'
   ```
   Expect `200`. (Linux: `script -qefc '<cmd>' /dev/null`.)
5. After setup, a human installs a model provider from the marketplace (plugin daemon), uploads a document to a knowledge base and sees it reach **Available** (worker, MinIO and Weaviate).

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `dify-plugin` logs show permission denied on `CREATE DATABASE` | Addon user can't create databases | Create a second PostgreSQL addon with `dbName: dify_plugin` and link its keys in a group restricted to `dify-plugin` |
| API logs `Connection refused` to `db_postgres` or `redis` | Linked keys missing | Re-check step 5 names; restart |
| Celery errors on `redis://:@:/1` | Templating didn't resolve `CELERY_BROKER_URL` | Link Redis `REDIS_MASTER_URL` as `CELERY_BROKER_URL` instead |
| nginx exits with `host not found in upstream` | It started before an upstream existed | Restart `dify-nginx` after the step 8 services exist |
| Plugin install fails, `permission denied` in `/app/storage` | Volume ownership | Restart once; if it persists, report it |
| Knowledge indexing stays **Queuing** | Worker down or broker URL wrong | Check `dify-worker` logs |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 10 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Never print any value from `dify-env`, `SBX_KEY`, `WV_KEY` or addon credentials. Print key names only.
- Never change `SECRET_KEY`, `PLUGIN_DAEMON_KEY` or `INNER_API_KEY_FOR_PLUGIN` on a running install without updating every service that uses them.
- Tell the user plainly that the SSRF proxy and Agent v2 containers aren't deployed, and what that means.
- Get explicit confirmation before deleting addons or volumes. Back up PostgreSQL before upgrading Dify.

## FAQ

### Where should I deploy Dify?
On infrastructure that can run about ten containers side by side with PostgreSQL, Redis, S3-compatible storage and persistent disks behind one HTTPS URL. A single VM with Docker Compose is the quickest; a container platform with managed databases, as here, runs each image as its own service with managed data stores.

### Can I use pgvector instead of Weaviate?
Dify supports `VECTOR_STORE=pgvector`, but it needs the `vector` extension in PostgreSQL. Check the extension is available on your database before switching; otherwise keep Weaviate.

### How do I upgrade Dify?
Back up PostgreSQL, read the release notes, then re-run the `svc` lines for `dify-api`, `dify-worker`, `dify-beat` and `dify-web` with the new tag, plus the sandbox and plugin daemon tags from that release's Compose file. Migrations run when `dify-api` starts.

## Conclusion

You now have Dify's core stack on managed PostgreSQL, Redis and MinIO, with the API, workers, web app, sandbox, plugin daemon and Weaviate behind nginx on a public HTTPS URL, and the unmapped containers called out.

**Next:** add a custom domain to `dify-nginx` by [linking a domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port), then repeat step 10 with it. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy Langflow](deploy-langflow.md)
- [How to deploy Flowise](deploy-flowise.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
