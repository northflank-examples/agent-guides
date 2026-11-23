<!--
agent-guide-meta:
  id: NF-STK-PREFECT
  description: "How to self-host Prefect 3 in production with the official image, a PostgreSQL database, basic auth on the API and UI, a process worker and PREFECT_API_URL set."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://docs.prefect.io/v3/advanced/self-hosted
    - https://docs.prefect.io/v3/advanced/security-settings
    - https://github.com/PrefectHQ/prefect/blob/3.9.4/src/prefect/settings/models/server/database.py
    - https://github.com/PrefectHQ/prefect/blob/3.9.4/src/prefect/settings/models/server/ui.py
    - https://github.com/PrefectHQ/prefect/blob/3.9.4/Dockerfile
    - https://hub.docker.com/r/prefecthq/prefect
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
  related: [NF-DB-001, NF-DB-003, NF-DEP-002, NF-STK-DAGSTER]
-->

# How to self-host Prefect

To self-host Prefect, run `prefect server start --host 0.0.0.0` from the official `prefecthq/prefect` image against PostgreSQL through the `postgresql+asyncpg` driver, set `PREFECT_SERVER_API_AUTH_STRING` because the open-source server has no users, point the UI at its public address with `PREFECT_SERVER_UI_API_URL`, and run a worker that reaches the API through `PREFECT_API_URL`. A wrong API URL is why most self-hosted UIs load but show nothing.

This guide does it on Northflank with the CLI: managed PostgreSQL linked as Prefect's database settings, the server on a public HTTPS URL with basic auth, and a process worker that talks to it privately at `http://prefect-server:4200/api`. You end with a healthy API, a UI that asks for the shared credential, and a work pool with a live worker. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to self-host Prefect

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team and project, then create the `prefect` project.
3. Create PostgreSQL with `northflank put addon`, link its credentials as `PREFECT_SERVER_DATABASE_*` in a secret group, and put a generated `admin:<password>` in a second group as `PREFECT_SERVER_API_AUTH_STRING` and `PREFECT_API_AUTH_STRING`.
4. Run `prefecthq/prefect:3.9.4-python3.12` with `prefect server start --host 0.0.0.0` on port 4200, then set `PREFECT_SERVER_UI_API_URL` to its public host.
5. Deploy a worker with `prefect worker start --pool default --type process` and `PREFECT_API_URL=http://prefect-server:4200/api`.
6. Verify that `/api/ready` returns 200 with credentials and the `default` pool shows a ready worker.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

A self-hosted Prefect deployment has two kinds of process:

- **The server** (`prefect server start`) serves the REST API, the UI and the background services (scheduler, late-run detection, event processing) from one process. By default it stores everything in SQLite. The Prefect docs require PostgreSQL 14.9 or later for production, and all database engines must use an async driver, so the driver is `postgresql+asyncpg`. The server runs migrations on start.
- **Workers** poll a work pool and run flows. A `process` worker runs each flow run as a subprocess in its own container, pulling flow code from Git when the deployment says so. The image includes `git`.

**The auth caveat.** Prefect's open-source server has no user accounts, roles or SSO; those are Prefect Cloud features. What it has is basic authentication: set `server.api.auth_string="admin:pass"` on the server and the same string as `api.auth_string` on every client. In the docs' words, "the UI will prompt for the full authentication string" on first load. Everyone shares one credential. Without it, anyone who finds the URL can run, cancel and delete your flows. If you need per-user access, add [SSO or IP rules on the port](https://northflank.com/docs/v1/application/network/add-security-policies-for-ports) in front of it, or use Prefect Cloud.

Where to self-host Prefect comes down to an always-on API server, PostgreSQL and workers that can reach the API. Serverless functions can't hold the server's background services. A VM with Docker Compose works if you manage Postgres and TLS. A container platform with managed Postgres runs the same image with less to operate.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`.**
- **You've pinned a Prefect version.** This guide uses `3.9.4-python3.12`. Use the same tag for server and workers, and the same Prefect version wherever you run `prefect deploy`.

Names used below: project `prefect`, addon `prefect-db`, secret groups `prefect-db-env` and `prefect-auth`, services `prefect-server` and `prefect-worker`.

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
northflank get project --projectId prefect -o json >/dev/null 2>&1 \
  && echo "project: prefect exists and will be reused" \
  || echo "project: prefect will be created"
```

**If you're an agent:** show the user the team, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "prefect", "region": "europe-west" }'
export PROJECT_ID=prefect
```

## Step 4: Create PostgreSQL

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "prefect-db", "type": "postgresql", "version": "16-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-50", "storage": 8192, "replicas": 1 },
  "customCredentials": { "dbName": "prefect" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId prefect-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon status and logs"; false; }
```

## Step 5: Create the secret groups

Prefect needs the `postgresql+asyncpg` scheme, so instead of the addon's `postgresql://` URI, link the individual credentials to Prefect's separate database settings. Print the key **names** first:

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId prefect-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

If the casing differs from below, change the `keyName` values. Only the server gets the database group:

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "prefect-db-env", "type": "secret", "secretType": "environment", "priority": 10,
  "restrictions": { "restricted": true, "nfObjects": [ { "id": "prefect-server", "type": "service" } ] },
  "addonDependencies": [ { "addonId": "prefect-db", "keys": [
    { "keyName": "HOST",     "aliases": ["PREFECT_SERVER_DATABASE_HOST"] },
    { "keyName": "PORT",     "aliases": ["PREFECT_SERVER_DATABASE_PORT"] },
    { "keyName": "USERNAME", "aliases": ["PREFECT_SERVER_DATABASE_USER"] },
    { "keyName": "PASSWORD", "aliases": ["PREFECT_SERVER_DATABASE_PASSWORD"] },
    { "keyName": "DATABASE", "aliases": ["PREFECT_SERVER_DATABASE_NAME"] } ] } ],
  "secrets": { "variables": { "PREFECT_SERVER_DATABASE_DRIVER": "postgresql+asyncpg" } }
}'
```

The auth string goes to the server and every worker, so this group is unrestricted:

```bash
jq -n --arg auth "admin:$(openssl rand -hex 16)" '{
  name: "prefect-auth", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  secrets: { variables: { PREFECT_SERVER_API_AUTH_STRING: $auth, PREFECT_API_AUTH_STRING: $auth } }
}' > prefect-auth.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file prefect-auth.json >/dev/null && rm prefect-auth.json
```

**Tell the user to copy `PREFECT_API_AUTH_STRING` from the secret group in the UI to their password manager.** They paste the whole `admin:…` string into the UI's prompt.

## Step 6: Deploy the server

Write `prefect-server.json`:

```json
{
  "name": "prefect-server",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "prefecthq/prefect:3.9.4-python3.12" },
    "docker": { "configType": "customCommand", "customCommand": "prefect server start --host 0.0.0.0 --port 4200" }
  },
  "ports": [ { "name": "http", "internalPort": 4200, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "TCP", "type": "readinessProbe", "port": 4200,
      "initialDelaySeconds": 15, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 6, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file prefect-server.json
```

Keep one instance. More server replicas need Redis for messaging and migrations run separately; see the FAQ.

## Step 7: Point the UI at its public URL

The UI runs in the browser, so it needs the public API address. This waits for the first rollout, reads the host, sends the complete `runtimeEnvironment` map, restarts and waits again:

```bash
wait_rollout() {
  OK=0
  for i in $(seq 1 60); do
    D=$(northflank get service --projectId "$PROJECT_ID" --serviceId prefect-server --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
    echo "deployment=$D"; [ "$D" = "COMPLETED" ] && { OK=1; break; }; [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
    sleep 10
  done
  [ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; return 1; }
}

wait_rollout
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId prefect-server --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId prefect-server --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi

[ -n "$HOST" ] && northflank patch service deployment --projectId "$PROJECT_ID" --serviceId prefect-server --noDefaults -o json \
  --input "$(jq -n --arg h "$HOST" '{ runtimeEnvironment: {
    PREFECT_SERVER_UI_API_URL: ("https://" + $h + "/api"), PREFECT_UI_URL: ("https://" + $h) } }')" \
  && northflank restart service --projectId "$PROJECT_ID" --serviceId prefect-server --noDefaults -o json \
  && wait_rollout
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId prefect-server`. When you add a custom domain, repeat this step with that hostname.

## Step 8: Deploy a worker

The worker uses the private address, so its API traffic never leaves the project. `--type process` creates the `default` pool if it doesn't exist:

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "prefect-worker",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "prefecthq/prefect:3.9.4-python3.12" },
    "docker": { "configType": "customCommand", "customCommand": "prefect worker start --pool default --type process" }
  },
  "runtimeEnvironment": { "PREFECT_API_URL": "http://prefect-server:4200/api" }
}'
```

Flow runs execute inside this container, so size its plan for your flows, and add Python dependencies by building your own image `FROM prefecthq/prefect:3.9.4-python3.12`.

## How to know it worked

1. The API is up, connected to Postgres, and protected. These send the credential without printing it:
   ```bash
   AUTH=$(script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId prefect-worker \
     --shell-cmd 'sh -c' --cmd '"printenv PREFECT_API_AUTH_STRING"' | tr -d '\r' | tail -n1)
   curl -s -o /dev/null -w '%{http_code}\n' -u "$AUTH" "https://$HOST/api/ready"     # 200
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/api/flows/count" -X POST  # 401
   unset AUTH
   ```
   `/api/ready` returns 503 if Postgres is unreachable. (Linux: `script -qefc '<cmd>' /dev/null`.)
2. The worker is connected:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId prefect-worker \
     --shell-cmd 'sh -c' --cmd '"prefect work-pool ls"'
   ```
   `default` is listed with status `READY`.
3. A deployed flow runs. From your machine, with `PREFECT_API_URL=https://$HOST/api` and `PREFECT_API_AUTH_STRING` set, deploy a flow from your repo and run it:
   ```python
   from prefect import flow

   if __name__ == "__main__":
       flow.from_source(
           source="https://github.com/<you>/<repo>.git",
           entrypoint="flows/hello.py:hello",
       ).deploy(name="hello", work_pool_name="default")
   ```
   Then `prefect deployment run 'hello/hello'` and `prefect flow-run ls --limit 1`, which shows `COMPLETED`.
4. **This step needs a human:** open `https://$HOST`, paste the auth string at the prompt, and check the run in the UI.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| UI loads but every page errors | `PREFECT_SERVER_UI_API_URL` unset or wrong | Redo step 7 with the exact host |
| Server logs show SQLite paths | Database variables not injected | Check `prefect-db-env` key names and restriction; restart |
| `InvalidPasswordError` or connection refused to Postgres | Wrong key linked | Re-check key names from step 5 |
| Worker logs `401 Unauthorized` | Auth strings differ, or `PREFECT_API_KEY` set | Keep both strings in `prefect-auth`; remove any API key |
| Pool shows `NOT_READY` | Worker can't reach `prefect-server:4200` | Same project? Check worker logs |
| Flow run fails cloning the repo | Private repo | Use a Prefect `GitRepository` with credentials in a Prefect secret block |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Never print the auth string or database password. Capture it into a variable as above and `unset` it.
- Never remove `PREFECT_SERVER_API_AUTH_STRING` to "fix" a 401; fix the client side instead.
- Don't set `PREFECT_API_KEY` against a self-hosted server; it overrides the auth string and fails.
- Back up the database before upgrading Prefect, and don't delete the addon or services without explicit, per-resource confirmation.

## FAQ

### Where should I self-host Prefect?
On a platform that keeps the server running next to PostgreSQL 14.9 or later, serves it over HTTPS, and lets workers reach it privately. A container platform with managed Postgres covers that; a VM with Docker Compose works if you run Postgres, TLS and backups yourself.

### Does self-hosted Prefect have user logins?
No. It has one shared basic-auth string. For per-user access, put SSO or IP rules on the Northflank port, or use Prefect Cloud.

### How do I run more than one server replica?
Add Redis and set Prefect's Redis messaging settings (`PREFECT_SERVER_EVENTS_MESSAGING_BROKER="prefect_redis.messaging"` and related), set `PREFECT_SERVER_DATABASE_MIGRATE_ON_START=false`, run `prefect server database upgrade -y` as a job, and run background services separately with `prefect server services start`, as the self-hosting docs describe. See [how to deploy Redis](deploy-redis.md).

### How do I upgrade Prefect?
Back up the database, change the tag on both services with `northflank patch service deployment`, server first. The server migrates the database on start.

## Conclusion

You now have a self-hosted Prefect server on PostgreSQL with basic auth on the API and UI, a UI that calls its public URL, and a process worker on the private network.

**Next:** add a custom domain with [link a domain to a port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) and repeat step 7, then add more workers or pools for different workloads. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy Dagster](deploy-dagster.md)
- [How to deploy Apache Airflow](deploy-apache-airflow.md)
