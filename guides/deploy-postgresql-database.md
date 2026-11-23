<!--
agent-guide-meta:
  id: NF-DB-001
  description: "How to deploy a PostgreSQL database as a managed instance from one JSON definition, poll until it runs, read credentials safely and verify with psql."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/create-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-a-database
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/access-a-database
  related: [NF-DEP-004, NF-DEP-006, NF-DEP-007, NF-DB-003, NF-DEP-012]
-->

# How to deploy a PostgreSQL database

To deploy a PostgreSQL database for production, use a managed instance with persistent storage and TLS, pin a major version, keep it on a private network with no public address, and give your app the generated `postgresql://` connection URI through an environment variable such as `DATABASE_URL`. Most managed services let storage grow but never shrink, so start small.

This guide does it on Northflank with the CLI, with a REST equivalent for each call: one JSON definition creates the database, a loop polls until it's `running`, and the credentials are read without printing secrets. You end with a running PostgreSQL instance you've connected to through a local tunnel, confirmed by `psql` returning `SELECT version()`. Every step is a non-interactive command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end.

## TL;DR: how to deploy a PostgreSQL database

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team and project, and check that the database name is free, before changing anything.
3. Create the project, then the database from a JSON definition (type `postgresql`, version, plan, storage) with `northflank create addon --file postgres.json`.
4. Poll `northflank get addon` until `status` is `running`, then read the credentials, printing **key names only**.
5. Verify with `northflank forward addon --skipHostnames` and `psql -c 'SELECT version();'`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Where to deploy PostgreSQL, and why script it

PostgreSQL needs a long-running process, persistent storage that survives restarts and redeploys, backups, and a private network path to your app. You have three usual places to run it: a **managed database** service, where the provider runs the process, storage, upgrades and credentials; a **self-run container** (the official `postgres` image on a container platform or Kubernetes, with a persistent volume), where backups and upgrades are yours; or a **VM** where you install and operate PostgreSQL yourself. Static and serverless hosts don't run databases; your app on them connects to one of these. This guide uses a managed database.

Clicking through a dashboard works once. It doesn't work when you need the same database in staging, in every preview environment, or when an AI coding agent is setting up infrastructure on your behalf. A scripted setup gives you:

- **Repeatability**: the same definition creates the same database every time.
- **A clear "done" signal**: you poll for a status instead of eyeballing a dashboard.
- **No leaked secrets**: you control exactly what gets printed, so passwords never land in terminal history, chat logs or commits.

On Northflank, a managed database is an **addon** of type `postgresql` that lives inside a project.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`psql`**, the PostgreSQL client, for the connection test in step 8.

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

Before changing anything, print the deployment target. The database goes in the project `my-app`; if your app already has a project, use its name instead throughout, so the app reaches the database over the private network.

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId my-app -o json >/dev/null 2>&1 \
  && echo "project: my-app exists and will be reused" \
  || echo "project: my-app will be created"
northflank get addon --projectId my-app --addonId app-db -o json >/dev/null 2>&1 \
  && echo "addon:   app-db already exists: pick another name" \
  || echo "addon:   app-db is free"
northflank get addon types -o json \
  | jq -c '(.data // .).addonTypes[] | select(.type == "postgresql") | .versions'
```

- If the addon name is taken, pick another and use it everywhere `app-db` appears.
- The last command lists the PostgreSQL versions you can pin.
- Run `northflank list plans -o json` to pick a `deploymentPlan` ID; step 4 uses `nf-compute-20`.
- Agree on plan, storage and replicas with whoever owns the bill. **Storage can only be increased later, never decreased**, so start small. The docs disagree on replicas: the scale-a-database page says PostgreSQL replicas can be reduced, but the PostgreSQL page says they can't be decreased once increased, so treat an increase as possibly permanent.

**If you're an agent:** show the user the team, the project (new or reused), the region (`europe-west` below, unless they choose another), the database name, version, plan and storage. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "my-app", "region": "europe-west" }'
export PROJECT_ID=my-app
```

For REST calls, also export an API token as `NF_API_TOKEN`.

## Step 4: Write the database definition

Save this as `postgres.json`. `version` accepts `latest` or a pinned major such as `16-latest`.

```json
{
  "name": "app-db",
  "description": "Primary Postgres for the app",
  "type": "postgresql",
  "version": "16-latest",
  "tlsEnabled": true,
  "externalAccessEnabled": false,
  "billing": {
    "deploymentPlan": "nf-compute-20",
    "storage": 6144,
    "replicas": 1
  },
  "customCredentials": { "dbName": "app" }
}
```

A few things to know about these fields, all from the create-addon schema:

- `name` must be 3–39 characters and start with a letter. The addon ID becomes its slug (`app-db`).
- `storage` is in **megabytes** and must be one of the sizes in `resources.storage.options` for `postgresql` on the addon-types endpoint (4096 MB upward; 6144 MB is the default).
- `tlsEnabled: true` makes internal connections use TLS. It must be `true` if you ever turn on `externalAccessEnabled`.
- `customCredentials.dbName` sets the default database name and **can't be changed after creation**. Leave it out to get a random name.
- `typeSpecificSettings.postgresqlWalLevel` (`replica` or `logical`) and `postgresqlSupabaseMode` are also fixed at creation. Only set them if you need them.
- You can add a backup schedule now with `backupSchedules`. See the [backup, restore and import docs](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data).

## Step 5: Create the database

```bash
northflank create addon --projectId "$PROJECT_ID" --file postgres.json -o json
```

Or with the REST API:

```bash
curl -sS -X POST "https://api.northflank.com/v1/projects/$PROJECT_ID/addons" \
  -H "Authorization: Bearer $NF_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data @postgres.json
```

Note the `id` in the response. Over REST it's at `.data.id`; the CLI prints the object without the `data` wrapper.

## Step 6: Wait until it's running

Provisioning "may take a few minutes, after which its status will change to running", according to the docs. The full `status` enum is `preDeployment`, `triggerAllocation`, `allocating`, `postDeployment`, `running`, `paused`, `scaling`, `upgrading`, `resetting`, `backup`, `restore`, `failed`, `error`, `errorAllocating`, `deleting` and `deleted`.

This loop waits for `running` and stops early on a failure state:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get addon --projectId "$PROJECT_ID" --addonId app-db -o json | jq -r '(.data // .).status')
  echo "status=$S"
  case "$S" in
    running) OK=1; break ;;
    failed|error|errorAllocating|deleting|deleted) echo "provisioning failed"; break ;;
  esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

Over REST, poll `GET https://api.northflank.com/v1/projects/my-app/addons/app-db` and read `.data.status`. Polling every 10 seconds keeps you well inside the API limit of 1,000 requests per hour.

## Step 7: Read the credentials without printing secrets

`get addon credentials` returns an object with two maps, `secrets` and `envs`, whose keys depend on the addon type. Start by printing **key names only**:

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-db -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

REST: `GET https://api.northflank.com/v1/projects/my-app/addons/app-db/credentials`.

According to the PostgreSQL docs, the generated connection details include:
- `POSTGRES_URI`, in the format `postgresql://user:password@host:port/database`.
- `JDBC_POSTGRES_URI`.
- `HOST`, `PORT`, `USERNAME`, `PASSWORD` and `DATABASE`.
- Admin variants ending in `_ADMIN`.

The exact casing inside the JSON isn't confirmed (the API example shows lowercase `username` and `password`). Rather than guess a key name, find the connection URI by its value:

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-db -o json \
  | jq -r '(.data // .) | (.secrets + .envs) | to_entries[]
           | select((.value|type)=="string" and (.value|startswith("postgresql://")))
           | .key'
```

Give applications the regular (non-admin) URI. Save admin credentials for maintenance.

## Step 8: Prove the database accepts connections

The database has no public address by default, so tunnel to it with the CLI. Plain `northflank forward addon` edits your local hostnames and needs `sudo`. If you're running without sudo (most agent sandboxes), add `--skipHostnames`, which exposes the database on a loopback IP and may pick a different local port.

```bash
northflank forward addon --projectId "$PROJECT_ID" --addonId app-db \
  --skipHostnames --portConflict random > /tmp/nf-forward.log 2>&1 &
FWD_PID=$!
sleep 10; cat /tmp/nf-forward.log   # note the local IP:port for 5432
```

Read the log for the actual address; don't assume `127.0.0.1:5432`. Then connect with `psql`, passing the password through the environment rather than the command line:

```bash
export PGPASSWORD='<PASSWORD_FROM_CREDENTIALS>'
psql "host=<LOCAL_IP> port=<LOCAL_PORT> user=<USERNAME> dbname=<DATABASE> sslmode=prefer" \
  -tAc 'SELECT version();'
unset PGPASSWORD
kill $FWD_PID
```

If TLS verification fails with `sslmode=prefer`, retry with `sslmode=require`.

## How to know it worked

You're done when all four of these hold:

1. `northflank get addon --projectId "$PROJECT_ID" --addonId app-db -o json | jq -r '(.data // .).status'` prints `running`.
2. `(.data // .).spec.type` is `postgresql`, and `.spec.config.versionTag` starts with the major version you asked for.
3. The credentials check in step 7 finds at least one value starting with `postgresql://`.
4. `psql … -tAc 'SELECT version();'` exits with code 0 and prints a line starting with `PostgreSQL`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Create fails validation on `name` | Too short/long or starts with a digit | Use 3–39 chars starting with a letter |
| Create rejects `deploymentPlan` or `storage` | Plan ID or size not allowed | Re-check `northflank list plans -o json` and the storage options in `northflank get addon types -o json` |
| Status stuck in `allocating` for 15+ minutes | Capacity or plan issue | Check `northflank get addon containers --projectId "$PROJECT_ID" --addonId app-db -o json`; don't delete without asking |
| Status `failed` or `errorAllocating` | Provisioning failed | Read `northflank get addon logs --projectId "$PROJECT_ID" --addonId app-db` |
| `forward addon` asks for a password | Hostname setup needs sudo | Add `--skipHostnames` |
| `psql: connection refused` | Tunnel not running or wrong port | Check `/tmp/nf-forward.log`; the addon must be `running` |
| App can't reach the DB during build | Builds run on separate infrastructure | Avoid DB access at build time |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 6 loop prints `ADDON_NOT_READY`, read the addon logs before changing anything.
- Print key names, never values. Keep passwords and URIs out of logs, commits and PR descriptions.
- Confirm `dbName`, WAL level and Supabase mode with the user before step 5. They're permanent, and storage can't be reduced.
- Don't enable `externalAccessEnabled` unless asked. If you do, add `ipPolicies` with `ALLOW` rules for known addresses.
- Never run `northflank delete addon` without explicit confirmation for that specific database and a recent backup. Deleting an addon destroys its data.

## FAQ

### Where should I deploy my PostgreSQL database?
For a production database, a managed PostgreSQL service is the lowest-effort choice: you get persistent storage, TLS, backups and generated credentials without operating the server. Run the `postgres` Docker image yourself on a container platform or Kubernetes only if you're ready to own volumes, backups and upgrades, and pick a VM when you need full control of the OS and configuration. Keep the database in the same region and private network as the app that uses it.

### Can I create the database with the API instead of the CLI?
Yes. Every CLI command here maps to a REST endpoint under `https://api.northflank.com/v1/`, authenticated with `Authorization: Bearer <token>`. The request body is the same JSON.

### How do I give my app the connection string?
Link the database to a secret group and expose the URI under an alias like `DATABASE_URL`. The `addonDependencies` entry maps the addon's `POSTGRES_URI` key to that alias:

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "app-db-env",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "addonDependencies": [
    { "addonId": "app-db", "keys": [ { "keyName": "POSTGRES_URI", "aliases": ["DATABASE_URL"] } ] }
  ]
}'
```

Restart the services that read it afterwards. See [connect database secrets to workloads](https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads).

### Can I make the database bigger later?
Yes. You can increase storage, but you can't decrease it, so start small. You can also add replicas. Whether you can reduce them again is unclear: the scale-a-database docs say PostgreSQL replicas can be reduced, while the PostgreSQL docs page says they can't be decreased once increased.

### How many connections does PostgreSQL allow?
Up to 750 concurrent connections. For many short-lived clients, enable the connection pooler (`typeSpecificSettings.postgresqlConnectionPoolerEnabled`).

## Conclusion

You now have a managed PostgreSQL database created from a single JSON definition, with a reliable status check and a connection test that never prints a secret. The same steps work from a laptop, a CI pipeline or a coding agent, which is the point.

**Next:** deploy the app that uses it, for example [a Node.js API with PostgreSQL](deploy-nodejs-application.md), then [set up scheduled backups](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data) before you go to production. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a Django project](deploy-django-project.md)
- [How to deploy a FastAPI app](deploy-fastapi-app.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a full stack app](deploy-full-stack-app.md)
