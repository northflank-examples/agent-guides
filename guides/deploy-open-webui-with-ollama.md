<!--
agent-guide-meta:
  id: NF-STK-OPENWEBUI
  description: "How to deploy Open WebUI with Ollama: the Docker image on a private network, a persistent data volume, a fixed secret key and a headless admin, all scripted."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/stacks/deploy-openwebui
    - https://docs.openwebui.com/getting-started/quick-start
    - https://docs.openwebui.com/reference/env-configuration
    - https://docs.openwebui.com/reference/monitoring
    - https://docs.openwebui.com/reference/api-endpoints
    - https://docs.openwebui.com/getting-started/advanced-topics/scaling
    - https://github.com/open-webui/open-webui/blob/main/backend/open_webui/routers/auths.py
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://github.com/open-webui/open-webui/blob/main/backend/open_webui/main.py
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/network/networking-on-northflank
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://docs.ollama.com/docker
    - https://docs.ollama.com/faq
    - https://northflank.com/docs/v1/application/gpu-workloads/deploy-gpus-on-northflank-cloud
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
  related: [NF-STK-OLLAMA, NF-STK-VLLM, NF-STK-LITELLM, NF-DEP-002]
-->

# How to deploy Open WebUI with Ollama

To deploy Open WebUI with Ollama, run the official `ghcr.io/open-webui/open-webui` image on port 8080, point `OLLAMA_BASE_URL` at Ollama's private address, mount a persistent volume at `/app/backend/data`, and set a fixed `WEBUI_SECRET_KEY` and admin account before the first start. Without the volume and fixed key, a restart loses chats and logs everyone out; without the admin variables, whoever signs up first owns the instance.

This guide does it on Northflank with the CLI: a secret group for the key and admin credentials, `/ready` and `/health` probes, Ollama reached privately at `http://ollama:11434` (or an OpenAI-compatible API instead), and a persistent volume. You end with a public HTTPS URL where the admin signs in and `/api/models` lists your models, checked with `curl`. Step 4 deploys Ollama in the same project if it isn't running there yet. Every step is a command a developer or a coding agent can run as-is.

## TL;DR: how to deploy Open WebUI with Ollama

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, the project, whether Ollama already runs there and whether the region has a GPU, then create the `llm` project if it doesn't exist and deploy Ollama if it isn't running.
3. Create a secret group with a generated `WEBUI_SECRET_KEY`, `WEBUI_ADMIN_EMAIL`, a generated `WEBUI_ADMIN_PASSWORD` and, only for OpenAI-compatible backends, `OPENAI_API_KEY`.
4. Run `ghcr.io/open-webui/open-webui:<version>` on port 8080 with `OLLAMA_BASE_URL=http://ollama:11434` (or `OPENAI_API_BASE_URL`), `/ready` and `/health` probes, and a volume at `/app/backend/data`.
5. Verify that `/health` returns 200, that you can sign in with `POST /api/v1/auths/signin`, and that `GET /api/models` lists your models.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Three things in the Open WebUI docs shape this deployment:

- **Data lives in `/app/backend/data`.** The quick start says "your chats, users and settings live in this volume… never run without it." The default database is SQLite inside that directory.
- **`WEBUI_SECRET_KEY` must be fixed.** It signs sessions and encrypts stored tokens. The image generates one if it's unset, but it's kept in a file inside the container, so a recreated container gets a new key. The docs list the effects: all sessions invalidated, MCP tools failing with `Error decrypting tokens`, and installed apps no longer connecting.
- **The first account becomes admin.** On a public URL, whoever signs up first owns the instance. `WEBUI_ADMIN_EMAIL` plus `WEBUI_ADMIN_PASSWORD` create the admin at first start when no users exist, and the docs say sign-up is then "automatically disabled".

One more: connection settings like `OLLAMA_BASE_URL` and `OPENAI_API_BASE_URL` are **`ConfigVar`** variables. Open WebUI reads them on first launch, then stores them, and after a restart "will no longer use the external environment variable values". Get them right before the first start, or change them later in **Admin Panel > Settings > Connections**.

Where to host Open WebUI follows from that: a long-running web container with persistent storage, HTTPS and a private network path to Ollama. Open WebUI itself needs no GPU, but Ollama does, so it helps if both run on the same platform. Static and serverless hosts don't keep a stateful container or a volume around. A VM with Docker works if you manage TLS, backups and upgrades yourself; a container platform runs the Docker image with a volume and HTTPS and lets it reach Ollama privately.

If you'd rather click through it, Northflank has an [Open WebUI stack template](https://northflank.com/stacks/deploy-openwebui) that deploys the same image with a volume at `/app/backend/data` from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`**, to generate the secret key and admin password.
- **A model backend.** Choose one:
  - **Ollama in the same project.** If it already runs there, note its service ID and private port (this guide assumes `ollama` and `11434`). If not, step 4 deploys it. Open WebUI must run in the **same project** to reach a private port.
  - **An OpenAI-compatible API.** You need its base URL ending in `/v1` (for example `http://vllm:8000/v1` inside the project, `http://litellm:4000/v1`, or `https://api.openai.com/v1`) and an API key, exported as `OPENAI_API_KEY`.
- **The admin's email address** from the user, exported as `ADMIN_EMAIL`.
- **A pinned version.** The docs say `:main` is a rolling build and recommend a version tag for production. Get the latest release with `curl -s https://api.github.com/repos/open-webui/open-webui/releases/latest | jq -r .tag_name` (for example `v0.13.4`).

Names used below: project `llm`, secret group `openwebui-env`, service `open-webui`, volume `openwebui-data`.

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

Before changing anything, print the deployment target and what already runs there:

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId llm -o json >/dev/null 2>&1 \
  && echo "project: llm exists and will be reused" \
  || echo "project: llm will be created"
northflank get service --projectId llm --serviceId ollama --noDefaults -o json >/dev/null 2>&1 \
  && echo "ollama:  already running in llm" \
  || echo "ollama:  not found in llm (step 4 deploys it, unless you use an OpenAI-compatible API)"
[ -n "${ADMIN_EMAIL:-}" ] && echo "admin:   $ADMIN_EMAIL" || echo "admin:   ADMIN_EMAIL is missing"
```

If Ollama will run on a GPU, the project's region must offer one, because a project's region is fixed. This prints `europe-west` (the region used below) if it offers one NVIDIA L4 (24 GB), which runs 7B–8B models comfortably; drop the `select(.id == …)` part to list every region that does:

```bash
curl -s https://api.northflank.com/v1/regions \
  | jq -r '.data.regions[] | select(.id == "europe-west") | select(any(.gpuDevices[]?; .id == "l4-24" and (.countOptions | index(1)))) | .id'
```

- If Ollama or vLLM already runs in a project under another name, use that name instead of `llm` here and in step 3.
- If `ADMIN_EMAIL` is missing, ask the user for it before going on.

**If you're an agent:** show the user the team, the region, whether the project is new or reused, whether Ollama will be deployed (and on a GPU or CPU) and the admin email. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. If Ollama or vLLM already runs in a project, use that project. Open WebUI itself needs no GPU, but if you're about to deploy Ollama on a GPU, use a region that offers it (step 2). Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "llm", "region": "europe-west" }'
export PROJECT_ID=llm
```

## Step 4: Deploy Ollama if it isn't running yet

### If Ollama isn't running yet: deploy it

Skip this step if you use an OpenAI-compatible API, or if step 2 said Ollama is already running.

Ollama runs best on an NVIDIA GPU. **This step needs a human:** free projects don't support GPU workloads, and GPU deployments need account credit (at least $50), so ask the account owner to confirm billing first. The project's region must offer the GPU (step 2 checks it).

Pin a plain Ollama version (not `-rc` or `rocm`) from `curl -s "https://hub.docker.com/v2/repositories/ollama/ollama/tags?page_size=20&ordering=last_updated" | jq -r '.results[].name'`. Write `ollama.json`, replacing `<OLLAMA_VERSION>`:

```json
{
  "name": "ollama",
  "billing": { "deploymentPlan": "nf-gpu-l4-24-1g" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "ollama/ollama:<OLLAMA_VERSION>" },
    "docker": { "configType": "default" },
    "gpu": { "enabled": true, "configuration": { "gpuType": "l4-24", "gpuCount": 1, "timesliced": false } }
  },
  "ports": [ { "name": "api", "internalPort": 11434, "public": false, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "OLLAMA_HOST": "0.0.0.0:11434", "OLLAMA_KEEP_ALIVE": "24h" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/api/tags", "port": 11434,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 6, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/api/tags", "port": 11434,
      "initialDelaySeconds": 30, "periodSeconds": 30, "timeoutSeconds": 5, "failureThreshold": 3 }
  ]
}
```

**CPU fallback.** With no GPU access, delete the `gpu` block and use a compute plan with enough RAM for the model, such as `nf-compute-400-16`. Small models like `llama3.2` run on CPU, only more slowly. The GPU plan ID follows `nf-gpu-<gpuType>-<gpuCount>g`; if the API rejects it and names a different one, use that.

Deploy it with a volume for the models, then wait for the rollout:

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file ollama.json
northflank get volume --projectId "$PROJECT_ID" --volumeId ollama-models --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "ollama-models",
  "mounts": [ { "containerMountPath": "/root/.ollama" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 20480 },
  "attachedObjects": [ { "id": "ollama", "type": "service" } ]
}'
OK=0
for i in $(seq 1 90); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId ollama --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "OLLAMA_ROLLOUT_NOT_COMPLETE: stop here and read the logs: northflank get service logs --projectId $PROJECT_ID --serviceId ollama"; false; }
```

The port stays private (`public: false`); Ollama's API has no authentication, and Open WebUI reaches it at `http://ollama:11434`. `storageSize` is in MB and can only be increased. Only if the rollout completed, pull a model (Ollama doesn't pull on start). `exec` drops output without a TTY, so wrap it with `script` (macOS shown; on Linux use `script -qefc '<cmd>' /dev/null`):

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId ollama \
  --shell-cmd 'sh -c' --cmd '"ollama pull llama3.2"'
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId ollama \
  --shell-cmd 'sh -c' --cmd '"ollama list"'
```

Replace `llama3.2` with the model the user asked for. It's written to the volume, so you only pull once.

## Step 5: Create the secret group

Generate the secret key and admin password without printing them:

```bash
WEBUI_SECRET_KEY=$(openssl rand -hex 32)
export WEBUI_ADMIN_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=')
jq -n --arg sk "$WEBUI_SECRET_KEY" --arg em "$ADMIN_EMAIL" --arg pw "$WEBUI_ADMIN_PASSWORD" --arg ok "${OPENAI_API_KEY:-}" '{
  name: "openwebui-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "open-webui", type: "service" } ] },
  secrets: { variables: ({ WEBUI_SECRET_KEY: $sk, WEBUI_ADMIN_EMAIL: $em, WEBUI_ADMIN_PASSWORD: $pw }
                         + (if $ok == "" then {} else { OPENAI_API_KEY: $ok } end)) }
}' > openwebui-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file openwebui-env.json >/dev/null && rm openwebui-env.json
unset WEBUI_SECRET_KEY
```

Keep `WEBUI_ADMIN_PASSWORD` in this shell for the verify step. Tell the user their admin password is in the `openwebui-env` secret group. If restricting the group to a service that doesn't exist yet is rejected, re-run with `"restrictions": { "restricted": false }`.

**Don't change `WEBUI_SECRET_KEY` later.** Rotating it logs everyone out and breaks stored tokens.

## Step 6: Deploy the Open WebUI service

Write `open-webui.json`. Replace `<OWUI_VERSION>`, and pick **one** `runtimeEnvironment` block from below:

```json
{
  "name": "open-webui",
  "billing": { "deploymentPlan": "nf-compute-200-4" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "ghcr.io/open-webui/open-webui:<OWUI_VERSION>" },
    "docker": { "configType": "default" }
  },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "OLLAMA_BASE_URL": "http://ollama:11434" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "startupProbe", "path": "/ready", "port": 8080,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 60, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/ready", "port": 8080,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/health", "port": 8080,
      "initialDelaySeconds": 10, "periodSeconds": 30, "timeoutSeconds": 5, "failureThreshold": 3 }
  ]
}
```

For an OpenAI-compatible API instead of Ollama, use this `runtimeEnvironment` (the key comes from the secret group):

```json
{ "ENABLE_OLLAMA_API": "False", "OPENAI_API_BASE_URL": "http://vllm:8000/v1" }
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file open-webui.json
```

REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/deployment`.

Notes on the settings:

- Port 8080 is Open WebUI's default `PORT`.
- The monitoring docs say `/health` is unauthenticated and "returns `{"status": true}` unconditionally", so it only shows the web server is answering. `/ready` returns 503 until start-up has finished and the database (and Redis, if configured) answers, and the docs recommend it for readiness probes. The startup probe allows up to 10 minutes for the first boot; liveness stays on `/health` so a slow database doesn't restart the container.
- `http://ollama:11434` uses the private address `[service-name]:[port]`, so Ollama's port never has to be public. Both services must be in the same project.
- To use both backends, keep `OLLAMA_BASE_URL` and add `OPENAI_API_BASE_URL`. For several OpenAI-compatible backends, use `OPENAI_API_BASE_URLS` and `OPENAI_API_KEYS`, which are semicolon-separated.
- The `:main` image includes local embedding and speech models. If the backend provides embeddings and you don't need local speech-to-text, the docs offer a much smaller `-slim` variant (`<OWUI_VERSION>-slim`).

## Step 7: Attach the data volume

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId openwebui-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "openwebui-data",
  "mounts": [ { "containerMountPath": "/app/backend/data" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "open-webui", "type": "service" } ]
}'
```

REST: `POST https://api.northflank.com/v1/projects/{projectId}/volumes`. Attaching redeploys the service, and the redeployed container starts with an empty data directory. It creates the admin and reads the connection variables on that start, so anything from the brief first boot without the volume is discarded. Storage can only be increased. A single read/write volume limits the service to one instance. For several replicas, follow Open WebUI's [scaling docs](https://docs.openwebui.com/getting-started/advanced-topics/scaling); that's out of scope here.

## Step 8: Wait for the rollout and get the URL

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId open-webui --noDefaults -o json)
  D=$(echo "$S" | jq -r '(.data // .).status.deployment.status // "NONE"'); echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId open-webui --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId open-webui --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. If it prints `ROLLOUT_NOT_COMPLETE`, read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId open-webui`. If the user wants OAuth/SSO later, set `WEBUI_URL=https://$HOST` first. It's also a `ConfigVar`.

## How to know it worked

You're done when all of these hold:

1. The health endpoint returns 200:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/health"
   ```
2. The headless admin can sign in. The returned JWT is accepted by the API:
   ```bash
   TOKEN=$(jq -n --arg e "$ADMIN_EMAIL" --arg p "$WEBUI_ADMIN_PASSWORD" '{email: $e, password: $p}' \
     | curl -s "https://$HOST/api/v1/auths/signin" -H "Content-Type: application/json" -d @- | jq -r '.token')
   [ -n "$TOKEN" ] && [ "$TOKEN" != "null" ] && echo SIGNIN_OK
   ```
3. Open WebUI reaches the backend and lists its models (for example `llama3.2:latest` from Ollama). This is the monitoring docs' model-connectivity check:
   ```bash
   curl -s "https://$HOST/api/models" -H "Authorization: Bearer $TOKEN" | jq -r '.data[].id'
   ```
4. A chat completion works end to end:
   ```bash
   curl -s "https://$HOST/api/chat/completions" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
     -d '{"model": "<MODEL_ID>", "messages": [{"role": "user", "content": "Reply with the word ready."}]}' | jq -r '.choices[0].message.content'
   ```
5. Data survives a restart: run `northflank restart service --projectId "$PROJECT_ID" --serviceId open-webui`, wait for `COMPLETED`, and repeat check 2. The same password still works.

Then give the user the URL and tell them to sign in as `ADMIN_EMAIL` with the password from the secret group.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `/api/models` is empty | Wrong backend URL, or Ollama has no models pulled | Check `ollama list`; fix the URL in **Admin Panel > Settings > Connections** |
| Changing `OLLAMA_BASE_URL` has no effect | `ConfigVar`: the stored value wins after the first start | Change it in the Admin Panel, or set `ENABLE_PERSISTENT_CONFIG=False` |
| `connection refused` to `ollama:11434` | Services are in different projects, or the wrong ID or port | Deploy both in one project; use the service ID |
| Everyone is logged out after a deploy | `WEBUI_SECRET_KEY` missing or changed | Keep it fixed in `openwebui-env` |
| Sign-in fails with the generated password | Admin was created before the env was set, or the volume already had users | `WEBUI_ADMIN_*` only applies when no users exist; see the Open WebUI password-reset docs |
| Startup probe fails, container restarts | Not enough memory for `:main`, or the database isn't answering (`/ready` returns 503) | Check the logs; use a larger plan, or the `-slim` tag |
| Chats lost after a restart | Volume not attached | Check `northflank get volume … | jq '(.data // .).attachedObjects'` |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, including which project to reuse and whether Ollama goes on a GPU.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE` (or step 4 prints `OLLAMA_ROLLOUT_NOT_COMPLETE`), read the logs before changing anything.
- Never print `WEBUI_SECRET_KEY`, `WEBUI_ADMIN_PASSWORD`, `OPENAI_API_KEY` or the JWT.
- Always set `WEBUI_ADMIN_EMAIL` and `WEBUI_ADMIN_PASSWORD` before the first public start, so nobody else can claim the admin account.
- Don't make the Ollama port public to "make the connection work". Put both services in one project instead.
- Get explicit confirmation before deleting the volume. It holds every user, chat and setting.

## FAQ

### Where should I host Open WebUI?
Next to your Ollama server, on a host that runs the Open WebUI Docker image as a long-lived container with a persistent volume and HTTPS. Keeping both in the same project means Open WebUI reaches Ollama over a private address and Ollama's unauthenticated API never goes public. A VM with Docker also works if you'll run the proxy and backups yourself.

### Does Open WebUI need a GPU?
No. Models run on the backend (Ollama, vLLM or a hosted API). Use the GPU for that service, not for Open WebUI.

### Can I connect to OpenAI, Anthropic or a LiteLLM proxy?
Yes. Any OpenAI-compatible endpoint works through `OPENAI_API_BASE_URL` and `OPENAI_API_KEY`. A [LiteLLM proxy](deploy-litellm-proxy.md) lets one Open WebUI reach many providers with a single key.

### How do I upgrade?
Change the tag with `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId open-webui --input '{"deployment": {"instances": 1, "external": {"imagePath": "ghcr.io/open-webui/open-webui:<NEW_VERSION>"}}}'` and wait for `COMPLETED`. Data stays on the volume, but back it up first.

## Conclusion

You now have Open WebUI on a public HTTPS URL, connected privately to your models, with a fixed secret key, an admin account no stranger can claim, and chats stored on a persistent volume.

**Next:** add a [custom domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port), then set `WEBUI_URL` to it. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to host Ollama in the cloud](host-ollama-in-the-cloud.md)
- [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)
- [How to deploy LiteLLM proxy](deploy-litellm-proxy.md)
- [How to deploy a Docker image](deploy-docker-image.md)
