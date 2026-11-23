<!--
agent-guide-meta:
  id: NF-STK-GRAFANA
  description: "How to deploy Grafana with the official Docker image, PostgreSQL as its database, a secret admin password, root_url and a provisioned Prometheus data source."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://grafana.com/docs/grafana/latest/setup-grafana/configure-docker/
    - https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/
    - https://grafana.com/docs/grafana/latest/administration/provisioning/
    - https://grafana.com/docs/grafana/latest/datasources/prometheus/
    - https://github.com/grafana/grafana/blob/main/Dockerfile
    - https://github.com/grafana/grafana/blob/main/pkg/services/sqlstore/database_config.go
    - https://hub.docker.com/r/grafana/grafana/tags
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/secure/upload-secret-files
    - https://northflank.com/docs/v1/application/secure/inject-secrets
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-STK-PROMETHEUS, NF-DB-001, NF-DEP-002, NF-STK-KEYCLOAK]
-->

# How to deploy Grafana

To deploy Grafana in production, run the official `grafana/grafana` Docker image with PostgreSQL as its database (`GF_DATABASE_TYPE=postgres` plus host, name, user and password), set the admin password and `secret_key` from secrets, set `GF_SERVER_ROOT_URL` to the public HTTPS address, and provision data sources from YAML in `/etc/grafana/provisioning/datasources`. The default SQLite file disappears with the container, and a wrong `root_url` breaks redirects and links.

This guide does it on Northflank with the CLI: a managed PostgreSQL addon, a secret group with the database credentials, admin password and secret key, and a Grafana service with a Prometheus data source mounted as a file. You end with Grafana on a public HTTPS URL, verified by health and data source checks. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to deploy Grafana

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team and project, and pin a Grafana version, before changing anything.
3. Create a project and a PostgreSQL addon, then a secret group that links the database as `GF_DATABASE_*` and stores a generated admin password and secret key.
4. Run `grafana/grafana:<version>` on port 3000 with `northflank put service deployment`, with a Prometheus data source passed as a runtime file.
5. Set `GF_SERVER_ROOT_URL` from the public host and wait for the rollout.
6. Check that `/api/health` reports `"database": "ok"`, the admin login works and the data source exists.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Grafana's defaults suit a laptop. The configuration docs say SQLite is the default database and "isn't recommended for production", and in a container it lives on a filesystem that's replaced on every deploy. So this setup changes four things:

- **PostgreSQL** holds users, dashboards, folders and alert rules. Grafana reads `GF_DATABASE_TYPE`, `GF_DATABASE_HOST`, `GF_DATABASE_NAME`, `GF_DATABASE_USER` and `GF_DATABASE_PASSWORD` (the `GF_<SECTION>_<KEY>` override pattern). With dashboards in Postgres, Grafana itself needs no volume.
- **`GF_SECURITY_ADMIN_PASSWORD`** replaces the default `admin`/`admin`. The docs note it's applied once, on first start, so later changes go through the UI or API.
- **`GF_SECURITY_SECRET_KEY`** signs and encrypts data source secrets. The docs say it can't be changed without re-entering those secrets, so it's generated once and kept.
- **`GF_SERVER_ROOT_URL`** is "the full URL used to access Grafana from a web browser". Login redirects, OAuth callbacks and alert links use it.

Data sources come from provisioning files, so they're in version control rather than clicked in by hand.

Where to host Grafana: it's a long-running web server with a database, so it needs a host that keeps a container running next to PostgreSQL behind HTTPS, and ideally on the same private network as Prometheus and your other metrics sources. Serverless functions don't fit a stateful dashboard server. A VM works if you manage the proxy, TLS and database backups; Kubernetes works with Helm but needs a cluster. A container platform with managed PostgreSQL and private networking covers it with one image.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl` and `base64`**, to generate secrets and encode the data source file.

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

Before changing anything, print the target and pin a Grafana version. If Prometheus already runs in a project, use that project's ID instead of `monitoring`, so Grafana can reach it privately.

```bash
export PROJECT_ID=monitoring
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId "$PROJECT_ID" -o json >/dev/null 2>&1 \
  && echo "project: $PROJECT_ID exists and will be reused" \
  || echo "project: $PROJECT_ID will be created"
northflank get service --projectId "$PROJECT_ID" --serviceId prometheus -o json >/dev/null 2>&1 \
  && echo "prometheus: found in $PROJECT_ID" || echo "prometheus: not in $PROJECT_ID (check 4 will fail until it is)"
GRAFANA_VERSION=$(curl -s "https://hub.docker.com/v2/repositories/grafana/grafana/tags?page_size=100&ordering=last_updated" \
  | jq -r '.results[].name' | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -n1); echo "grafana: $GRAFANA_VERSION"
```

- The version line lists the newest plain version tag from Docker Hub (13.x at the time of writing). If it prints nothing, set `GRAFANA_VERSION` by hand.
- Prometheus is optional. Grafana deploys and passes its health, database and login checks without it. The provisioned data source points at `http://prometheus:9090` and only reports healthy once a Prometheus service by that name runs in the same project; until then, only check 4 in "How to know it worked" fails. Point the `url` in step 7 elsewhere if your Prometheus lives at another address.

Names used below: addon `grafana-db`, secret group `grafana-env`, service `grafana`. Prometheus, if any, is assumed to be the service `prometheus` on port 9090 in the same project.

**If you're an agent:** show the user the team, the project, the region (`europe-west` below, unless they choose another), whether the project is new or reused, and the Grafana version. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. If you chose another project name in step 2, use it here. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "monitoring", "region": "europe-west" }'
export PROJECT_ID=monitoring
```

## Step 4: Create PostgreSQL and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "grafana-db", "type": "postgresql", "version": "16-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "grafana" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId grafana-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

The database only accepts traffic from inside the project. TLS is off so Grafana's default `ssl_mode` of `disable` works; if you turn TLS on, set `GF_DATABASE_SSL_MODE=require`. TLS and `dbName` are fixed at creation, and storage can only be increased.

## Step 5: Check the credential key names

Print the key **names** (never the values):

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId grafana-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

The PostgreSQL docs list `HOST`, `PORT`, `USERNAME`, `PASSWORD` and `DATABASE`. If the casing differs, change the `keyName` values in step 6 to match.

## Step 6: Create the secret group

Grafana reads `host` as `host:port` and, according to its source, defaults to port 5432 when none is given, which is the addon's port. Generate the admin password and secret key without echoing them:

```bash
GF_ADMIN_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=')
jq -n --arg pw "$GF_ADMIN_PASSWORD" --arg sk "$(openssl rand -hex 32)" '{
  name: "grafana-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "grafana-db", keys: [
    { keyName: "HOST",     aliases: ["GF_DATABASE_HOST"] },
    { keyName: "DATABASE", aliases: ["GF_DATABASE_NAME"] },
    { keyName: "USERNAME", aliases: ["GF_DATABASE_USER"] },
    { keyName: "PASSWORD", aliases: ["GF_DATABASE_PASSWORD"] }
  ] } ],
  secrets: { variables: {
    GF_DATABASE_TYPE: "postgres", GF_DATABASE_SSL_MODE: "disable",
    GF_SECURITY_ADMIN_USER: "admin", GF_SECURITY_ADMIN_PASSWORD: $pw, GF_SECURITY_SECRET_KEY: $sk
  } }
}' > grafana-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file grafana-env.json >/dev/null && rm grafana-env.json
```

Keep `GF_ADMIN_PASSWORD` in your shell for the checks below. **Tell the user to copy the admin password from the secret group in the UI into their password manager.**

## Step 7: Deploy Grafana with a provisioned data source

Save the data source as `prometheus-datasource.yaml`. It uses the private address `prometheus:9090`, so queries never leave the project. Don't put `$` in this file: Grafana interpolates `$VAR` in provisioning files, and Northflank substitutes `${VAR}` templates in secret files.

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

Build the service definition with the file embedded as base64:

```bash
DS_B64=$(base64 < prometheus-datasource.yaml | tr -d '\n')
jq -n --arg img "grafana/grafana:$GRAFANA_VERSION" --arg ds "$DS_B64" '{
  name: "grafana",
  billing: { deploymentPlan: "nf-compute-50" },
  deployment: { instances: 1, external: { imagePath: $img }, docker: { configType: "default" } },
  ports: [ { name: "http", internalPort: 3000, public: true, protocol: "HTTP" } ],
  runtimeEnvironment: { GF_USERS_ALLOW_SIGN_UP: "false" },
  runtimeFiles: { "/etc/grafana/provisioning/datasources/prometheus.yaml": { data: $ds, encoding: "utf-8" } },
  healthChecks: [
    { protocol: "HTTP", type: "readinessProbe", path: "/api/health", port: 3000,
      initialDelaySeconds: 10, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 6, successThreshold: 1 },
    { protocol: "HTTP", type: "livenessProbe", path: "/api/health", port: 3000,
      initialDelaySeconds: 60, periodSeconds: 20, timeoutSeconds: 3, failureThreshold: 3 }
  ]
}' > grafana.json
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file grafana.json
```

Notes:

- `/etc/grafana/provisioning` is the image's `GF_PATHS_PROVISIONING`, and the image creates the `datasources` folder. Add more files (dashboards providers, alerting) the same way.
- Grafana runs its schema migrations against Postgres on start. On first start it creates the `admin` user with the password from the secret group.
- `nf-compute-50` (0.5 vCPU, 1 GB) is a starting point; size up for many users or heavy dashboards. Several instances can share the one database.
- To install plugins, add `GF_PLUGINS_PREINSTALL` (comma-separated plugin IDs) to `runtimeEnvironment`. Without a volume they're downloaded on each start.

## Step 8: Set `root_url` from the public host

Read the hostname Northflank assigned to the public port, then patch the complete `runtimeEnvironment` map (send every key, since the map may be replaced) and restart:

```bash
for i in $(seq 1 30); do
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId grafana --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId grafana --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  [ -n "$HOST" ] && break; sleep 10
done; [ -n "$HOST" ] && echo "$HOST" || { echo "NO_PUBLIC_HOST: stop here and do not run the patch below"; false; }

northflank patch service deployment --projectId "$PROJECT_ID" --serviceId grafana --noDefaults -o json --input "$(jq -n --arg h "$HOST" '{
  runtimeEnvironment: { GF_USERS_ALLOW_SIGN_UP: "false", GF_SERVER_ROOT_URL: ("https://" + $h + "/") }
}')"
northflank restart service --projectId "$PROJECT_ID" --serviceId grafana --noDefaults -o json
```

When you add a custom domain, repeat this step with that hostname.

## Step 9: Wait for the rollout

```bash
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId grafana --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

`COMPLETED` means rolled out and serving. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId grafana`.

## How to know it worked

You're done when all of these hold:

1. Grafana is healthy and connected to its database:
   ```bash
   curl -s "https://$HOST/api/health" | jq '{database, version}'    # "database": "ok"
   ```
2. Grafana is on PostgreSQL. Only safe values are printed (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId grafana \
     --shell-cmd 'sh -c' --cmd '"printenv GF_DATABASE_TYPE GF_SERVER_ROOT_URL"'
   ```
   Expect `postgres` and `https://<HOST>/`.
3. The admin password works and the provisioned data source exists:
   ```bash
   curl -s -u "admin:$GF_ADMIN_PASSWORD" "https://$HOST/api/datasources/uid/prometheus" | jq '{name, type, url}'
   ```
4. Grafana can query Prometheus (needs Prometheus running in the same project):
   ```bash
   curl -s -u "admin:$GF_ADMIN_PASSWORD" "https://$HOST/api/datasources/uid/prometheus/health" | jq '{status, message}'   # "OK"
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Logs show `database is locked` or a SQLite path | `GF_DATABASE_*` not injected | Check key names from step 5; restart |
| `connection refused` to `127.0.0.1:5432` | `GF_DATABASE_HOST` empty | Same as above |
| `SSL is not enabled on the server` | `ssl_mode` mismatch with the addon | Set `GF_DATABASE_SSL_MODE` to `disable` (TLS off) or `require` (TLS on) |
| Login redirects to `localhost:3000` | `GF_SERVER_ROOT_URL` not set | Redo step 8 |
| Admin password from the group doesn't work | Admin was created on an earlier start with another password | Reset with `grafana cli admin reset-admin-password` through `exec`, or in the UI |
| Data source missing, logs show `permission denied` on provisioning | Mounted file not readable by UID 472 | Report it; as a fallback create the data source through the HTTP API |
| Data source health fails with `no such host` | Prometheus isn't in this project or isn't named `prometheus` | Fix the `url`, or [enable multi-project networking](https://northflank.com/docs/v1/application/network/enable-multi-project-networking) |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- Don't report success until checks 1 to 3 pass (and check 4 when Prometheus runs in the project).
- If a wait loop prints `ROLLOUT_NOT_COMPLETE` or `ADDON_NOT_READY`, read the logs before changing anything.
- Never print `GF_SECURITY_ADMIN_PASSWORD`, `GF_SECURITY_SECRET_KEY`, database credentials or the credentials response. Print key names only.
- Never change `GF_SECURITY_SECRET_KEY` on a running instance; stored data source secrets become unreadable.
- Keep `GF_USERS_ALLOW_SIGN_UP=false` unless the user asks otherwise.
- Back up the database before a Grafana major-version upgrade. Get explicit confirmation before deleting the addon or service.

## FAQ

### Where should I deploy Grafana?
On a host that runs the Grafana container permanently with PostgreSQL next to it, HTTPS in front, and private network access to Prometheus and your other data sources. A VM or Kubernetes cluster works if you'll run TLS and the database yourself; a container platform with managed PostgreSQL, like the setup here, handles both.

### Do I need a volume for `/var/lib/grafana`?
Not with PostgreSQL. Dashboards, users and alert rules live in the database. A volume only helps if you install plugins by hand or use features that write to the data directory.

### Can I provision dashboards too?
Yes. Add a dashboard provider file under `/etc/grafana/provisioning/dashboards` and the dashboard JSON as further `runtimeFiles`, following the provisioning docs.

### How do I deploy Grafana and Prometheus together?
Deploy Prometheus first with [how to deploy Prometheus](deploy-prometheus.md), keep it private, and point Grafana at `http://prometheus:9090` as above.

### How do I upgrade Grafana?
Back up the database, then change the image tag with `northflank patch service deployment` and wait for `COMPLETED`. Grafana migrates the schema on start.

## Conclusion

You now have Grafana on the official image, backed by managed PostgreSQL, with a secret admin password, a fixed secret key, the correct root URL and a Prometheus data source provisioned from a file.

**Next:** add your own domain by [linking a domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) (then repeat step 8), and add SSO such as Keycloak. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy Prometheus](deploy-prometheus.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Keycloak](deploy-keycloak.md)
- [How to deploy a Docker image](deploy-docker-image.md)
