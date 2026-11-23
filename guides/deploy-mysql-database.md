<!--
agent-guide-meta:
  id: NF-STK-MYSQL
  description: "How to deploy a MySQL database as a managed instance with TLS, inject its credentials into your app through a secret group and verify it with SELECT VERSION()."
  sources:
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-mysql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/configure-addons-for-high-availability
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/databases-and-persistence/access-a-database
    - https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data
    - https://northflank.com/docs/v1/application/databases-and-persistence/migrate-data-to-northflank/migrate-your-mysql-database-to-northflank
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
    - https://dev.mysql.com/doc/refman/8.4/en/connecting-using-uri-or-key-value-pairs.html
  related: [NF-STK-MONGODB, NF-DB-001, NF-DEP-009, NF-DEP-002]
-->

# How to deploy a MySQL database

To deploy a MySQL database, run it with persistent storage and TLS on a private network next to your app, choose the version, default database name and failover mode up front, and pass the host, port, database, user and password to the app as environment variables such as `DB_HOST` and `DB_PASSWORD`. Changing TLS or replication mode after data exists usually means migrating, so decide them before you create it.

This guide does it on Northflank with the CLI: a managed MySQL addon created from one reviewed JSON file with TLS on, its connection details linked into your service through a restricted secret group, and a restart. You end with a private database your app reads, verified by `SELECT VERSION();` through a local tunnel without printing a password. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to deploy a MySQL database

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project and database name, and check versions with the public `https://api.northflank.com/v1/addon-types`, before changing anything.
3. Create the project, then the database with `northflank put addon` (`type: "mysql"`, `tlsEnabled: true`), and poll `northflank get addon` until `status` is `running`.
4. Link `HOST`, `PORT`, `DATABASE`, `USERNAME`, `PASSWORD` (or `MYSQL_CONNECTOR_URI`) into a restricted secret group with `addonDependencies`, then restart the app.
5. Verify through `northflank forward addon --skipHostnames` with `mysql … -e 'SELECT VERSION();'`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Where to deploy MySQL, and why a managed addon

A MySQL server needs a long-running process, a persistent volume, backups, TLS and a private network path to your app. The usual places to run it are a **managed MySQL database** (the provider operates the server, storage and credentials), a **self-run container** such as the official `mysql` Docker image on a container platform or Kubernetes, or a **VM** where you install and operate MySQL yourself. The further down that list you go, the more control you get and the more operations work you take on; static and serverless hosts don't run MySQL at all, they connect to one of these.

Running `mysql:8` in a container yourself means you own the volume, backups, TLS, upgrades and password handling. On Northflank, MySQL is an **addon**: a managed database inside a project, reachable by default only from services and jobs in that project, with dump backups, read replicas and an optional failover mode.

MySQL has more create-time decisions than most databases. **TLS can't be changed after creation**, and neither can HA mode or the database name, so writing them into a reviewed JSON file beats clicking through a form.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The `mysql` client**, for the connection test in step 8.

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

Before changing anything, print the deployment target. The database goes in the project `my-app`; if your app already has a project, use its name instead throughout, so the app reaches it over the private network.

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId my-app -o json >/dev/null 2>&1 \
  && echo "project: my-app exists and will be reused" \
  || echo "project: my-app will be created"
northflank get addon --projectId my-app --addonId app-mysql -o json >/dev/null 2>&1 \
  && echo "addon:   app-mysql already exists: pick another name" \
  || echo "addon:   app-mysql is free"
curl -s https://api.northflank.com/v1/addon-types \
  | jq '.data.addonTypes[] | select(.type=="mysql") | {versions, majors, storage: .resources.storage.options, replicas: .resources.replicas.options}'
```

- If the addon name is taken, pick another and use it everywhere `app-mysql` appears. `put addon` in step 5 would otherwise change the existing one.
- The addon-types endpoint is public. At the time of writing it listed versions `9.7.2`, `9.6.0`, `8.4.11`, `8.4.9`, `8.0.46`, majors `9` and `8`, storage from 4096 MB, and 1–3 replicas. List plan IDs with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[].id'`.
- Agree on size with whoever owns the bill. Storage can only be increased later, and the plan's memory caps connections (see the FAQ).
- Find which variables the app reads. Laravel reads `DB_CONNECTION`, `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`. WordPress's official image reads `WORDPRESS_DB_HOST`, `WORDPRESS_DB_USER`, `WORDPRESS_DB_PASSWORD`, `WORDPRESS_DB_NAME`. ORMs like Prisma read one URL such as `DATABASE_URL`. Grep the code.

**If you're an agent:** show the user the team, the project (new or reused), the region (`europe-west` below, unless they choose another), the database's name, version, plan, storage and replicas. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "my-app", "region": "europe-west" }'
export PROJECT_ID=my-app
```

REST: `POST https://api.northflank.com/v1/projects` with the same body. For REST calls, also export an API token as `NF_API_TOKEN`.

## Step 4: Write the MySQL definition

Save this as `mysql.json`:

```json
{
  "name": "app-mysql",
  "description": "Primary MySQL for the app",
  "type": "mysql",
  "version": "8-latest",
  "tlsEnabled": true,
  "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "app" }
}
```

What each choice means, from the create-addon schema and the MySQL docs:

- `version` accepts `latest`, a `<major>-latest` pin such as `8-latest`, or an exact version from the list. Never pick a lower major than a database you'll import from.
- **`tlsEnabled` can't be changed after creation.** Public access later requires TLS, so leave it `true` unless your client can't do TLS.
- `customCredentials.dbName` sets the default database name and **can't be changed**. Leave it out to get a random name.
- `name` is 3–39 characters starting with a letter; the addon ID is its slug (`app-mysql`). `storage` is in MB.
- `replicas` (1, 2 or 3) above 1 adds **read replicas** and a `HOST_READ` connection detail. If the primary goes down, writes stop until it returns. The docs conflict on scaling back down: the scale-a-database page says standard MySQL replicas can be reduced (MySQL HA can't), while the MySQL page says replicas can't be decreased once increased.
- **High availability** is a separate mode: `"typeSpecificSettings": { "mysqlHaModeEnabled": true }` deploys an InnoDB Cluster (three nodes with Group Replication, MySQL Router and automated failover; tune with `mysqlHaRouterEnabled` and `mysqlRouterReplicas`). It **can't be changed after creation**, and every table must have a primary key.

## Step 5: Create MySQL and wait until it's running

```bash
northflank put addon --projectId "$PROJECT_ID" --file mysql.json -o json | jq '(.data // .) | {id, status}'
```

REST:

```bash
curl -sS -X PUT "https://api.northflank.com/v1/projects/$PROJECT_ID/addons" \
  -H "Authorization: Bearer $NF_API_TOKEN" -H "Content-Type: application/json" --data @mysql.json
```

`put` creates or updates; use `northflank create addon` if you'd rather get a 409 when the name is taken. Then poll every 10 seconds, stopping on a failure state:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get addon --projectId "$PROJECT_ID" --addonId app-mysql -o json | jq -r '(.data // .).status')
  echo "status=$S"
  case "$S" in
    running) OK=1; break ;;
    failed|error|errorAllocating|deleting|deleted) echo "provisioning failed"; break ;;
  esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

## Step 6: Inspect the connection details without printing them

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-mysql -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

The MySQL docs list these connection details:

| Detail | Use |
|---|---|
| `HOST`, `PORT` (3306), `DATABASE`, `USERNAME`, `PASSWORD` | Framework config (Laravel, WordPress, Django) |
| `MYSQL_CONNECTOR_URI` | One-string clients; format `[scheme://][user:password@][host][:port][/schema][...options]` |
| `MYSQL_JDBC_URI` | Java / JDBC |
| `CONNECT_COMMAND` | The `mysql` command-line client |
| `HOST_READ` | Read replica host, when replicas > 1 |
| `ADMIN_USERNAME`, `ADMIN_PASSWORD`, keys ending `_ADMIN` | Maintenance only |

The exact casing in the JSON response isn't confirmed, so treat the key list as the source of truth. Never link admin keys into application workloads.

## Step 7: Link MySQL into your app with a secret group

A secret group with `addonDependencies` injects the addon's details into every service or job it applies to, under aliases you choose. Save this as `mysql-secrets.json` (Laravel names shown; swap the aliases for what your app reads):

```json
{
  "name": "app-mysql-connection",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": true, "nfObjects": [ { "id": "<SERVICE_ID>", "type": "service" } ] },
  "addonDependencies": [
    {
      "addonId": "app-mysql",
      "keys": [
        { "keyName": "HOST", "aliases": ["DB_HOST"] },
        { "keyName": "PORT", "aliases": ["DB_PORT"] },
        { "keyName": "DATABASE", "aliases": ["DB_DATABASE"] },
        { "keyName": "USERNAME", "aliases": ["DB_USERNAME"] },
        { "keyName": "PASSWORD", "aliases": ["DB_PASSWORD"] },
        { "keyName": "MYSQL_CONNECTOR_URI", "aliases": ["DATABASE_URL"] }
      ]
    }
  ],
  "secrets": { "variables": { "DB_CONNECTION": "mysql" } }
}
```

```bash
northflank put secret --projectId "$PROJECT_ID" --file mysql-secrets.json -o json
northflank get secret-link --projectId "$PROJECT_ID" --secretId app-mysql-connection --addonId app-mysql -o json \
  | jq '(.data // .) | {linked, linkedKeys, availableKeys}'
```

`secretType: "environment"` injects at runtime only, not into builds. If `linkedKeys` is missing a key, use the exact names from `availableKeys` and re-link with `northflank update secret-link --projectId "$PROJECT_ID" --secretId app-mysql-connection --addonId app-mysql --input '{"keys":[…]}'`.

Restricting to a service that doesn't exist yet hasn't been confirmed to work. If the app isn't deployed, deploy it first, or create the group with `"restrictions": { "restricted": false }` and tighten it afterwards.

Restart the app so it picks up the new variables, and wait for `COMPLETED` (rolled out and serving):

```bash
northflank restart service --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> -o json
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> -o json | jq -r '(.data // .).status.deployment.status')
  echo "deployment=$D"; [ "$D" = "COMPLETED" ] && { OK=1; break; }; [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }; sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

**TLS in the app.** With TLS on, the app's image needs CA certificates. Debian/Ubuntu images: `apt-get install -y ca-certificates`; Alpine: `apk add --no-cache ca-certificates`. Laravel can pass the CA path through `MYSQL_ATTR_SSL_CA`.

## Step 8: Prove MySQL accepts connections

Open a tunnel without sudo and read the local address from the log rather than assuming `127.0.0.1:3306`:

```bash
northflank forward addon --projectId "$PROJECT_ID" --addonId app-mysql \
  --skipHostnames --portConflict random > /tmp/nf-mysql-fwd.log 2>&1 &
FWD_PID=$!
sleep 10; cat /tmp/nf-mysql-fwd.log
```

Load credentials into variables without printing them, then query the version. `MYSQL_PWD` keeps the password off the command line:

```bash
CREDS=$(northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-mysql -o json)
pick() { echo "$CREDS" | jq -r --arg k "$1" '(.data // .) | (.secrets + .envs) | to_entries[] | select(.key|ascii_downcase==$k) | .value' | head -1; }
DB_USER=$(pick username); DB_NAME=$(pick database)
MYSQL_PWD="$(pick password)" mysql -h <LOCAL_IP> -P <LOCAL_PORT> -u "$DB_USER" --ssl-mode=REQUIRED "$DB_NAME" \
  -N -e 'SELECT VERSION();'
unset CREDS; kill $FWD_PID
```

`--ssl-mode=REQUIRED` encrypts without checking the hostname, which is what you want against a loopback IP. Use `--ssl-mode=DISABLED` only if you created the addon without TLS.

## How to know it worked

1. `northflank get addon --projectId "$PROJECT_ID" --addonId app-mysql -o json | jq -r '(.data // .) | .status, .spec.type'` prints `running` and `mysql`.
2. The `mysql` query exits 0 and prints a version starting with the major you asked for (for example `8.4.`).
3. The service has the variables (names only, no values):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> \
     --shell-cmd 'sh -c' --cmd '"env | cut -d= -f1 | grep -E ^DB_ | sort"'
   ```
   `exec` drops output without a TTY, hence `script`. On Linux use `script -qefc '…' /dev/null`.
4. No connection errors in the app logs since the restart:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> --duration 300 --textIncludes "SQLSTATE"
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Can't enable public access | Created without TLS, which is permanent | Create a new addon with TLS and migrate the data |
| `Too many connections` | Plan memory caps connections | Move to a bigger plan or pool connections |
| InnoDB Cluster rejects writes to a table | Table has no primary key | Add primary keys; Group Replication requires them |
| TLS handshake or certificate error in the app | Image lacks CA certificates | Install `ca-certificates`, or mount the CA as a secret file |
| `DB_HOST` missing in the container | Service not in `restrictions`, or no restart | Fix `nfObjects`; restart |
| Status stuck in `allocating` | Capacity or plan issue | `northflank get addon logs --projectId "$PROJECT_ID" --addonId app-mysql`; don't delete without asking |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 5 loop prints `ADDON_NOT_READY`, or the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Confirm TLS, `dbName`, HA mode and the storage size with the user before step 5. The first three are permanent and storage can't shrink.
- Print key names, never values. Don't echo `DATABASE_URL`, `CONNECT_COMMAND` or passwords into logs, commits or PR text.
- Link the regular user into apps. Keep `_ADMIN` credentials for maintenance.
- Never run `DROP DATABASE` or `northflank delete addon` without explicit confirmation for that specific database and a recent backup. Prefer `northflank pause addon` when the goal is to stop paying for compute.

## FAQ

### Where should I deploy my MySQL database?
For most apps, a managed MySQL database is the simplest production choice: storage, backups, TLS and credentials are handled, and the database stays on a private network next to your app. Running the `mysql` Docker image yourself on a container platform or Kubernetes, or installing MySQL on a VM, gives more control over configuration but makes backups, upgrades and failover your job. Whatever you pick, put it in the same region as the app.

### How many connections does MySQL allow?
It depends on the plan's memory. The docs list 32 connections at 512 MB, 96 at 1 GB, 250 at 4 GB, 350 at 8 GB, 700 at 16 GB, 1,000 at 32 GB and 2,000 above that.

### Should I use HA mode or read replicas?
HA mode (InnoDB Cluster) gives automated write failover but needs a primary key on every table and can't be turned off. Standard MySQL with extra replicas only adds read capacity; writes pause while the primary is down. See [configure addons for high availability](https://northflank.com/docs/v1/application/databases-and-persistence/configure-addons-for-high-availability).

### How do I bring data over from another MySQL server?
Addons support importing a dump or a live database by connection string. Follow [migrate your MySQL database to Northflank](https://northflank.com/docs/v1/application/databases-and-persistence/migrate-data-to-northflank/migrate-your-mysql-database-to-northflank), and use the same or a newer major version.

### How do I back it up?
MySQL supports native dump and disk snapshot backups, on demand or on a schedule. See [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data).

## Conclusion

You now have a managed MySQL database with its permanent settings chosen on purpose, its connection details injected into only the app that needs them, and a version query that proved it works without printing a password.

**Next:** schedule backups, then deploy the app that uses it, such as [a Laravel app with a queue worker](deploy-laravel-project.md) or [WordPress](deploy-wordpress-website.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Laravel project](deploy-laravel-project.md)
- [How to deploy a WordPress website](deploy-wordpress-website.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy MongoDB](deploy-mongodb.md)
