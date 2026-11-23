<!--
agent-guide-meta:
  id: NF-STK-SUPABASE
  description: "How to self-host Supabase with the supabase/postgres image, generated JWT, anon and service keys, Auth, PostgREST, Studio and a Kong gateway, all scripted."
  sources:
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/stacks/deploy-supabase
    - https://supabase.com/docs/guides/self-hosting/docker
    - https://github.com/supabase/supabase/blob/master/docker/docker-compose.yml
    - https://github.com/supabase/supabase/blob/master/docker/docker-compose.kong.yml
    - https://github.com/supabase/supabase/blob/master/docker/volumes/api/kong.yml
    - https://github.com/supabase/supabase/blob/master/docker/volumes/api/kong-entrypoint.sh
    - https://github.com/supabase/supabase/blob/master/docker/volumes/db/roles.sql
    - https://github.com/supabase/supabase/blob/master/docker/utils/generate-keys.sh
    - https://github.com/supabase/postgres/blob/develop/migrations/db/migrate.sh
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/application/secure/upload-secret-files
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule
  related: [NF-DB-001, NF-DEP-002, NF-STK-MINIO, NF-STK-N8N]
-->

# How to self-host Supabase

To self-host Supabase, run Supabase's own Postgres image (`supabase/postgres`) on a persistent volume, generate a JWT secret and sign `anon` and `service_role` keys with it, start Auth, PostgREST, postgres-meta and Studio against that database, and put an API gateway (Kong or Envoy) in front that checks the `apikey` header. A generic Postgres server lacks the roles and schemas these services expect.

This guide does it on Northflank with the CLI: the upstream Docker Compose images pinned to one commit, a secret group with the generated keys, a one-off job that sets service-role passwords, and Kong as the only public port. You end with a public HTTPS API and a password-protected Studio, verified with four `curl` checks. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run.

## TL;DR: how to self-host Supabase

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Pin a commit of `supabase/supabase`, read the image tags from its `docker/docker-compose.yml`, confirm the team and project, then create the `supabase` project.
3. Generate keys with upstream's `utils/generate-keys.sh`, store them in a secret group, and run `supabase/postgres` as a service with a volume.
4. Run a one-off job that sets the role passwords upstream's `roles.sql` sets.
5. Deploy Kong with `kong.yml` as a runtime file, read its public host, then deploy `auth`, `rest`, `meta` and `studio`.
6. Verify `/auth/v1/health`, `/rest/v1/` with the service key, a 401 without a key, and Studio behind basic auth.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup, and how much work it is

Self-hosted Supabase isn't one container. The official `docker/docker-compose.yml` runs Studio, an API gateway, Auth (GoTrue), PostgREST, Realtime, Storage, imgproxy, postgres-meta, Edge Runtime, Postgres and Supavisor, plus optional Logflare and Vector. Supabase's docs ask for at least 4 GB RAM, 2 CPU cores and 40 GB of SSD, and the self-hosted edition is community-supported.

Three things trip people up:

- **The database must be `supabase/postgres`.** Every connection string in the compose file logs in as a role that image creates (`authenticator`, `supabase_auth_admin`, `supabase_admin`), and Auth, Realtime and Storage keep their own schemas in it. A plain managed PostgreSQL, including Northflank's PostgreSQL addon, doesn't have them, so this guide runs the image as a service with a volume.
- **The keys are linked.** `ANON_KEY` and `SERVICE_ROLE_KEY` are HS256 JWTs signed with `JWT_SECRET`. Change the secret and both keys stop working.
- **The gateway is the front door.** Kong checks the `apikey` header, maps it to the `anon` or `admin` group, and serves Studio behind HTTP basic auth. Its config routes to the hostnames `auth`, `rest`, `meta` and `studio`. On Northflank, services reach each other as `[service-name]:[port]`, so this guide uses those names. The database is named `supabase-db`, because service names need at least 3 characters.

**Scope:** this guide deploys the core: database, Auth, REST, postgres-meta, Studio and Kong. It leaves out Realtime, Storage, imgproxy, Edge Functions and Supavisor. Kong returns errors on their routes until you add them. Realtime is the hard one: it takes its tenant ID from the first label of the hostname it's called on, so the compose file names its container `realtime-dev.supabase-realtime` and `kong.yml` routes to that dotted name. A Northflank service name can't contain a dot, so adding Realtime means changing how Kong addresses it, not just deploying the image.

On where to self-host Supabase: you need several long-running containers on a private network, a persistent disk for Postgres and a public HTTPS endpoint. Serverless and static hosts can't run it. A VM with Docker Compose is the upstream path, but then the TLS, backups and upgrades are yours to run. A container platform runs the same images as separate services, with TLS and volumes handled for you.

If you'd rather click through it, Northflank has a [Supabase stack template](https://northflank.com/stacks/deploy-supabase) that also includes Storage (backed by MinIO), Realtime and Edge Functions.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`curl`, `openssl` and `base64`.**
- **You've pinned an upstream commit.** All files and image tags come from one commit of the `docker/` directory:
  ```bash
  SB_REF=$(curl -s "https://api.github.com/repos/supabase/supabase/commits?path=docker&per_page=1" | jq -r '.[0].sha'); echo "$SB_REF"
  RAW="https://raw.githubusercontent.com/supabase/supabase/$SB_REF/docker"
  curl -fsSL "$RAW/docker-compose.yml" -o sb-compose.yml
  curl -fsSL "$RAW/docker-compose.kong.yml" -o sb-kong.yml
  img() { grep -hoE "image: $1:[^ ]+" sb-compose.yml sb-kong.yml | head -n1 | cut -d' ' -f2; }
  PG_IMAGE=$(img supabase/postgres); AUTH_IMAGE=$(img supabase/gotrue); REST_IMAGE=$(img postgrest/postgrest)
  META_IMAGE=$(img supabase/postgres-meta); STUDIO_IMAGE=$(img supabase/studio); KONG_IMAGE=$(img kong/kong)
  echo "$PG_IMAGE $AUTH_IMAGE $REST_IMAGE $META_IMAGE $STUDIO_IMAGE $KONG_IMAGE"
  ```
  At the time of writing this printed `supabase/postgres:17.6.1.136`, `supabase/gotrue:v2.197.0`, `postgrest/postgrest:v14.18`, `supabase/postgres-meta:v0.100.0`, `supabase/studio:2026.11.02-sha-a05e7ec` and `kong/kong:3.9.3`.

Names used below: project `supabase`, secret group `supabase-secrets`, services `supabase-db`, `kong`, `auth`, `rest`, `meta`, `studio`, volume `supabase-db-data`, job `supabase-init`.

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
echo "commit:  $SB_REF"
echo "images:  $PG_IMAGE $AUTH_IMAGE $REST_IMAGE $META_IMAGE $STUDIO_IMAGE $KONG_IMAGE"
northflank get project --projectId supabase -o json >/dev/null 2>&1 \
  && echo "project: supabase exists and will be reused" \
  || echo "project: supabase will be created"
```

If `commit` or any image is empty, re-run the pinning commands from the prerequisites in this shell.

**If you're an agent:** show the user the team, the upstream commit, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project and two helpers

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "supabase", "region": "europe-west", "description": "Self-hosted Supabase" }'
export PROJECT_ID=supabase

wait_svc() {
  for i in $(seq 1 60); do
    D=$(northflank get service --projectId "$PROJECT_ID" --serviceId "$1" --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
    echo "$1=$D"; [ "$D" = COMPLETED ] && return 0; [ "$D" = FAILED ] && return 1; sleep 10
  done; return 1
}
svc() {  # name image port plan env-json
  jq -n --arg n "$1" --arg i "$2" --argjson p "$3" --arg plan "$4" --argjson env "$5" '{
    name: $n, billing: {deploymentPlan: $plan},
    deployment: {instances: 1, external: {imagePath: $i}, docker: {configType: "default"}},
    ports: [{name: "http", internalPort: $p, public: false, protocol: "HTTP"}],
    runtimeEnvironment: $env,
    healthChecks: [{protocol: "TCP", type: "readinessProbe", port: $p, initialDelaySeconds: 10,
      periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 30, successThreshold: 1}]
  }' > "$1.json"
  northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file "$1.json" >/dev/null && echo "put $1"
}
```

The helpers wait for a service's rollout and create one service. `put` creates or updates services, so re-running anything below is safe. The ports are private, and only Kong gets a public port. The probes are TCP probes on each service's port.

## Step 4: Generate the keys and create the secret group

Upstream's `generate-keys.sh` creates `JWT_SECRET`, `ANON_KEY` and `SERVICE_ROLE_KEY` (signed with that secret, valid for five years), `POSTGRES_PASSWORD`, `DASHBOARD_PASSWORD` and `PG_META_CRYPTO_KEY`. Write them to a file only you can read, and don't print it:

```bash
curl -fsSL "$RAW/utils/generate-keys.sh" -o generate-keys.sh
umask 077; sh generate-keys.sh </dev/null > supabase.keys
k() { sed -n "s/^$1=//p" supabase.keys; }

jq -n --arg pw "$(k POSTGRES_PASSWORD)" --arg jwt "$(k JWT_SECRET)" --arg anon "$(k ANON_KEY)" \
      --arg svc "$(k SERVICE_ROLE_KEY)" --arg dash "$(k DASHBOARD_PASSWORD)" --arg crypto "$(k PG_META_CRYPTO_KEY)" '{
  name: "supabase-secrets", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  secrets: { variables: {
    POSTGRES_PASSWORD: $pw, PGPASSWORD: $pw, PG_META_DB_PASSWORD: $pw,
    GOTRUE_DB_DATABASE_URL: ("postgres://supabase_auth_admin:" + $pw + "@supabase-db:5432/postgres"),
    PGRST_DB_URI: ("postgres://authenticator:" + $pw + "@supabase-db:5432/postgres"),
    GOTRUE_JWT_SECRET: $jwt, PGRST_JWT_SECRET: $jwt, AUTH_JWT_SECRET: $jwt,
    SUPABASE_ANON_KEY: $anon, SUPABASE_SERVICE_KEY: $svc,
    PG_META_CRYPTO_KEY: $crypto, CRYPTO_KEY: $crypto,
    DASHBOARD_USERNAME: "supabase", DASHBOARD_PASSWORD: $dash
  } }
}' > supabase-secrets.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file supabase-secrets.json >/dev/null && rm supabase-secrets.json
```

Each service reads the same values under its own variable name, as in the compose file (`GOTRUE_JWT_SECRET`, `PGRST_JWT_SECRET` and so on). The password is hex, so it's safe inside the connection URLs.

**This step needs a human:** move `supabase.keys` into a password manager. Your apps need `ANON_KEY` (and server code `SERVICE_ROLE_KEY`), and losing `JWT_SECRET` means reissuing both.

## Step 5: Run the Supabase Postgres image with a volume

```bash
jq -n --arg i "$PG_IMAGE" '{
  name: "supabase-db", billing: {deploymentPlan: "nf-compute-100-2"},
  deployment: {instances: 1, external: {imagePath: $i},
    docker: {configType: "customCommand", customCommand: "postgres -c config_file=/etc/postgresql/postgresql.conf -c log_min_messages=fatal"}},
  ports: [{name: "pg", internalPort: 5432, public: false, protocol: "TCP"}],
  runtimeEnvironment: {POSTGRES_HOST: "/var/run/postgresql", PGPORT: "5432", POSTGRES_PORT: "5432",
    PGDATABASE: "postgres", POSTGRES_DB: "postgres", JWT_EXP: "3600"},
  healthChecks: [{protocol: "TCP", type: "readinessProbe", port: 5432, initialDelaySeconds: 10,
    periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 30, successThreshold: 1}]
}' > supabase-db.json
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file supabase-db.json >/dev/null

northflank get volume --projectId "$PROJECT_ID" --volumeId supabase-db-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "supabase-db-data",
  "mounts": [
    { "containerMountPath": "/var/lib/postgresql/data", "volumeMountPath": "pgdata" },
    { "containerMountPath": "/etc/postgresql-custom", "volumeMountPath": "config" }
  ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "supabase-db", "type": "service" } ]
}'
wait_svc supabase-db || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

The command and environment come from the compose file's `db` service. One volume holds both paths the compose file persists: the data directory and `/etc/postgresql-custom`, which holds the pgsodium key. Each path gets its own subdirectory, so Postgres never sees the volume root. Attaching the volume redeploys the service, and the image initialises its roles and schemas on that first boot. The read/write volume keeps the database at one instance.

## Step 6: Set the role passwords with a one-off job

Upstream mounts `roles.sql` and `jwt.sql` as init scripts. Instead of mounting files over the image's own init directory, this guide runs the same statements once as a job. It uses the Postgres image (for `psql`) and a runtime file in a fresh path:

```bash
cat > init.sh <<'EOF'
set -e
for r in authenticator pgbouncer supabase_auth_admin supabase_functions_admin supabase_storage_admin; do
  echo "ALTER USER $r WITH PASSWORD :'pw';" | psql -h supabase-db -U supabase_admin -d postgres -v ON_ERROR_STOP=1 -v pw="$POSTGRES_PASSWORD"
done
psql -h supabase-db -U supabase_admin -d postgres -v ON_ERROR_STOP=1 -c "ALTER DATABASE postgres SET \"app.settings.jwt_exp\" TO '3600';"
echo INIT_OK
EOF
jq -n --arg i "$PG_IMAGE" --arg f "$(base64 < init.sh | tr -d '\n')" '{
  name: "supabase-init", billing: {deploymentPlan: "nf-compute-20"},
  settings: {backoffLimit: 0, activeDeadlineSeconds: 600},
  deployment: {external: {imagePath: $i},
    docker: {configType: "customEntrypointCustomCommand", customEntrypoint: "/bin/sh", customCommand: "/nf/init.sh"}},
  runtimeFiles: {"/nf/init.sh": {data: $f, encoding: "utf-8"}}
}' > supabase-init.json
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file supabase-init.json >/dev/null

RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId supabase-init --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 60); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId supabase-init --runId "$RUN_ID" --noDefaults -o json)
  S=$(echo "$R" | jq -r '(.data // .).status'); echo "run=$S"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$S" = "SUCCESS" ] && OK=1; break; }; sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId supabase-init --runId "$RUN_ID" --lineLimit 50
[ "$OK" = 1 ] || { echo "JOB_NOT_SUCCEEDED: stop here; the logs above show why"; false; }
```

`put job` without `settings.cron` makes a manual job. The job inherits `POSTGRES_PASSWORD` and `PGPASSWORD` from the unrestricted secret group. The image sets `supabase_admin`'s password to `POSTGRES_PASSWORD` during its first boot. Expect `SUCCESS` and `INIT_OK`.

## Step 7: Deploy Kong and read the public host

Kong's config comes from upstream's `kong.yml` and `kong-entrypoint.sh`, unchanged. The entrypoint substitutes `$SUPABASE_ANON_KEY`, `$DASHBOARD_PASSWORD` and the other variables at start:

```bash
curl -fsSL "$RAW/volumes/api/kong.yml" -o kong.yml
curl -fsSL "$RAW/volumes/api/kong-entrypoint.sh" -o kong-entrypoint.sh
jq -n --arg i "$KONG_IMAGE" --arg yml "$(base64 < kong.yml | tr -d '\n')" --arg ep "$(base64 < kong-entrypoint.sh | tr -d '\n')" '{
  name: "kong", billing: {deploymentPlan: "nf-compute-50"},
  deployment: {instances: 1, external: {imagePath: $i},
    docker: {configType: "customEntrypoint", customEntrypoint: "/bin/sh /home/kong/kong-entrypoint.sh"}},
  ports: [{name: "http", internalPort: 8000, public: true, protocol: "HTTP"}],
  runtimeEnvironment: {KONG_DATABASE: "off", KONG_DECLARATIVE_CONFIG: "/usr/local/kong/kong.yml",
    KONG_ROUTER_FLAVOR: "expressions", KONG_DNS_ORDER: "LAST,A,CNAME", KONG_DNS_NOT_FOUND_TTL: "1", KONG_DNS_VALID_TTL: "5",
    KONG_PLUGINS: "request-transformer,cors,key-auth,acl,basic-auth,request-termination,ip-restriction,post-function",
    KONG_NGINX_PROXY_PROXY_BUFFER_SIZE: "160k", KONG_NGINX_PROXY_PROXY_BUFFERS: "64 160k",
    KONG_PROXY_ACCESS_LOG: "/dev/stdout combined"},
  runtimeFiles: {"/home/kong/temp.yml": {data: $yml, encoding: "utf-8"},
                 "/home/kong/kong-entrypoint.sh": {data: $ep, encoding: "utf-8"}},
  healthChecks: [{protocol: "TCP", type: "readinessProbe", port: 8000, initialDelaySeconds: 10,
    periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 12, successThreshold: 1}]
}' > kong.json
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file kong.json >/dev/null
OK=0; wait_svc kong && OK=1
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId kong --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId kong --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

The Kong environment matches upstream's `docker-compose.kong.yml`. Its `SUPABASE_*` keys and dashboard credentials come from the secret group. Kong deploys before the services it routes to, so those routes return errors for now. That lets you read the public hostname before you deploy Auth and Studio.

## Step 8: Deploy Auth, PostgREST, postgres-meta and Studio

```bash
URL="https://$HOST"
svc auth "$AUTH_IMAGE" 9999 nf-compute-20 "$(jq -n --arg u "$URL" '{
  GOTRUE_API_HOST: "0.0.0.0", GOTRUE_API_PORT: "9999", GOTRUE_DB_DRIVER: "postgres",
  API_EXTERNAL_URL: ($u + "/auth/v1"), GOTRUE_JWT_ISSUER: ($u + "/auth/v1"), GOTRUE_SITE_URL: $u,
  GOTRUE_URI_ALLOW_LIST: "", GOTRUE_DISABLE_SIGNUP: "false", GOTRUE_JWT_EXP: "3600",
  GOTRUE_JWT_ADMIN_ROLES: "service_role", GOTRUE_JWT_AUD: "authenticated", GOTRUE_JWT_DEFAULT_GROUP_NAME: "authenticated",
  GOTRUE_EXTERNAL_EMAIL_ENABLED: "true", GOTRUE_MAILER_AUTOCONFIRM: "false",
  GOTRUE_EXTERNAL_PHONE_ENABLED: "false", GOTRUE_EXTERNAL_ANONYMOUS_USERS_ENABLED: "false" }')"
svc rest "$REST_IMAGE" 3000 nf-compute-20 '{"PGRST_DB_SCHEMAS": "public,graphql_public", "PGRST_DB_ANON_ROLE": "anon",
  "PGRST_DB_MAX_ROWS": "1000", "PGRST_DB_EXTRA_SEARCH_PATH": "public", "PGRST_DB_USE_LEGACY_GUCS": "false",
  "PGRST_APP_SETTINGS_JWT_EXP": "3600"}'
svc meta "$META_IMAGE" 8080 nf-compute-20 '{"PG_META_PORT": "8080", "PG_META_DB_HOST": "supabase-db",
  "PG_META_DB_PORT": "5432", "PG_META_DB_NAME": "postgres", "PG_META_DB_USER": "postgres"}'
svc studio "$STUDIO_IMAGE" 3000 nf-compute-100-1 "$(jq -n --arg u "$URL" '{
  HOSTNAME: "0.0.0.0", STUDIO_PG_META_URL: "http://meta:8080", SUPABASE_URL: "http://kong:8000", SUPABASE_PUBLIC_URL: $u,
  POSTGRES_HOST: "supabase-db", POSTGRES_PORT: "5432", POSTGRES_DB: "postgres", POSTGRES_USER_READ_WRITE: "postgres",
  PGRST_DB_SCHEMAS: "public,graphql_public", DEFAULT_ORGANIZATION_NAME: "Default Organization",
  DEFAULT_PROJECT_NAME: "Default Project", ENABLED_FEATURES_LOGS_ALL: "false" }')"
for s in auth rest meta studio; do wait_svc "$s" || echo "CHECK_$s"; done
```

These environments are the compose file's, with hostnames swapped for Northflank service names. Secrets come from the group, so none appear in these bodies. `GOTRUE_MAILER_AUTOCONFIRM=false` means email sign-ups need SMTP (`GOTRUE_SMTP_HOST`, `GOTRUE_SMTP_PORT`, `GOTRUE_SMTP_USER`, `GOTRUE_SMTP_PASS`, `GOTRUE_SMTP_ADMIN_EMAIL`). Add those to `auth` before real users sign up.

## How to know it worked

Load the keys without echoing them (`ANON=$(k ANON_KEY); SVC=$(k SERVICE_ROLE_KEY); DASH=$(k DASHBOARD_PASSWORD)`). You're done when all of these hold:

1. Auth answers through Kong:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/auth/v1/health" -H "apikey: $ANON"   # 200
   ```
2. PostgREST connects as `authenticator` (this proves the init job worked). The OpenAPI root is admin-only in `kong.yml`:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/rest/v1/" -H "apikey: $SVC"   # 200
   ```
3. The gateway rejects requests with no key:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/rest/v1/"   # 401
   ```
4. Studio loads behind basic auth (and returns 401 without it):
   ```bash
   curl -sL -o /dev/null -w '%{http_code}\n' -u "supabase:$DASH" "https://$HOST/"   # 200
   ```

A Supabase client then connects with `createClient("https://<HOST>", "<ANON_KEY>")`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `supabase-db` never ready, logs mention `initdb` or permissions | Volume mount issue | Check `get volume` shows both mounts; see [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) |
| Init job fails with `password authentication failed for user "supabase_admin"` | Database first booted with a different `POSTGRES_PASSWORD` | Put back the original password; on a fresh install, delete and recreate the volume |
| `/rest/v1/` returns 503 or PostgREST logs auth errors | Init job didn't run or failed | Re-run step 6 and read its logs |
| Kong returns `name resolution failed` | Upstream service not deployed or not named as in `kong.yml` | Service names must be `auth`, `rest`, `meta`, `studio` |
| Kong crashes on start reading `/home/kong/temp.yml` | Runtime file not mounted or not readable | Check `runtimeFiles` with `get service`; report a permission error to the user |
| `Invalid API key` / 401 with a key | Key signed with a different `JWT_SECRET` | Regenerate all three together and update the group, then restart every service |
| Auth email links point to `localhost` | `API_EXTERNAL_URL` or `GOTRUE_SITE_URL` not set from `$HOST` | Re-run `svc auth` in step 8 |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

- Confirm the target from step 2 with the user before any write.
- If a wait in step 5, 7 or 8 prints `ROLLOUT_NOT_COMPLETE` or `CHECK_<service>`, read that service's logs (`northflank get service logs --projectId "$PROJECT_ID" --serviceId <service>`) before changing anything.
- Never print `supabase.keys`, `JWT_SECRET`, `SERVICE_ROLE_KEY`, `POSTGRES_PASSWORD` or the secret group. `ANON_KEY` is meant for browsers, but treat it as config, not something to log.
- Never regenerate `JWT_SECRET` or `POSTGRES_PASSWORD` on a running install. The first invalidates every key and session. The second breaks the database's role passwords until the init job runs again.
- Keep the upstream commit (`SB_REF`) with the deployment, and take image tags and `kong.yml` from the same commit when upgrading.
- Get explicit confirmation before deleting `supabase-db`, its volume or the project, and back up the database before an upgrade.
- If the user needs Realtime or Storage, say plainly that this guide's core setup doesn't include them, and point to the stack template or the upstream compose file.

## FAQ

### Can I use a managed PostgreSQL database instead of `supabase/postgres`?
Not with the upstream compose setup. The services log in as roles the `supabase/postgres` image creates, and they rely on its schemas and extensions. Supabase's Docker guide only covers the bundled database. Run the image as a service with a volume, as this guide does.

### Kong or Envoy?
Upstream now uses Envoy as the default gateway and still ships Kong as an override (`docker-compose.kong.yml`). Both route the same paths and check the same keys. This guide uses Kong because its config is two files: `kong.yml` and a small entrypoint.

### What about the new publishable and secret keys?
Upstream's `utils/add-new-auth-keys.sh` creates `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_SECRET_KEY` and an asymmetric key pair. `kong-entrypoint.sh` uses them when they're set. This guide uses the legacy HS256 `ANON_KEY` and `SERVICE_ROLE_KEY`, which `kong.yml` still accepts.

### Where should I host self-hosted Supabase?
Anywhere that can run about six long-running containers on a private network, with a persistent disk for Postgres and HTTPS on the gateway, and at least the 4 GB RAM and 2 cores Supabase lists. A VM with Docker Compose is the upstream route. A container platform runs the same images as separate services and handles TLS and volumes. Serverless hosts can't run it at all.

## Conclusion

You now have Supabase's core running on Northflank. It uses Supabase's own Postgres image on a volume, keys generated with upstream's script, Auth, PostgREST, postgres-meta and Studio on private ports, and Kong as the single public HTTPS entry point.

**Next:** configure SMTP for Auth, [link a custom domain](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) to Kong's port (then re-run step 8 with that hostname), and add Storage from the compose file if you need file uploads. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
- [How to deploy PocketBase](deploy-pocketbase.md)
- [How to self-host n8n](self-host-n8n.md)
