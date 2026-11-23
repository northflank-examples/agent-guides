<!--
agent-guide-meta:
  id: NF-STK-FLOWISE
  description: "How to deploy Flowise with the official Docker image, managed PostgreSQL, fixed encryption and JWT secrets, a .flowise volume and a public HTTPS URL."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/stacks/deploy-flowiseai
    - https://docs.flowiseai.com/configuration/environment-variables
    - https://docs.flowiseai.com/configuration/authorization/app-level
    - https://github.com/FlowiseAI/Flowise/blob/flowise%403.1.4/docker/.env.example
    - https://github.com/FlowiseAI/Flowise/blob/flowise%403.1.4/docker/docker-compose.yml
    - https://github.com/FlowiseAI/Flowise/blob/flowise%403.1.4/Dockerfile
    - https://github.com/FlowiseAI/Flowise/blob/flowise%403.1.4/packages/server/src/enterprise/utils/authSecrets.ts
    - https://hub.docker.com/r/flowiseai/flowise/tags
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-STK-LANGFLOW, NF-STK-DIFY, NF-DB-001, NF-STK-N8N]
-->

# How to deploy Flowise

To deploy Flowise, run the official `flowiseai/flowise` Docker image on port 3000 with `DATABASE_TYPE=postgres` and the `DATABASE_*` connection variables, set fixed `FLOWISE_SECRETKEY_OVERWRITE`, `JWT_AUTH_TOKEN_SECRET`, `JWT_REFRESH_TOKEN_SECRET`, `EXPRESS_SESSION_SECRET` and `TOKEN_HASH_SECRET`, persist `/home/node/.flowise` on a volume, and set `APP_URL` to the public HTTPS address. Left unset, the encryption key and auth secrets are generated into files under `.flowise`, so a lost disk takes them with it.

This guide does it on Northflank with the CLI: a managed PostgreSQL addon, a secret group with the database credentials and generated secrets, a `flowiseai/flowise` service and a volume for `.flowise`. You end with a public HTTPS Flowise whose API rejects anonymous requests, verified by `/api/v1/ping` and a 401 check. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human creates the admin account once.

## TL;DR: how to deploy Flowise

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team and project, then create the `flowise` project.
3. Create PostgreSQL with `northflank put addon`, and a secret group that links its credentials as `DATABASE_*` plus generated encryption, JWT, session and token-hash secrets.
4. Run `flowiseai/flowise:3.1.4` on port 3000 with health checks on `/api/v1/ping`, and attach a volume at `/home/node/.flowise`.
5. Wait for the rollout, read the public host and set `APP_URL`.
6. Verify that `/api/v1/ping` returns `pong` and `/api/v1/chatflows` returns 401 without credentials, then have a human create the admin account.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Flowise defaults to SQLite and keeps its encryption key, logs and uploads under the home directory's `.flowise` folder. The official `docker-compose.yml` mounts `~/.flowise:/home/node/.flowise` and the image runs as the `node` user. For a server:

- **PostgreSQL** (`DATABASE_TYPE=postgres`) holds chatflows, credentials, users and executions.
- **`FLOWISE_SECRETKEY_OVERWRITE`** fixes the key that encrypts stored credentials, instead of a file generated on first start.
- **Auth secrets.** In Flowise 3.1.4, unset auth secrets are generated once and stored as files under `SECRETKEY_PATH` (or in AWS Secrets Manager), and the old weak defaults such as `flowise` for `EXPRESS_SESSION_SECRET` are treated as unset. Setting them as environment variables keeps them independent of the disk. `docker/.env.example` lists `JWT_AUTH_TOKEN_SECRET`, `JWT_REFRESH_TOKEN_SECRET`, `EXPRESS_SESSION_SECRET` and `TOKEN_HASH_SECRET`, generated with `openssl rand -hex 32`.
- **`APP_URL`** (default `http://localhost:3000`) is used in invite and password-reset links.
- **A volume** at `/home/node/.flowise` keeps uploads (`BLOB_STORAGE_PATH`) and logs.

The old `FLOWISE_USERNAME`/`FLOWISE_PASSWORD` login is deprecated; Flowise 3 uses accounts.

Where to host Flowise follows from that: an always-on Node.js container, PostgreSQL, a persistent disk and a stable HTTPS URL for the editor, embedded chat widgets and the prediction API. Static and serverless hosts don't fit. A VM works if you run the proxy, TLS and database yourself. A container platform with managed PostgreSQL runs the same image and handles those.

If you'd rather click through it, Northflank has a [FlowiseAI stack template](https://northflank.com/stacks/deploy-flowiseai) that deploys Flowise with PostgreSQL from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`**, to generate the secrets.
- **A pinned version.** `3.1.4` was the latest at the time of writing:
  ```bash
  curl -s "https://hub.docker.com/v2/repositories/flowiseai/flowise/tags?page_size=100&ordering=last_updated" \
    | jq -r '.results[].name' | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -n1
  ```

Names used below: project `flowise`, addon `flowise-db`, secret group `flowise-env`, service `flowise`, volume `flowise-data`.

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
northflank get project --projectId flowise -o json >/dev/null 2>&1 \
  && echo "project: flowise exists and will be reused" \
  || echo "project: flowise will be created"
```

**If you're an agent:** show the user the team, the image (`flowiseai/flowise:3.1.4`), the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "flowise", "region": "europe-west" }'
export PROJECT_ID=flowise
```

## Step 4: Create PostgreSQL and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "flowise-db", "type": "postgresql", "version": "16",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "flowise" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId flowise-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

TLS is off so Flowise's default `DATABASE_SSL=false` works; the database is reachable only inside the project. If you turn TLS on, set `DATABASE_SSL=true`.

## Step 5: Create the secret group

Print the credential key names (never the values):

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId flowise-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

If the casing differs from `HOST`, `PORT`, `DATABASE`, `USERNAME`, `PASSWORD`, change the `keyName` values below. Then create the group without echoing any secret:

```bash
jq -n --arg enc "$(openssl rand -hex 32)" --arg jwt "$(openssl rand -hex 32)" --arg rjwt "$(openssl rand -hex 32)" \
      --arg sess "$(openssl rand -hex 32)" --arg th "$(openssl rand -hex 32)" '{
  name: "flowise-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "flowise-db", keys: [
    { keyName: "HOST",     aliases: ["DATABASE_HOST"] },
    { keyName: "PORT",     aliases: ["DATABASE_PORT"] },
    { keyName: "DATABASE", aliases: ["DATABASE_NAME"] },
    { keyName: "USERNAME", aliases: ["DATABASE_USER"] },
    { keyName: "PASSWORD", aliases: ["DATABASE_PASSWORD"] }
  ] } ],
  secrets: { variables: {
    DATABASE_TYPE: "postgres", DATABASE_SSL: "false",
    FLOWISE_SECRETKEY_OVERWRITE: $enc,
    JWT_AUTH_TOKEN_SECRET: $jwt, JWT_REFRESH_TOKEN_SECRET: $rjwt,
    EXPRESS_SESSION_SECRET: $sess, TOKEN_HASH_SECRET: $th
  } }
}' > flowise-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file flowise-env.json >/dev/null && rm flowise-env.json
```

The secret group is now the only copy of the encryption key. **Tell the user to copy `FLOWISE_SECRETKEY_OVERWRITE` from the secret group in the UI to their password manager.** If it changes, Flowise can't decrypt stored credentials.

## Step 6: Deploy Flowise and attach the volume

Write `flowise.json`:

```json
{
  "name": "flowise",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "flowiseai/flowise:3.1.4" },
    "docker": { "configType": "default" }
  },
  "ports": [ { "name": "http", "internalPort": 3000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": {
    "PORT": "3000", "NUMBER_OF_PROXIES": "1",
    "SECRETKEY_PATH": "/home/node/.flowise", "BLOB_STORAGE_PATH": "/home/node/.flowise/storage",
    "LOG_PATH": "/home/node/.flowise/logs"
  },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/api/v1/ping", "port": 3000,
      "initialDelaySeconds": 20, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 12, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/api/v1/ping", "port": 3000,
      "initialDelaySeconds": 90, "periodSeconds": 30, "timeoutSeconds": 5, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file flowise.json

northflank get volume --projectId "$PROJECT_ID" --volumeId flowise-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "flowise-data",
  "mounts": [ { "containerMountPath": "/home/node/.flowise" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 5120 },
  "attachedObjects": [ { "id": "flowise", "type": "service" } ]
}'
```

`/api/v1/ping` is on Flowise's auth whitelist and is what the official Compose health check calls. `NUMBER_OF_PROXIES=1` tells Flowise's rate limiter about the load balancer in front of it. Attaching the volume redeploys the service and limits it to one instance.

## Step 7: Set `APP_URL` and wait for the rollout

This waits for the first rollout, reads the public host, sends the complete `runtimeEnvironment` map with `APP_URL` added (a patch replaces it), restarts and waits again:

```bash
wait_rollout() {
  OK=0
  for i in $(seq 1 60); do
    D=$(northflank get service --projectId "$PROJECT_ID" --serviceId flowise --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
    echo "deployment=$D"
    [ "$D" = "COMPLETED" ] && { OK=1; break; }
    [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
    sleep 10
  done
  [ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; return 1; }
}

wait_rollout
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId flowise --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId flowise --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi

[ -n "$HOST" ] && northflank patch service deployment --projectId "$PROJECT_ID" --serviceId flowise --noDefaults -o json --input "$(jq -n --arg h "$HOST" '{
  runtimeEnvironment: {
    PORT: "3000", NUMBER_OF_PROXIES: "1",
    SECRETKEY_PATH: "/home/node/.flowise", BLOB_STORAGE_PATH: "/home/node/.flowise/storage",
    LOG_PATH: "/home/node/.flowise/logs", APP_URL: ("https://" + $h)
  }
}')" \
  && northflank restart service --projectId "$PROJECT_ID" --serviceId flowise --noDefaults -o json \
  && wait_rollout
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId flowise`.

**This step needs a human:** open `https://$HOST` right away and create the admin account. Until someone does, anyone who finds the URL can claim the instance.

## How to know it worked

You're done when all of these hold:

1. The server answers:
   ```bash
   curl -s "https://$HOST/api/v1/ping"                                                   # pong
   ```
2. The API rejects anonymous requests:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/api/v1/chatflows"              # 401
   ```
3. Flowise is on PostgreSQL with the right URL (prints no secrets):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId flowise \
     --shell-cmd 'sh -c' --cmd '"printenv DATABASE_TYPE APP_URL; printenv DATABASE_HOST | cut -c1-6"'
   ```
   Expect `postgres`, `https://<HOST>` and the start of a hostname. (Linux: `script -qefc '<cmd>' /dev/null`.)
4. The volume is attached: `northflank get volume --projectId "$PROJECT_ID" --volumeId flowise-data --noDefaults -o json | jq '(.data // .).attachedObjects'` lists `flowise`.
5. After the admin account exists, a chatflow saved in the UI is still there after `northflank restart service --projectId "$PROJECT_ID" --serviceId flowise`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Logs show `ECONNREFUSED 127.0.0.1:5432` or SQLite paths | `DATABASE_*` not injected | Check key names from step 5; restart |
| SSL errors connecting to Postgres | Addon created with TLS | Set `DATABASE_SSL=true` in the group |
| Credentials fail to decrypt after a redeploy | `FLOWISE_SECRETKEY_OVERWRITE` changed | Restore the original value |
| Everyone is logged out after each deploy | JWT or session secrets missing and regenerated on a disk that wasn't kept | Check the group has all four auth secrets and the volume is attached |
| `EACCES` under `/home/node/.flowise` | Volume not writable by `node` | Restart once; if it persists, report it and see [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) |
| Invite or reset links point to `localhost` | `APP_URL` not set | Re-run step 7 |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Never print the encryption key, JWT, session or token-hash secrets, database credentials or the credentials response. Print key names only.
- Never regenerate `FLOWISE_SECRETKEY_OVERWRITE` on an instance that already stores credentials.
- Hand the admin-account step to the user immediately after the first rollout.
- Get explicit confirmation before deleting the addon, the volume or the service.

## FAQ

### Where should I host Flowise?
Anywhere that runs the Flowise Docker image as an always-on container next to PostgreSQL and a persistent volume, behind HTTPS. Serverless functions and static hosts don't fit. A VM works if you'll run the proxy, TLS and database yourself; a container platform with managed PostgreSQL, as in this guide, takes those off your hands.

### Can I scale Flowise to several instances?
Not with a read/write volume. Flowise's queue mode moves executions to workers through Redis, and uploads can go to S3 with `STORAGE_TYPE=s3`; with both in place you can drop the volume. See [How to deploy MinIO](deploy-minio.md) for S3-compatible storage.

### How do I upgrade Flowise?
Back up the database, then change the tag with `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId flowise --input '{"deployment":{"instances":1,"external":{"imagePath":"flowiseai/flowise:<NEW_VERSION>"}}}'` and wait for `COMPLETED`.

### Can Flowise use a self-hosted model?
Yes. Point the ChatOpenAI-compatible or Ollama nodes at a [vLLM server](deploy-llm-in-production-with-vllm.md) or an [Ollama service](host-ollama-in-the-cloud.md) in the same project.

## Conclusion

You now have Flowise on the official image with PostgreSQL, fixed encryption and auth secrets, a persistent `.flowise` folder and `APP_URL` pointing at its public HTTPS address.

**Next:** add a custom domain by [linking a domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) and repeat step 7 with that hostname. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy Langflow](deploy-langflow.md)
- [How to deploy Dify](deploy-dify.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to self-host n8n](self-host-n8n.md)
