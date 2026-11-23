<!--
agent-guide-meta:
  id: NF-STK-LANGFLOW
  description: "How to deploy Langflow with the official Docker image, managed PostgreSQL, LANGFLOW_* auth settings, a fixed secret key and a config volume, verified by login."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/stacks/deploy-langflow
    - https://docs.langflow.org/deployment-docker
    - https://docs.langflow.org/environment-variables
    - https://docs.langflow.org/api-keys-and-authentication
    - https://github.com/langflow-ai/langflow/blob/v1.13.3/docker/build_and_push.Dockerfile
    - https://github.com/langflow-ai/langflow/blob/v1.13.3/src/backend/base/langflow/api/health_check_router.py
    - https://hub.docker.com/r/langflowai/langflow/tags
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-STK-FLOWISE, NF-STK-DIFY, NF-DB-001, NF-STK-N8N]
-->

# How to deploy Langflow

To deploy Langflow, run the official `langflowai/langflow` Docker image on port 7860 against PostgreSQL through `LANGFLOW_DATABASE_URL`, keep `LANGFLOW_AUTO_LOGIN=False` with a `LANGFLOW_SUPERUSER` and password, set a fixed `LANGFLOW_SECRET_KEY`, and persist `LANGFLOW_CONFIG_DIR` on a volume. Without a fixed secret key, a new container can't decrypt the API keys and credentials stored in your flows.

This guide does it on Northflank with the CLI: a managed PostgreSQL addon, a secret group with the database URL, generated superuser password and secret key, a `langflowai/langflow` service and a volume for `/app/langflow`. You end with a public HTTPS Langflow editor that requires a login, verified by a health check, a token from `/api/v1/login` and a `whoami` call. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to deploy Langflow

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project and plan, then create the `langflow` project.
3. Create PostgreSQL with `northflank put addon`, then a secret group that links `POSTGRES_URI` as `LANGFLOW_DATABASE_URL` and sets `LANGFLOW_AUTO_LOGIN=False`, `LANGFLOW_SUPERUSER`, a generated password and `LANGFLOW_SECRET_KEY`.
4. Run `langflowai/langflow:1.13.3` on port 7860 with health checks on `/health_check`, and attach a volume at `/app/langflow`.
5. Wait for the rollout, then verify that `/health_check` returns `ok`, `/api/v1/login` returns a token and `/api/v1/users/whoami` returns the superuser.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Langflow's Docker quick start runs SQLite inside the container. For a server, the Langflow docs' own Compose example points `LANGFLOW_DATABASE_URL` at PostgreSQL and mounts `LANGFLOW_CONFIG_DIR` (`/app/langflow`) on a volume. Their authentication page recommends this production configuration:

- **`LANGFLOW_AUTO_LOGIN=False`**, so the editor asks for a login. Docker images already default to this, and then **`LANGFLOW_SUPERUSER_PASSWORD` is required**: startup fails without it.
- **`LANGFLOW_SECRET_KEY`** set to a generated value. It encrypts stored API keys; the docs stress it for multi-instance deployments "to ensure consistent encryption across instances".
- **`LANGFLOW_NEW_USER_IS_ACTIVE=False`**, **`LANGFLOW_ENABLE_SIGNUP=False`** and **`LANGFLOW_ENABLE_SUPERUSER_CLI=False`**, so nobody else can create or activate accounts.
- **`LANGFLOW_ACCESS_SECURE=True`** and **`LANGFLOW_REFRESH_SECURE=True`** for HTTPS deployments, so the login cookies are marked `Secure`.

The volume keeps uploaded files and anything else Langflow writes to its config directory, so a redeploy or restart doesn't lose them. Everything else lives in PostgreSQL, which you can back up and restore independently of the container.

Where to host Langflow follows from that: an always-on container (flows serve API requests at any time), PostgreSQL, a small persistent disk and a public HTTPS URL for the editor and the `/api/v1/run` endpoint. Serverless platforms don't fit a long-running Python server with a persistent config directory. A VM works if you run the proxy, TLS and database yourself. A container platform with managed PostgreSQL runs the same image and handles those.

If you'd rather click through it, Northflank has a [Langflow stack template](https://northflank.com/stacks/deploy-langflow) that deploys Langflow with PostgreSQL from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`.**
- **A pinned version.** `1.13.3` was the latest stable tag at the time of writing; Docker Hub also carries `.dev` pre-releases, so filter them out:
  ```bash
  curl -s "https://hub.docker.com/v2/repositories/langflowai/langflow/tags?page_size=100&ordering=last_updated" \
    | jq -r '.results[].name' | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -n1
  ```

Names used below: project `langflow`, addon `langflow-db`, secret group `langflow-env`, service `langflow`, volume `langflow-data`.

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
northflank get project --projectId langflow -o json >/dev/null 2>&1 \
  && echo "project: langflow exists and will be reused" \
  || echo "project: langflow will be created"
```

Langflow loads many Python packages, so this guide uses `nf-compute-200-4` (2 vCPU, 4 GB). The public plan list shows the alternatives: `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | "\(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'`.

**If you're an agent:** show the user the team, the plan, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "langflow", "region": "europe-west", "description": "Langflow" }'
export PROJECT_ID=langflow
```

## Step 4: Create PostgreSQL and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "langflow-db", "type": "postgresql", "version": "16",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "langflow" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId langflow-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

The database is reachable only inside the project. `dbName` and TLS are fixed at creation, and storage can only grow.

## Step 5: Create the secret group

Print the credential key names (never the values):

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId langflow-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

Expect `POSTGRES_URI` (format `postgresql://user:password@host:port/database`). Then generate the secrets and create the group without echoing them:

```bash
LF_PASS=$(openssl rand -hex 16)
LF_KEY=$(openssl rand -base64 32 | tr '+/' '-_' | tr -d '=')
jq -n --arg pw "$LF_PASS" --arg key "$LF_KEY" '{
  name: "langflow-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "langflow-db", keys: [ { keyName: "POSTGRES_URI", aliases: ["LANGFLOW_DATABASE_URL"] } ] } ],
  secrets: { variables: {
    LANGFLOW_AUTO_LOGIN: "False", LANGFLOW_SUPERUSER: "admin", LANGFLOW_SUPERUSER_PASSWORD: $pw,
    LANGFLOW_SECRET_KEY: $key, LANGFLOW_NEW_USER_IS_ACTIVE: "False",
    LANGFLOW_ENABLE_SIGNUP: "False", LANGFLOW_ENABLE_SUPERUSER_CLI: "False",
    LANGFLOW_ACCESS_SECURE: "True", LANGFLOW_REFRESH_SECURE: "True"
  } }
}' > langflow-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file langflow-env.json >/dev/null && rm langflow-env.json
unset LF_KEY
```

Keep `LF_PASS` in this shell for the checks. **Tell the user to copy the superuser password from the `langflow-env` secret group into their password manager**, and never to change `LANGFLOW_SECRET_KEY` once flows store credentials.

## Step 6: Deploy the Langflow service

Write `langflow.json`:

```json
{
  "name": "langflow",
  "billing": { "deploymentPlan": "nf-compute-200-4" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "langflowai/langflow:1.13.3" },
    "docker": { "configType": "default" }
  },
  "ports": [ { "name": "http", "internalPort": 7860, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "LANGFLOW_HOST": "0.0.0.0", "LANGFLOW_PORT": "7860", "LANGFLOW_CONFIG_DIR": "/app/langflow", "LANGFLOW_LOG_LEVEL": "info" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "startupProbe", "path": "/health_check", "port": 7860,
      "initialDelaySeconds": 20, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 30, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/health_check", "port": 7860,
      "initialDelaySeconds": 5, "periodSeconds": 15, "timeoutSeconds": 5, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/health", "port": 7860,
      "initialDelaySeconds": 10, "periodSeconds": 30, "timeoutSeconds": 5, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file langflow.json
```

In the Langflow source, `/health_check` checks the database and other services and returns 500 on failure, while `/health` can answer before Langflow is fully up, so readiness uses the first and liveness the second. The startup probe gives the first boot (database migrations) up to five minutes.

## Step 7: Attach the config volume

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId langflow-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "langflow-data",
  "mounts": [ { "containerMountPath": "/app/langflow" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 5120 },
  "attachedObjects": [ { "id": "langflow", "type": "service" } ]
}'
```

Attaching redeploys the service. With one read/write volume, the service runs as a single instance.

## Step 8: Wait for the rollout

```bash
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId langflow --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId langflow --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId langflow --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId langflow`.

## How to know it worked

You're done when all of these hold:

1. The health check reports the database as healthy:
   ```bash
   curl -s "https://$HOST/health_check" | jq '.'
   ```
   Expect `"status": "ok"` and `"db": "ok"`.
2. The superuser can log in and the token works:
   ```bash
   TOKEN=$(curl -s -X POST "https://$HOST/api/v1/login" -H "Content-Type: application/x-www-form-urlencoded" \
     --data-urlencode "username=admin" --data-urlencode "password=$LF_PASS" | jq -r '.access_token')
   curl -s "https://$HOST/api/v1/users/whoami" -H "Authorization: Bearer $TOKEN" | jq '{username, is_superuser}'
   ```
3. Anonymous requests are refused: `curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/api/v1/users/whoami"` prints `401` or `403`.
4. The database URL is injected (prints only the scheme):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId langflow \
     --shell-cmd 'sh -c' --cmd '"printenv LANGFLOW_DATABASE_URL | cut -d: -f1"'
   ```
   Expect `postgresql`. (Linux: `script -qefc '<cmd>' /dev/null`.)

To call flows from code, create an API key with `POST /api/v1/api_key/` (Bearer token, body `{"name": "my-key"}`) and send it as `x-api-key` to `/api/v1/run/<flow-id>`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Container exits at start mentioning the superuser password | `LANGFLOW_SUPERUSER_PASSWORD` missing | Check `langflow-env`; restart |
| `/health_check` returns 500, logs show a connection error | `LANGFLOW_DATABASE_URL` not injected or wrong key | Re-check key names from step 5 |
| Stored API keys fail to decrypt after a redeploy | `LANGFLOW_SECRET_KEY` changed | Restore the original key |
| `Permission denied` under `/app/langflow` | Volume not writable by UID 1000 | Restart once; if it persists, report it and see [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) |
| Out-of-memory restarts | Plan too small for loaded components | Move to `nf-compute-200-8` |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Never print `LANGFLOW_SUPERUSER_PASSWORD`, `LANGFLOW_SECRET_KEY`, database credentials or API keys. Print key names only.
- Never regenerate `LANGFLOW_SECRET_KEY` on an instance that already holds credentials.
- Langflow components can run arbitrary Python. Keep signup off and only create accounts for people the user trusts.
- Get explicit confirmation before deleting the addon, the volume or the service.

## FAQ

### Where should I host Langflow?
Anywhere that runs the Langflow Docker image as an always-on container next to PostgreSQL and a small persistent volume, behind HTTPS. Serverless functions don't fit. A VM works if you manage the proxy, TLS and database; a container platform with managed PostgreSQL, like this setup, handles those for you.

### Can I run more than one instance?
Not with a read/write volume attached. For multiple instances, drop the volume, keep the fixed `LANGFLOW_SECRET_KEY` and PostgreSQL, and move file storage off local disk; see Langflow's multi-worker deployment guide.

### How do I upgrade Langflow?
Back up the database, then change the tag with `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId langflow --input '{"deployment":{"instances":1,"external":{"imagePath":"langflowai/langflow:<NEW_VERSION>"}}}'` and wait for `COMPLETED`. Langflow migrates the database on start.

### Can I use it with a self-hosted model?
Yes. Point Langflow's OpenAI-compatible model components at a [vLLM server](deploy-llm-in-production-with-vllm.md) or an [Ollama service](host-ollama-in-the-cloud.md) in the same project.

## Conclusion

You now have Langflow on the official image with PostgreSQL, login enforced, a fixed secret key and a persistent config directory, behind a public HTTPS URL.

**Next:** add a custom domain by [linking a domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port), and create per-app API keys for flows you call from code. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy Flowise](deploy-flowise.md)
- [How to deploy Dify](deploy-dify.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to self-host n8n](self-host-n8n.md)
