<!--
agent-guide-meta:
  id: NF-STK-MATTERMOST
  description: "How to deploy Mattermost in production with the official Docker image, managed PostgreSQL, a persistent volume for data and config, and a public HTTPS site URL."
  sources:
    - https://northflank.com/stacks/deploy-mattermost
    - https://docs.mattermost.com/deployment-guide/server/containers/install-docker.html
    - https://docs.mattermost.com/deployment-guide/software-hardware-requirements.html
    - https://docs.mattermost.com/administration-guide/configure/configuration-settings.html
    - https://github.com/mattermost/docker/blob/main/env.example
    - https://github.com/mattermost/docker/blob/main/docker-compose.yml
    - https://github.com/mattermost/mattermost/blob/master/server/build/Dockerfile
    - https://github.com/mattermost/mattermost/blob/master/server/channels/api4/system.go
    - https://hub.docker.com/r/mattermost/mattermost-team-edition/tags
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/secure/inject-secrets
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DB-001, NF-DEP-002, NF-STK-N8N, NF-STK-GITLAB]
-->

# How to deploy Mattermost

To deploy Mattermost, run the official `mattermost/mattermost-team-edition` (or enterprise-edition) image on port 8065 against PostgreSQL with `MM_SQLSETTINGS_DRIVERNAME=postgres` and a `postgres://` `MM_SQLSETTINGS_DATASOURCE`, persist `/mattermost/config`, `data`, `logs` and `plugins`, and set `MM_SERVICESETTINGS_SITEURL` to the public HTTPS address. A wrong site URL breaks links, emails and plugins; a missing volume loses every uploaded file.

This guide does it on Northflank with the CLI: managed PostgreSQL linked through a secret group, the official image, one volume mounted at all five Mattermost paths, and the site URL set from the assigned hostname. You end with a public HTTPS Mattermost whose health endpoint reports the database and file store as `OK`. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human creates the first admin account.

## TL;DR: how to deploy Mattermost

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Pick an edition and version, and confirm the team, project and region before changing anything.
3. Create a project, PostgreSQL and a secret group that links the database's connection details as `MM_DB_*`.
4. Run `mattermost/mattermost-team-edition:<version>` on port 8065 and attach one volume at `/mattermost/config`, `data`, `logs`, `plugins` and `client/plugins`.
5. Set `MM_SERVICESETTINGS_SITEURL` to the public host, restart, and create the first admin account.
6. Verify that `/api/v4/system/ping?get_server_status=true` reports `database_status` and `filestore_status` as `OK` and the client config shows your `https://` site URL.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Mattermost's official production layout, the `mattermost/docker` repository, is two containers and some folders: PostgreSQL, the Mattermost server, and five host directories mounted into it (`config`, `data`, `logs`, `plugins`, `client/plugins`), all owned by uid 2000. Settings come from `MM_*` environment variables, which take precedence over `config.json` and are greyed out in the System Console. This guide keeps that shape:

- **PostgreSQL** holds users, channels and posts. Mattermost v12 requires PostgreSQL 15 or later, and MySQL support is deprecated from v11, so start on PostgreSQL.
- **One volume, five mount paths.** Uploaded files live in `/mattermost/data`, and installed plugins in the two plugin folders. Without persistent storage they disappear on every restart.
- **`MM_SERVICESETTINGS_SITEURL`** must be the public `https://` address, or links, notifications and some plugins point at the wrong place.

Where to deploy Mattermost comes down to those needs: an always-on server process with WebSockets (clients keep a live connection), a PostgreSQL database, persistent disk, and a stable HTTPS hostname. Serverless and static hosts can't hold long-lived WebSocket connections or local files. A VM with the official Docker Compose setup works, but you run NGINX, TLS certificates, backups and upgrades yourself. A container platform with managed PostgreSQL and HTTPS ports covers those pieces and still runs the official image.

If you'd rather click through it, Northflank has a [Mattermost stack template](https://northflank.com/stacks/deploy-mattermost) that deploys a similar setup from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

Names used below: project `mattermost`, addon `mattermost-db`, secret group `mattermost-env`, service `mattermost`, volume `mattermost-data`.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Pick a version and confirm where it will deploy

Before changing anything, pin a Mattermost version and print the deployment target. `mattermost-team-edition` is the open-source build; the official Compose setup defaults to `mattermost-enterprise-edition`, which you'd pick if you plan to add a license. Both take the same settings. This lists the newest plain version tag:

```bash
MM_VERSION=$(curl -s "https://hub.docker.com/v2/repositories/mattermost/mattermost-team-edition/tags?page_size=100&ordering=last_updated" \
  | jq -r '.results[].name' | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -n1); echo "version: $MM_VERSION"

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId mattermost -o json >/dev/null 2>&1 \
  && echo "project: mattermost exists and will be reused" \
  || echo "project: mattermost will be created"
```

- For an Extended Support Release instead, pick the newest tag of the ESR line from the Mattermost release notes.
- Mattermost's requirements list 1 vCPU and 2 GB RAM for up to 1,000 users and 2 vCPU / 4 GB up to 2,000, so the service starts on `nf-compute-100-2` (1 vCPU, 2 GB) and `nf-compute-200-4` is the next step. List plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | "\(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'`.
- If the project is reused and already has a `mattermost` service, addon or volume, stop and ask the user: the steps below change them.

**If you're an agent:** show the user the team, the edition and version, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create a project

A project holds your services and addons. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "mattermost", "region": "europe-west" }'
export PROJECT_ID=mattermost
```

## Step 4: Create PostgreSQL and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "mattermost-db", "type": "postgresql", "version": "17-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 12288, "replicas": 1 },
  "customCredentials": { "dbName": "mattermost" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId mattermost-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

The database only accepts traffic from inside the project, so TLS is off and the connection string uses `sslmode=disable`, the same as Mattermost's own Compose example. If you enable TLS, change that to `sslmode=require` in step 6. `dbName` is fixed at creation and storage can only be increased. For a large team, raise the addon plan from `nf-compute-20` before you invite everyone.

## Step 5: Create the secret group

Mattermost reads one `postgres://` URL in `MM_SQLSETTINGS_DATASOURCE`. The documented `POSTGRES_URI` format has no `sslmode` parameter, and without one Mattermost's Postgres driver may try TLS, so this guide links the individual connection details and builds the URL in step 6. First print the credential **key names** (never the values):

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId mattermost-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

The PostgreSQL docs list `HOST`, `PORT`, `USERNAME`, `PASSWORD` and `DATABASE`. If the output uses different casing, change the `keyName` values below to match.

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "mattermost-env", "type": "secret", "secretType": "environment", "priority": 10,
  "restrictions": { "restricted": false },
  "addonDependencies": [ { "addonId": "mattermost-db", "keys": [
    { "keyName": "HOST",     "aliases": ["MM_DB_HOST"] },
    { "keyName": "PORT",     "aliases": ["MM_DB_PORT"] },
    { "keyName": "DATABASE", "aliases": ["MM_DB_NAME"] },
    { "keyName": "USERNAME", "aliases": ["MM_DB_USER"] },
    { "keyName": "PASSWORD", "aliases": ["MM_DB_PASSWORD"] }
  ] } ],
  "secrets": { "variables": { "MM_SQLSETTINGS_DRIVERNAME": "postgres" } }
}'
```

## Step 6: Deploy the Mattermost service

Write `mattermost.json`. Replace `<MM_VERSION>` with the tag from step 2. The `${MM_DB_*}` references are Northflank [dynamic templating](https://northflank.com/docs/v1/application/secure/inject-secrets): they're filled in from the secret group when the container starts, so keep them literal (don't let your shell expand them).

```json
{
  "name": "mattermost",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "mattermost/mattermost-team-edition:<MM_VERSION>" },
    "docker": { "configType": "default" }
  },
  "ports": [ { "name": "http", "internalPort": 8065, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": {
    "TZ": "UTC",
    "MM_SQLSETTINGS_DATASOURCE": "postgres://${MM_DB_USER}:${MM_DB_PASSWORD}@${MM_DB_HOST}:${MM_DB_PORT}/${MM_DB_NAME}?sslmode=disable&connect_timeout=10",
    "MM_BLEVESETTINGS_INDEXDIR": "/mattermost/data/bleve-indexes"
  },
  "healthChecks": [
    { "protocol": "HTTP", "type": "startupProbe", "path": "/api/v4/system/ping", "port": 8065,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 30, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/api/v4/system/ping", "port": 8065,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/api/v4/system/ping", "port": 8065,
      "initialDelaySeconds": 30, "periodSeconds": 20, "timeoutSeconds": 5, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file mattermost.json
```

Mattermost runs its database migrations on start, so the first boot can take a few minutes; the startup probe allows five. `/api/v4/system/ping` answers once the server is listening. Public HTTP ports on Northflank carry WebSockets, which Mattermost clients use for live updates, so no extra proxy is needed. The Bleve index directory is optional search indexing kept on the data volume.

## Step 7: Attach one volume at the Mattermost paths

The volume's `volumeMountPath` lets one volume back several directories, matching the five folders in the official Compose file:

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId mattermost-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "mattermost-data",
  "mounts": [
    { "containerMountPath": "/mattermost/config",         "volumeMountPath": "config" },
    { "containerMountPath": "/mattermost/data",           "volumeMountPath": "data" },
    { "containerMountPath": "/mattermost/logs",           "volumeMountPath": "logs" },
    { "containerMountPath": "/mattermost/plugins",        "volumeMountPath": "plugins" },
    { "containerMountPath": "/mattermost/client/plugins", "volumeMountPath": "client-plugins" }
  ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 12288 },
  "attachedObjects": [ { "id": "mattermost", "type": "service" } ]
}'
```

`create volume` returns 409 if the volume exists, hence the `get` guard. Attaching redeploys the service. On first start Mattermost writes a default `config.json` into the empty config folder, as it does in the Compose setup. Size the volume for your uploads; it can grow but not shrink. With a single read/write volume the service runs one instance, and restarts have a few seconds of downtime.

## Step 8: Set the site URL

Read the hostname Northflank assigned to the public port. Then send the complete `runtimeEnvironment` map, because the patch replaces it, and restart:

```bash
for i in $(seq 1 30); do
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId mattermost --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId mattermost --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns // empty' | head -n1)
  [ -n "$HOST" ] && break; sleep 10
done; [ -n "$HOST" ] && echo "$HOST" || { echo "NO_PUBLIC_HOST: stop here and do not run the patch below"; false; }

northflank patch service deployment --projectId "$PROJECT_ID" --serviceId mattermost --noDefaults -o json --input "$(jq -n --arg h "$HOST" '{
  runtimeEnvironment: {
    TZ: "UTC",
    MM_SQLSETTINGS_DATASOURCE: "postgres://${MM_DB_USER}:${MM_DB_PASSWORD}@${MM_DB_HOST}:${MM_DB_PORT}/${MM_DB_NAME}?sslmode=disable&connect_timeout=10",
    MM_BLEVESETTINGS_INDEXDIR: "/mattermost/data/bleve-indexes",
    MM_SERVICESETTINGS_SITEURL: ("https://" + $h)
  }
}')"
northflank restart service --projectId "$PROJECT_ID" --serviceId mattermost --noDefaults -o json
```

The jq program is single-quoted, so `${MM_DB_*}` stays literal for Northflank to resolve. When you add a custom domain, repeat this step with that hostname.

## Step 9: Wait for the rollout and get your URL

```bash
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId mattermost --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
[ "$OK" = 1 ] && echo "https://$HOST"
```

`COMPLETED` means rolled out and serving. If it prints `ROLLOUT_NOT_COMPLETE`, read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId mattermost`.

**This step needs a human:** open `https://$HOST` right away and create the first account. On a new server, the first account created becomes the system admin, so don't leave the URL unclaimed.

## How to know it worked

You're done when all of these hold:

1. The extended health check writes to the database and tests the file store:
   ```bash
   curl -s "https://$HOST/api/v4/system/ping?get_server_status=true" | jq '{status, database_status, filestore_status}'
   ```
   Expect `"OK"` for all three.
2. The public client config shows your HTTPS site URL:
   ```bash
   curl -s "https://$HOST/api/v4/config/client?format=old" | jq -r '.SiteURL'
   ```
   Expect `https://<HOST>`.
3. The volume is attached to `mattermost`:
   ```bash
   northflank get volume --projectId "$PROJECT_ID" --volumeId mattermost-data --noDefaults -o json | jq '(.data // .) | {status, attachedObjects}'
   ```
4. Optional: the WebSocket endpoint upgrades (expect `101`):
   ```bash
   curl -s --http1.1 -o /dev/null -w '%{http_code}\n' -m 5 -H 'Connection: Upgrade' -H 'Upgrade: websocket' \
     -H 'Sec-WebSocket-Version: 13' -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' "https://$HOST/api/v4/websocket"
   ```
5. After the admin exists, upload a file to a channel, restart the service, and confirm the file is still there.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Logs show `dial tcp` errors to `localhost:5432` or an empty host | `MM_DB_*` not injected, or templates not resolved | Check key names from step 5; if `${MM_DB_HOST}` appears literally in the logs, put the full datasource in the secret group instead (see below) |
| Logs show an SSL/TLS error connecting to Postgres | `sslmode` mismatch | Keep `sslmode=disable` with TLS off, or `sslmode=require` with TLS on |
| Password authentication fails though the host is right | Password contains URL-special characters | Report it; the datasource needs the password URL-encoded |
| `permission denied` on `/mattermost/config` or `/mattermost/data` | Volume not owned by uid 2000 | Restart once; if it persists, report it. See [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) |
| Links or emails point at `localhost:8065` | Step 8 not applied, or not restarted | Re-run step 8 |
| Startup probe fails on first boot | Migrations slower than five minutes | Raise `failureThreshold` on the startup probe |
| Can't scale to 2 instances | Single read/write volume; clustering needs an Enterprise license | Scale the plan up instead |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

Fallback for unresolved templates: build the datasource locally from `northflank get addon credentials` with `jq`, write it straight into the secret group as `MM_SQLSETTINGS_DATASOURCE` without echoing it, and drop it from `runtimeEnvironment`. You'll need to repeat that if the database password changes.

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write: team, edition and version, region and whether the project is new or reused.
- Don't report success until checks 1 to 3 pass.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the deployment logs before changing anything.
- Never print database passwords, the datasource or the credentials response. Print key names only.
- Hand the first-account step to the user immediately after the first rollout; the first account is the system admin.
- Don't `exec` into the container for checks. Recent Mattermost images may have no shell; the HTTP checks above cover the database, file store and site URL.
- Get explicit confirmation before deleting the addon, the volume or the service, and back up the database before a major-version upgrade.

## FAQ

### Do I need PostgreSQL, or can I use MySQL?
Use PostgreSQL. Mattermost's requirements page lists PostgreSQL 15 or later for v12 and says MySQL support is being deprecated starting with v11. MariaDB isn't supported.

### Team Edition or Enterprise Edition image?
`mattermost-team-edition` is the open-source build. `mattermost-enterprise-edition` is the image the official Compose setup uses and the one that accepts a license for paid features such as clustering. Switching later is an image change; both read the same database and volume.

### Can I store files in S3 instead of the volume?
Yes. Set `MM_FILESETTINGS_DRIVERNAME=amazons3` and the matching `AmazonS3*` file settings from the Mattermost configuration docs, pointed at a bucket such as a [MinIO addon](deploy-minio.md). Keep the volume for config and plugins.

### How do I upgrade Mattermost?
Back up the database, then change the tag with `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId mattermost --input '{"deployment":{"instances":1,"external":{"imagePath":"mattermost/mattermost-team-edition:<NEW_VERSION>"}}}'` and wait for `COMPLETED`. Read the release notes for the versions you skip; migrations run on start.

### Where should I deploy Mattermost?
Anywhere that keeps the official image running as an always-on container with WebSockets, next to PostgreSQL and persistent disk, behind a public HTTPS URL. That rules out serverless functions and static hosts. A VM with Docker Compose works if you're happy to run NGINX, TLS and backups yourself; a container platform with managed PostgreSQL, like the setup in this guide, takes those off your hands.

## Conclusion

You now have Mattermost on the official image, backed by managed PostgreSQL, with config, uploads, logs and plugins on a persistent volume, and a site URL that matches its public HTTPS address.

**Next:** add a custom domain by [linking a domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) (then repeat step 8 with that hostname), and move the database to a larger plan as your team grows. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to self-host GitLab](self-host-gitlab.md)
- [How to self-host n8n](self-host-n8n.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
