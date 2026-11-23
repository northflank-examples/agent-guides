<!--
agent-guide-meta:
  id: NF-STK-MONGODB
  description: "How to deploy MongoDB as a managed replica set, link its connection string into your app as MONGODB_URI through a secret group and verify it with mongosh."
  sources:
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-mongodb-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/configure-addons-for-high-availability
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/databases-and-persistence/access-a-database
    - https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data
    - https://northflank.com/docs/v1/application/databases-and-persistence/migrate-data-to-northflank/migrate-your-mongodb-database-to-northflank
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
    - https://www.mongodb.com/docs/manual/reference/connection-string/
    - https://www.mongodb.com/docs/manual/core/read-preference/
  related: [NF-STK-MYSQL, NF-DB-001, NF-DB-003, NF-DEP-004]
-->

# How to deploy MongoDB

To deploy MongoDB, run it with persistent storage, authentication and TLS, use a three-node replica set in production so one node can fail without downtime, keep it private to your app's network, and pass the full connection string to the app as `MONGODB_URI`. Don't rebuild the URI from parts: it carries the `authSource`, replica set and TLS options, and dropping them is a common cause of authentication failures.

This guide does it on Northflank with the CLI: a managed MongoDB addon created from one JSON file, its connection string linked into your service as `MONGODB_URI` through a restricted secret group, and a restart. You end with a private database your app reads, verified by `mongosh` running `db.version()` through a local tunnel without printing a password. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to deploy MongoDB

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project and database name, and check versions with the public `https://api.northflank.com/v1/addon-types`, before changing anything.
3. Create the project, then the database with `northflank put addon` (`type: "mongodb"`), and poll `northflank get addon` until `status` is `running`.
4. Link `MONGO_SRV` into a restricted secret group under the alias `MONGODB_URI`, then restart the app.
5. Verify through `northflank forward addon --skipHostnames` with `mongosh --eval 'db.version()'`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Where to deploy MongoDB, and why a managed addon

MongoDB needs a long-running process, persistent storage, authentication, TLS and, for production, a replica set so one node can fail without downtime. You can get that from a **managed MongoDB database** (the provider runs the nodes, storage and credentials), a **self-run container** such as the official `mongo` Docker image on a container platform or Kubernetes, or **VMs** where you install and operate MongoDB yourself. Self-running gives more control, but replica set setup, backups and upgrades become your job; static and serverless hosts don't run MongoDB, they connect to one of these.

A self-run `mongo` container leaves you to set up the replica set, authentication, TLS, backups and upgrades. On Northflank, MongoDB is an **addon**: a managed database inside a project, reachable by default only from services and jobs in that project. You choose 1, 3 or 5 replicas; with 3 or 5 you get a replica set with automated failover. Native dump backups, imports and forks are built in, and the generated connection string already carries the replica set, auth source and TLS options.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`mongosh`**, the MongoDB shell, for the connection test in step 8.

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
northflank get addon --projectId my-app --addonId app-mongo -o json >/dev/null 2>&1 \
  && echo "addon:   app-mongo already exists: pick another name" \
  || echo "addon:   app-mongo is free"
curl -s https://api.northflank.com/v1/addon-types \
  | jq '.data.addonTypes[] | select(.type=="mongodb") | {versions, majors, storage: .resources.storage.options, replicas: .resources.replicas.options}'
```

- If the addon name is taken, pick another and use it everywhere `app-mongo` appears. `put addon` in step 5 would otherwise change the existing one.
- The addon-types endpoint is public. At the time of writing it listed versions from `8.0.37` down to `4.2.21`, majors `8.0` to `4.2`, storage from 4096 MB, and replicas `1`, `3` or `5`. List plan IDs with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[].id'`.
- Agree on size with whoever owns the bill. Storage can only be increased later. The docs conflict on replicas: the MongoDB page says they can't be decreased, while the scale-a-database page says MongoDB replicas can be reduced, so treat an increase as possibly permanent.
- Find which variable the app reads. Mongoose and Payload apps usually read `MONGODB_URI`; some read `MONGO_URL` or `DATABASE_URI`. Grep the code.

**If you're an agent:** show the user the team, the project (new or reused), the region (`europe-west` below, unless they choose another), the database's name, version, plan, storage and replicas. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "my-app", "region": "europe-west" }'
export PROJECT_ID=my-app
```

REST: `POST https://api.northflank.com/v1/projects` with the same body. For REST calls, also export an API token as `NF_API_TOKEN`.

## Step 4: Write the MongoDB definition

Save this as `mongo.json`:

```json
{
  "name": "app-mongo",
  "description": "Primary MongoDB for the app",
  "type": "mongodb",
  "version": "8.0-latest",
  "tlsEnabled": true,
  "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "app" }
}
```

What each choice means, from the create-addon schema and the MongoDB docs:

- `version` accepts `latest`, a `<major>-latest` pin such as `8.0-latest`, or an exact version. Match or exceed the major of any database you'll import from.
- `customCredentials.dbName` sets the default database and **can't be changed after creation**. Leave it out to get a random name. Most apps use the database named in the URI, so this is the name your data lands in.
- `tlsEnabled` can be changed later for MongoDB, but public access requires it. Leave it on.
- `replicas: 3` (or 5) creates a replica set: the primary handles reads and writes, secondaries replicate it, and a secondary is promoted if the primary fails.
- `name` is 3–39 characters starting with a letter; the addon ID is its slug (`app-mongo`). `storage` is in MB and must be one of the sizes in `resources.storage.options`; `replicas` must be 1, 3 or 5.

## Step 5: Create MongoDB and wait until it's running

```bash
northflank put addon --projectId "$PROJECT_ID" --file mongo.json -o json | jq '(.data // .) | {id, status}'
```

REST:

```bash
curl -sS -X PUT "https://api.northflank.com/v1/projects/$PROJECT_ID/addons" \
  -H "Authorization: Bearer $NF_API_TOKEN" -H "Content-Type: application/json" --data @mongo.json
```

`put` creates or updates; `northflank create addon` returns 409 if the name is taken. Poll every 10 seconds:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get addon --projectId "$PROJECT_ID" --addonId app-mongo -o json | jq -r '(.data // .).status')
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
northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-mongo -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

Per the MongoDB docs, you get a connection string in the format `mongodb://[username:password@][host][:port][/database][...options]` (the docs call it `mongoSrv`; templates link it as `MONGO_SRV`), plus `HOST`, `PORT` (27017), `DATABASE`, `USERNAME`, `PASSWORD`, and admin variants (`ADMIN_USERNAME`, `ADMIN_PASSWORD`, keys ending `_ADMIN`). To find which key holds the URI without printing it:

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-mongo -o json \
  | jq -r '(.data // .) | (.secrets + .envs) | to_entries[]
           | select((.value|type)=="string" and (.value|startswith("mongodb"))) | .key'
```

Link the regular URI into apps, not the `_ADMIN` one.

## Step 7: Link MongoDB into your app with a secret group

Save this as `mongo-secrets.json`:

```json
{
  "name": "app-mongo-connection",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": true, "nfObjects": [ { "id": "<SERVICE_ID>", "type": "service" } ] },
  "addonDependencies": [
    { "addonId": "app-mongo", "keys": [ { "keyName": "MONGO_SRV", "aliases": ["MONGODB_URI"] } ] }
  ],
  "secrets": { "variables": {} }
}
```

```bash
northflank put secret --projectId "$PROJECT_ID" --file mongo-secrets.json -o json
northflank get secret-link --projectId "$PROJECT_ID" --secretId app-mongo-connection --addonId app-mongo -o json \
  | jq '(.data // .) | {linked, linkedKeys, availableKeys}'
```

If `linkedKeys` is empty, take the URI key's exact name from `availableKeys` (or the step 6 output) and re-link with `northflank update secret-link --projectId "$PROJECT_ID" --secretId app-mongo-connection --addonId app-mongo --input '{"keys":[{"keyName":"<KEY>","aliases":["MONGODB_URI"]}]}'`.

Restricting to a service that doesn't exist yet hasn't been confirmed to work. If the app isn't deployed, deploy it first, or start with `"restrictions": { "restricted": false }` and tighten later.

Restart the app and wait for `COMPLETED` (rolled out and serving):

```bash
northflank restart service --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> -o json
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> -o json | jq -r '(.data // .).status.deployment.status')
  echo "deployment=$D"; [ "$D" = "COMPLETED" ] && { OK=1; break; }; [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }; sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

**In the app**, pass the URI to the driver unchanged, for example `mongoose.connect(process.env.MONGODB_URI)` or `MongoClient(os.environ["MONGODB_URI"])`. Don't rebuild it from parts: the URI carries `authSource`, the replica set and TLS options. With TLS on, the image needs CA certificates (`apt-get install -y ca-certificates` or `apk add --no-cache ca-certificates`).

## Step 8: Prove MongoDB accepts connections

Open a tunnel without sudo and read the local address from the log rather than assuming `127.0.0.1:27017`:

```bash
northflank forward addon --projectId "$PROJECT_ID" --addonId app-mongo \
  --skipHostnames --portConflict random > /tmp/nf-mongo-fwd.log 2>&1 &
FWD_PID=$!
sleep 10; cat /tmp/nf-mongo-fwd.log
```

Load credentials into variables without printing them, and read the auth database from the URI:

```bash
CREDS=$(northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-mongo -o json)
pick() { echo "$CREDS" | jq -r --arg k "$1" '(.data // .) | (.secrets + .envs) | to_entries[] | select(.key|ascii_downcase==$k) | .value' | head -1; }
DB_USER=$(pick username); DB_PASS=$(pick password); DB_NAME=$(pick database)
URI=$(echo "$CREDS" | jq -r '(.data // .) | (.secrets + .envs) | to_entries[] | select((.value|type)=="string" and (.value|startswith("mongodb"))) | .value' | grep -v -i admin | head -1)
AUTH_DB=$(echo "$URI" | grep -o 'authSource=[^&]*' | cut -d= -f2); AUTH_DB=${AUTH_DB:-$DB_NAME}
mongosh --quiet --host <LOCAL_IP> --port <LOCAL_PORT> --tls --tlsAllowInvalidHostnames \
  -u "$DB_USER" -p "$DB_PASS" --authenticationDatabase "$AUTH_DB" "$DB_NAME" --eval 'db.version()'
unset CREDS DB_PASS URI; kill $FWD_PID
```

`--tlsAllowInvalidHostnames` is only for this local check, because you're connecting to a loopback IP rather than the certificate's hostname. Drop `--tls` if the addon was created without TLS.

## How to know it worked

1. `northflank get addon --projectId "$PROJECT_ID" --addonId app-mongo -o json | jq -r '(.data // .) | .status, .spec.type'` prints `running` and `mongodb`.
2. `mongosh … --eval 'db.version()'` exits 0 and prints a version starting with the major you asked for (for example `8.0.`).
3. The service has the URI, checked by scheme only:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> \
     --shell-cmd 'sh -c' --cmd '"printenv MONGODB_URI | cut -c1-10"'
   ```
   Expect `mongodb://`. `exec` drops output without a TTY, hence `script`; on Linux use `script -qefc '…' /dev/null`.
4. No `MongoServerError` or `MongoNetworkError` in the app logs since the restart:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> --duration 300 --textIncludes "Mongo"
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Authentication failed` | Wrong auth database | Use the linked URI unchanged; locally, use its `authSource` |
| `MONGODB_URI` empty in the container | Service not in `restrictions`, wrong key name, or no restart | Check `linkedKeys`; fix `nfObjects`; restart |
| TLS or certificate error in the app | Image lacks CA certificates | Install `ca-certificates`, or mount the CA as a secret file |
| Data lands in database `test` | Driver URI has no database path and app doesn't set one | Use the linked URI, which names the default database |
| Writes fail briefly during an incident | Primary failover in a replica set | Enable retryable writes (the default in current drivers) |
| Status stuck in `allocating` | Capacity or plan issue | `northflank get addon logs --projectId "$PROJECT_ID" --addonId app-mongo`; don't delete without asking |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 5 loop prints `ADDON_NOT_READY`, or the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Confirm `dbName`, replica count and storage size with the user before step 5. The name is permanent, storage can't shrink, and the MongoDB docs page says replicas can't be reduced (the scale-a-database page says they can).
- Never print `MONGODB_URI` or passwords. Print key names; check URIs with `cut`.
- Keep `externalAccessEnabled: false` unless asked; if enabled, add `ipPolicies` with `ALLOW` rules.
- Never run `db.dropDatabase()` or `northflank delete addon` without explicit confirmation for that specific database and a recent backup. Prefer `northflank pause addon` to stop compute.

## FAQ

### Where should I deploy my MongoDB database?
For a production app, a managed MongoDB database is the least work: you get a replica set, backups, TLS and a ready-made connection string without operating the nodes. Running the `mongo` Docker image on Kubernetes or a container platform, or installing MongoDB on VMs, makes sense when you need custom configuration and are prepared to own replica set setup, backups and upgrades. Keep the database in the same region as the app that reads from it.

### Do I need a replica set?
For production, 3 replicas gives automated failover. You can also set `readPreference=secondaryPreferred` in the client to move reads to secondaries; see [configure addons for high availability](https://northflank.com/docs/v1/application/databases-and-persistence/configure-addons-for-high-availability).

### How many connections can MongoDB take?
The docs give MongoDB's default limit: 80% of the system's open-file limit. A larger plan handles more.

### How do I move data from Atlas or another server?
Addons can import a dump or a live database by connection string. Follow [migrate your MongoDB database to Northflank](https://northflank.com/docs/v1/application/databases-and-persistence/migrate-data-to-northflank/migrate-your-mongodb-database-to-northflank).

### How do I back it up?
MongoDB supports native dump and disk snapshot backups, on demand or on a schedule. See [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data).

## Conclusion

You now have a managed MongoDB database, created from a reviewed JSON file, with its connection string injected as `MONGODB_URI` into only the app that needs it, and a `db.version()` check that never printed a password.

**Next:** schedule backups, then deploy the app that uses it, such as [a Node.js API](deploy-nodejs-application.md) (swap the Postgres addon for this one) or [Payload CMS](deploy-payload-cms.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy Payload CMS](deploy-payload-cms.md)
- [How to deploy a NestJS app](deploy-nestjs-app.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a MySQL database](deploy-mysql-database.md)
