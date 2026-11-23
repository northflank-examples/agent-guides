<!--
agent-guide-meta:
  id: NF-STK-LITELLM
  description: "How to deploy LiteLLM proxy with Docker and managed PostgreSQL, with provider keys in a secret group, config.yaml as a secret file and virtual keys verified."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/stacks/deploy-litellm
    - https://docs.litellm.ai/docs/proxy/docker_quick_start
    - https://docs.litellm.ai/docs/proxy/configs
    - https://docs.litellm.ai/docs/proxy/prod
    - https://docs.litellm.ai/docs/proxy/health
    - https://docs.litellm.ai/docs/proxy/virtual_keys
    - https://docs.litellm.ai/docs/providers/vllm
    - https://github.com/BerriAI/litellm/blob/main/litellm/deploy/docker/docker-compose.quickstart.yml
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/secure/upload-secret-files
    - https://northflank.com/docs/v1/application/infrastructure-as-code/make-a-template-dynamic
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DB-001, NF-STK-VLLM, NF-STK-OLLAMA, NF-STK-OPENWEBUI]
-->

# How to deploy LiteLLM proxy

To deploy LiteLLM proxy in production, run the official Docker image with a `config.yaml` that reads provider keys through `os.environ/`, connect a PostgreSQL database through `DATABASE_URL`, set `LITELLM_MASTER_KEY` and `LITELLM_SALT_KEY` as secrets, and probe `/health/liveliness` and `/health/readiness` on port 4000. Without the database, virtual keys, budgets and spend tracking don't work, and changing the salt key later makes stored credentials unreadable.

This guide does it on Northflank with the CLI: a managed PostgreSQL addon, a secret group for the master, salt and provider keys, and `config.yaml` mounted as a secret file. You end with the proxy on a public HTTPS URL, verified by a readiness check, a model list, a chat completion and a virtual key. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human supplies the provider keys.

## TL;DR: how to deploy LiteLLM proxy

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, the project and that the provider keys are set, then create the `llm` project if it doesn't exist.
3. Create PostgreSQL with `northflank put addon`, then a secret group that links `POSTGRES_URI` as `DATABASE_URL` and stores `LITELLM_MASTER_KEY`, `LITELLM_SALT_KEY` and the provider keys.
4. Run `ghcr.io/berriai/litellm:<tag>` with `config.yaml` mounted as a secret file at `/etc/litellm/config.yaml` and probes on `/health/liveliness` and `/health/readiness`.
5. Verify with `GET /v1/models` and `POST /v1/chat/completions` using the master key, then create and use a virtual key with `POST /key/generate`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

The LiteLLM docs define the pieces:

- **`config.yaml`** lists models under `model_list`. Secrets stay out of the file through the `os.environ/VAR_NAME` syntax, for example `api_key: os.environ/OPENAI_API_KEY`. The proxy starts with `--config /path/to/config.yaml`.
- **`LITELLM_MASTER_KEY`** "authorizes every management API call" and is the Admin UI password. The production docs say it must start with `sk-`.
- **`LITELLM_SALT_KEY`** encrypts provider keys stored in the database. It "has no in-place rotation", and changing it makes stored credentials unreadable until they are re-entered.
- **A database (`DATABASE_URL`)** is required for virtual keys, budgets and spend tracking. Without one, requests with a virtual key fail with `No connected db.` The proxy runs `prisma migrate deploy` on start-up unless `DISABLE_SCHEMA_UPDATE=true`.
- **Health probes.** `/health/liveliness` and `/health/readiness` need no auth and make no LLM calls. `/health` sends a real test request to every model, so it isn't a probe endpoint.

On Northflank, that's a PostgreSQL addon, a secret group, and a deployment service with `config.yaml` mounted as a secret file. The file's content is stored base64-encoded and decoded to UTF-8 when it's mounted.

So where to host LiteLLM? It's a long-running server that every LLM call passes through, plus a PostgreSQL database for keys and spend, and it runs migrations on start. Serverless hosts don't suit a gateway that holds a database connection and sits in the path of streaming responses. A VM with Docker works if you manage TLS, Postgres and upgrades yourself; Kubernetes works too but means writing and maintaining the manifests. A container platform with managed PostgreSQL runs the official Docker image and keeps it on the same private network as any self-hosted models.

If you'd rather click through it, Northflank has a [LiteLLM stack template](https://northflank.com/stacks/deploy-litellm) that deploys the official image with PostgreSQL and preconfigured master and salt keys from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl` and `base64`**, to generate keys and encode the config.
- **The provider keys** the user wants to route to, exported as `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` and so on. **This needs a human:** the user creates these in each provider's console and passes them through environment variables, never through a committed file.
- **The model IDs** to expose, for example the provider's model names, or the model a [vLLM server](deploy-llm-in-production-with-vllm.md) in the same project serves.
- **An image tag.** The production docs say to pin a version rather than `:latest`. This guide uses `ghcr.io/berriai/litellm:main-stable` from those docs. Swap in a specific release tag for production.

Names used below: project `llm`, addon `litellm-db`, secret group `litellm-env`, service `litellm`.

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

Before changing anything, print the deployment target and check which provider keys are set (names only, never values):

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId llm -o json >/dev/null 2>&1 \
  && echo "project: llm exists and will be reused" \
  || echo "project: llm will be created"
for k in OPENAI_API_KEY ANTHROPIC_API_KEY; do
  printenv "$k" >/dev/null && echo "$k: set" || echo "$k: missing"
done
```

- If your vLLM or Ollama services already run in a project under another name, use that name instead of `llm` here and in step 3, so LiteLLM can reach them privately.
- Every provider in `config.yaml` (step 6) needs its key set. Drop the models whose key says `missing`, or ask the user for the key.

**If you're an agent:** show the user the team, the region (`europe-west` below, unless they choose another), whether the project is new or reused, and which provider keys are set. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Use the project your vLLM or Ollama services run in, if there is one, so LiteLLM can reach them privately. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "llm", "region": "europe-west" }'
export PROJECT_ID=llm
```

## Step 4: Create PostgreSQL and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "litellm-db", "type": "postgresql", "version": "16-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "litellm" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId litellm-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

The database is reachable only from inside the project, so this guide leaves TLS off to keep the connection string simple. `dbName` is fixed at creation, and storage can only be increased. Spend logs grow with traffic, so watch disk usage.

## Step 5: Create the secret group

Print the credential key names (never the values) and confirm `POSTGRES_URI` is there:

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId litellm-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

Generate the master and salt keys with the `sk-` prefix (required for the master key, and used for both in LiteLLM's quickstart), and create the group without echoing anything:

```bash
export LITELLM_MASTER_KEY="sk-$(openssl rand -hex 32)"
LITELLM_SALT_KEY="sk-$(openssl rand -hex 32)"
jq -n --arg mk "$LITELLM_MASTER_KEY" --arg sk "$LITELLM_SALT_KEY" \
      --arg oa "${OPENAI_API_KEY:-}" --arg an "${ANTHROPIC_API_KEY:-}" '{
  name: "litellm-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "litellm", type: "service" } ] },
  addonDependencies: [ { addonId: "litellm-db", keys: [ { keyName: "POSTGRES_URI", aliases: ["DATABASE_URL"] } ] } ],
  secrets: { variables: ({ LITELLM_MASTER_KEY: $mk, LITELLM_SALT_KEY: $sk }
    + (if $oa == "" then {} else { OPENAI_API_KEY: $oa } end)
    + (if $an == "" then {} else { ANTHROPIC_API_KEY: $an } end)) }
}' > litellm-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file litellm-env.json >/dev/null && rm litellm-env.json
unset LITELLM_SALT_KEY
```

Keep `LITELLM_MASTER_KEY` in this shell for verification. If restricting the group to a service that doesn't exist yet is rejected, re-run with `"restrictions": { "restricted": false }`. Add other providers the same way (`GEMINI_API_KEY`, `VLLM_API_KEY`, …).

**Never regenerate `LITELLM_SALT_KEY`** once models or credentials are stored in the database. Tell the user it's in `litellm-env` and should also be kept in their password manager.

## Step 6: Write `config.yaml`

Save this as `litellm-config.yaml`. Keep only the models the user asked for, and ask them for the exact provider model IDs:

```yaml
model_list:
  - model_name: openai-default
    litellm_params:
      model: openai/<OPENAI_MODEL_ID>
      api_key: os.environ/OPENAI_API_KEY
  - model_name: claude-default
    litellm_params:
      model: anthropic/<ANTHROPIC_MODEL_ID>
      api_key: os.environ/ANTHROPIC_API_KEY
  # Self-hosted vLLM in the same project (optional)
  - model_name: qwen-small
    litellm_params:
      model: hosted_vllm/Qwen/Qwen3-0.6B
      api_base: http://vllm:8000/v1
      api_key: os.environ/VLLM_API_KEY

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

`model_name` is what clients send in `"model"`. The `hosted_vllm/` prefix routes to an OpenAI-compatible vLLM server at `api_base`, according to LiteLLM's vLLM provider docs. Don't write `${…}` anywhere in this file: Northflank secret files substitute `${ENV_KEY}` templates, and LiteLLM's `os.environ/` syntax avoids that collision.

## Step 7: Deploy the LiteLLM service

Build the service definition with the config embedded as base64:

```bash
CONFIG_B64=$(base64 < litellm-config.yaml | tr -d '\n')
jq -n --arg cfg "$CONFIG_B64" '{
  name: "litellm",
  billing: { deploymentPlan: "nf-compute-100-4" },
  deployment: {
    instances: 1,
    external: { imagePath: "ghcr.io/berriai/litellm:main-stable" },
    docker: { configType: "customCommand", customCommand: "--config /etc/litellm/config.yaml --port 4000" }
  },
  ports: [ { name: "http", internalPort: 4000, public: true, protocol: "HTTP" } ],
  runtimeEnvironment: { STORE_MODEL_IN_DB: "True" },
  runtimeFiles: { "/etc/litellm/config.yaml": { data: $cfg, encoding: "utf-8" } },
  healthChecks: [
    { protocol: "HTTP", type: "startupProbe", path: "/health/liveliness", port: 4000,
      initialDelaySeconds: 10, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 60, successThreshold: 1 },
    { protocol: "HTTP", type: "readinessProbe", path: "/health/readiness", port: 4000,
      initialDelaySeconds: 5, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 3, successThreshold: 1 },
    { protocol: "HTTP", type: "livenessProbe", path: "/health/liveliness", port: 4000,
      initialDelaySeconds: 10, periodSeconds: 20, timeoutSeconds: 3, failureThreshold: 3 }
  ]
}' > litellm.json
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file litellm.json
```

REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/deployment`. Notes:

- The custom command replaces the image's CMD and keeps its entrypoint, so the container runs the proxy with `--config … --port 4000`, matching `docker run … litellm --config /app/config.yaml` in the docs. Port 4000 is the proxy's port in every LiteLLM example.
- `STORE_MODEL_IN_DB=True` (as in LiteLLM's compose file) lets the Admin UI add models to the database. Models in `config.yaml` keep working either way.
- The startup probe gives the first boot up to 10 minutes for database migrations. Readiness returns 503 while the database is unreachable.
- `nf-compute-100-4` (1 vCPU, 4 GB) follows the production docs' sizing of about 1 CPU and 4 GiB per worker. LiteLLM runs one worker by default. Scale out with more instances, and add Redis once you run more than one, as those docs advise.
- To change the config later, re-run this step with the new file, then `northflank restart service --projectId "$PROJECT_ID" --serviceId litellm`.

## Step 8: Wait for the rollout and get the URL

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId litellm --noDefaults -o json)
  D=$(echo "$S" | jq -r '(.data // .).status.deployment.status // "NONE"'); echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId litellm --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId litellm --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId litellm`. The troubleshooting table covers the common causes.

## How to know it worked

You're done when all of these hold:

1. Readiness is healthy and the database is connected:
   ```bash
   curl -s "https://$HOST/health/readiness" | jq '{status, db}'
   ```
2. Unauthenticated calls are rejected, and the master key lists the models from `config.yaml`:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/v1/models"     # 401
   curl -s "https://$HOST/v1/models" -H "Authorization: Bearer $LITELLM_MASTER_KEY" | jq -r '.data[].id'
   ```
3. A chat completion goes through to a provider:
   ```bash
   curl -s "https://$HOST/v1/chat/completions" -H "Authorization: Bearer $LITELLM_MASTER_KEY" -H "Content-Type: application/json" \
     -d '{"model": "openai-default", "messages": [{"role": "user", "content": "Reply with the word ready."}]}' | jq -r '.choices[0].message.content'
   ```
4. The database works: a virtual key can be created and used. Don't print the key:
   ```bash
   VKEY=$(curl -s "https://$HOST/key/generate" -H "Authorization: Bearer $LITELLM_MASTER_KEY" -H "Content-Type: application/json" \
     -d '{"models": ["openai-default"], "metadata": {"purpose": "deploy-check"}}' | jq -r '.key')
   curl -s "https://$HOST/v1/models" -H "Authorization: Bearer $VKEY" | jq -r '.data[].id'    # openai-default
   ```

The Admin UI is at `https://$HOST/ui`. The username is `admin` and the password is the master key. Clients use `base_url="https://$HOST"` (or `…/v1`) with a virtual key.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Readiness 503, logs show Prisma connection errors | `DATABASE_URL` not injected, or the addon isn't `running` | Check the key names from step 5 and the group's restriction; restart |
| `No connected db.` on virtual keys | No `DATABASE_URL` | As above |
| Start-up error reading the config file | File not mounted, or not readable by the image's non-root user | Check `runtimeFiles` with `get service`; if it's a permission error, report it to the user (the mounted file's mode isn't documented) |
| `AuthenticationError` from a provider | Provider key missing from the group, or the wrong env var name | Match `os.environ/NAME` with the secret's key; restart |
| Stored provider credentials suddenly unreadable | `LITELLM_SALT_KEY` changed | Restore the original salt key |
| Calls to `http://vllm:8000/v1` fail | vLLM is in another project, or `VLLM_API_KEY` is missing | Same project; add the key to `litellm-env` |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, including which project to reuse.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Never print the master key, salt key, provider keys, generated virtual keys or `DATABASE_URL`. Print key names only.
- Don't send `config.yaml` with raw keys inline. Always use `os.environ/…`.
- Never change `LITELLM_SALT_KEY` on a live instance.
- Back up the database before upgrading the image, because migrations run on start. Get explicit confirmation before deleting the addon or service, and prefer pausing.

## FAQ

### Where should I deploy my LiteLLM proxy?
On a host that runs the LiteLLM Docker image as an always-on container with a PostgreSQL database next to it and HTTPS in front. Ideally it shares a private network with any self-hosted models (vLLM, Ollama) so that traffic never leaves the project. A VM or Kubernetes cluster works if you'll run the database and TLS yourself; a container platform with managed PostgreSQL, as in this guide, handles both.

### Can I run LiteLLM without a database?
Yes, but only the master key works. The docs say there are no virtual keys, Admin UI model management or spend tracking, and budgets aren't enforced.

### How do I add a model without redeploying?
With `STORE_MODEL_IN_DB=True`, add it in the Admin UI under **Models + Endpoints**. It's stored in Postgres. Or edit `config.yaml` and re-run step 7.

### Can Open WebUI use this proxy?
Yes. Set Open WebUI's `OPENAI_API_BASE_URL` to `http://litellm:4000/v1` and `OPENAI_API_KEY` to a virtual key. See [how to deploy Open WebUI](deploy-open-webui-with-ollama.md).

## Conclusion

You now have a LiteLLM gateway on a public HTTPS URL, backed by managed PostgreSQL, with provider keys in a secret group, a config file mounted as a secret, and virtual keys that survive restarts.

**Next:** add a self-hosted model with [vLLM](deploy-llm-in-production-with-vllm.md), and put [Open WebUI](deploy-open-webui-with-ollama.md) in front for a chat UI. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)
- [How to deploy Open WebUI with Ollama](deploy-open-webui-with-ollama.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
