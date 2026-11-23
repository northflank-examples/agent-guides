<!--
agent-guide-meta:
  id: NF-STK-VLLM
  description: "How to deploy an LLM in production with vLLM: the official Docker image on a cloud GPU, a model cache volume, an API key and an OpenAI-compatible endpoint."
  sources:
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/stacks/deploy-vllm-gcp
    - https://docs.vllm.ai/en/latest/deployment/docker.html
    - https://docs.vllm.ai/en/latest/serving/online_serving/openai_compatible_server/
    - https://docs.vllm.ai/en/latest/serving/online_serving/
    - https://docs.vllm.ai/en/latest/cli/serve/
    - https://github.com/vllm-project/vllm/blob/main/docker/Dockerfile
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
    - https://northflank.com/docs/v1/application/billing/credits
    - https://northflank.com/docs/v1/application/gpu-workloads/deploy-gpus-on-northflank-cloud
    - https://northflank.com/docs/v1/application/gpu-workloads/configure-and-optimise-workloads-for-gpus
    - https://northflank.com/docs/v1/application/use-cases/host-a-model-on-gpus
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/application/scale/scale-cpu-and-memory
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/miscellaneous/miscellaneous/list-regions
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/volumes/attach-volume
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
  related: [NF-STK-OLLAMA, NF-STK-OPENWEBUI, NF-STK-LITELLM, NF-JOB-004]
-->

# How to deploy an LLM in production with vLLM

To deploy an LLM in production with vLLM, run the official `vllm/vllm-openai` Docker image on an NVIDIA GPU sized for the model, mount a persistent volume at `/root/.cache/huggingface` so weights aren't re-downloaded, pass `HF_TOKEN` and `VLLM_API_KEY` as secrets, and give the server a long startup probe on `/health`. The API key only covers routes such as `/v1`, so `/health` and `/metrics` stay open unless you restrict the port.

This guide does it on Northflank with the CLI on an L4 GPU, serving `Qwen/Qwen3-0.6B` with a cache volume and keys in a secret group. You end with an OpenAI-compatible endpoint on a public HTTPS URL, verified by `/health`, an authenticated `/v1/models` and a chat completion. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. It needs GPU billing enabled; gated models need a Hugging Face token from a human.

## TL;DR: how to deploy an LLM in production with vLLM

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Find the regions that offer your GPU with the public `curl -s https://api.northflank.com/v1/regions`, and confirm the team, region, hourly price and project before creating anything.
3. Create the project with `northflank create project`, store `VLLM_API_KEY` (generated) and `HF_TOKEN` in a secret group with `northflank put secret`, and create a volume for `/root/.cache/huggingface` with `northflank create volume`.
4. Run `vllm/vllm-openai:<version>` with `northflank put service deployment`: plan `nf-gpu-l4-24-1g`, a `deployment.gpu` block, the model as a custom command, port 8000 and a startup probe on `/health`.
5. Verify with `GET /v1/models` and `POST /v1/chat/completions` using `Authorization: Bearer $VLLM_API_KEY`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

The vLLM Docker docs run the server like this: `docker run --runtime nvidia --gpus all -v ~/.cache/huggingface:/root/.cache/huggingface --env "HF_TOKEN=$HF_TOKEN" -p 8000:8000 --ipc=host vllm/vllm-openai:latest --model Qwen/Qwen3-0.6B`. Each flag maps to a platform setting:

- **`--gpus all`** → a GPU plan plus a `deployment.gpu` block.
- **`-v …/huggingface`** → a persistent volume at `/root/.cache/huggingface`, the path the Northflank GPU docs list for vLLM. Without it, every restart downloads the weights again.
- **`HF_TOKEN`** → a secret group variable, needed for gated models.
- **`--ipc=host`** → enough shared memory for PyTorch. The docs accept `--shm-size` instead, which maps to `deployment.storage.shmSize` (MiB). On Northflank cloud GPU plans you can leave it unset: the resource plans docs say the API defaults it to the plan's system RAM (48 GB on `nf-gpu-l4-24-1g`), and a smaller explicit value would lower it.
- **Arguments after the image** → a custom command. The image's entrypoint is `vllm serve`, and a custom command replaces the image's CMD while keeping the entrypoint, so the command is just the model and flags.

For access control, vLLM reads `--api-key` or the `VLLM_API_KEY` environment variable. Its docs say this "only authenticates requests" to `/v1`, `/v2`, `/inference` and `/cohere`, and warn: "Do not rely on `--api-key` alone to secure vLLM." The key goes in the secret group (not on the command line, where it would be visible in the service spec), and you can add an IP allow-list on top.

Where to deploy an LLM follows from those flags: you need an NVIDIA GPU with enough memory for the weights, a persistent cache so restarts don't re-download them, extra shared memory and a long-running server process. Serverless and static hosts aren't built for that. A raw GPU VM or your own Kubernetes cluster works if you install drivers and manage the server yourself. A container platform with GPU plans runs the vLLM image with the GPU, volume and secrets attached.

If you'd rather click through it, Northflank has vLLM stack templates, for example [vLLM OpenAI on GCP](https://northflank.com/stacks/deploy-vllm-gcp), which deploy into your own cloud account with an A100 and a cache disk.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **`openssl`** (to generate the API key).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **GPU access.** Free projects can't run GPUs, and the GPU docs ask for at least $50 of account credit.
- **A model that fits the GPU.** At 16-bit precision, weights take about 2 bytes per parameter, plus GPU memory for the KV cache. `Qwen/Qwen3-0.6B` (the vLLM docs' example, ungated) fits one L4 (24 GB) easily; an 8B model needs a reduced `--max-model-len` on an L4, and larger models need an A100 or H100.
- **For gated models only (Llama, Gemma, some Mistral), a Hugging Face token.** A human accepts the model's licence on Hugging Face, creates a read token and exports it as `HF_TOKEN`, never pasted into a file.
- **A pinned vLLM version.** `curl -s "https://hub.docker.com/v2/repositories/vllm/vllm-openai/tags?page_size=50" | jq -r '.results[].name' | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$'` lists them; at the time of writing, the latest is `v0.34.0`.

Names used below: project `llm`, secret group `vllm-env`, volume `hf-cache`, service `vllm`.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Check GPU availability and confirm where it will deploy

Before creating anything, print the team, whether the project exists, and the regions that offer one NVIDIA L4 (24 GB) with its on-demand price in US cents per GPU-hour. The regions endpoint is public, so it needs no token:

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
if P=$(northflank get project --projectId llm -o json 2>/dev/null); then
  echo "project: llm exists in $(echo "$P" | jq -r '(.data // .).deployment.region') and will be reused"
else
  echo "project: llm will be created"
fi

GPU=l4-24; COUNT=1
curl -s https://api.northflank.com/v1/regions | jq -r --arg g "$GPU" --argjson n "$COUNT" '
  .data.regions[] | select(any(.gpuDevices[]?; .id == $g and (.countOptions | index($n))))
  | "\(.id)\t\(.gpuDevices[] | select(.id == $g) | .pricing.onDemand) cents/h"'

GPU_PLAN=$(jq -rn --arg g "$GPU" --argjson n "$COUNT" '"nf-gpu-\($g | gsub("_"; "-"))-\($n)g"'); echo "$GPU_PLAN"   # nf-gpu-l4-24-1g
```

- If the GPU list is empty, no region offers 1 × `l4-24` right now. Pick another GPU (the `.data.regions[].gpuDevices[].id` values in the same response) and change `GPU`.
- If the project already exists, its region must be in the GPU list, because a project's region can't be changed. If it isn't, use a new project name throughout.
- **GPUs need a human.** Free projects can't run GPU workloads, and GPU deployments need account credit. Ask the account owner to confirm billing and the hourly price before you create anything.
- GPU plans aren't listed by `/v1/plans`. The resource plans docs give the ID rule `nf-gpu-<gpuType>-<gpuCount>g`, with underscores replaced by hyphens, which is what `GPU_PLAN` prints.

**If you're an agent:** show the user the team, the region (`europe-west` below, unless they choose another from the GPU list), the GPU's hourly price and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services, and every service in it runs in the project's region. The region can't be changed later. Skip this step if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "llm", "region": "europe-west", "description": "Self-hosted LLMs" }'
```

To use another region from the step 2 list, change `region`. REST: `POST https://api.northflank.com/v1/projects`.

## Step 4: Store the API key and Hugging Face token

Generate the API key locally and send it straight to the secret group without printing it:

```bash
export VLLM_API_KEY=$(openssl rand -hex 32)
jq -n --arg k "$VLLM_API_KEY" --arg hf "${HF_TOKEN:-}" '{
  name: "vllm-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "vllm", type: "service" } ] },
  secrets: { variables: ({ VLLM_API_KEY: $k } + (if $hf == "" then {} else { HF_TOKEN: $hf } end)) }
}' > vllm-env.json
northflank put secret --projectId llm --noDefaults -o json --file vllm-env.json >/dev/null && rm vllm-env.json
```

Keep `VLLM_API_KEY` in this shell for the verify step, and tell the user it's stored in the `vllm-env` secret group. It hasn't been confirmed that a group can be restricted to a service that doesn't exist yet. If the call is rejected, re-run it with `"restrictions": { "restricted": false }`.

## Step 5: Create the model cache volume

```bash
northflank get volume --projectId llm --volumeId hf-cache --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId llm --noDefaults -o json --input '{
  "name": "hf-cache",
  "mounts": [ { "containerMountPath": "/root/.cache/huggingface" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 20480 }
}'
```

`storageSize` is in MB and can only be increased. Size it at about twice the model's weight files. The volume is created unattached so the service can mount it from its first start (step 6), and the first download lands on the volume.

## Step 6: Deploy the vLLM service

Write `vllm.json`. Replace `<VLLM_VERSION>`, and the GPU values if you chose a different GPU:

```json
{
  "name": "vllm",
  "billing": { "deploymentPlan": "nf-gpu-l4-24-1g" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "vllm/vllm-openai:<VLLM_VERSION>" },
    "docker": { "configType": "customCommand", "customCommand": "Qwen/Qwen3-0.6B --host 0.0.0.0 --port 8000" },
    "gpu": { "enabled": true, "configuration": { "gpuType": "l4-24", "gpuCount": 1, "timesliced": false } }
  },
  "ports": [ { "name": "http", "internalPort": 8000, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "HTTP", "type": "startupProbe", "path": "/health", "port": 8000,
      "initialDelaySeconds": 30, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 180, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/health", "port": 8000,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/health", "port": 8000,
      "initialDelaySeconds": 10, "periodSeconds": 30, "timeoutSeconds": 5, "failureThreshold": 3 }
  ],
  "createOptions": { "volumesToAttach": ["hf-cache"] }
}
```

```bash
northflank put service deployment --projectId llm --noDefaults -o json --file vllm.json
```

REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/deployment`.

What the settings do:

- **The command** becomes `vllm serve Qwen/Qwen3-0.6B --host 0.0.0.0 --port 8000`. Port 8000 is vLLM's default, and `--host 0.0.0.0` makes it listen on all interfaces. Add flags from `vllm serve --help` as needed, for example `--max-model-len 8192` to fit a bigger model on an L4, `--served-model-name my-model` to give the model a stable name, or `--tensor-parallel-size 2` with `gpuCount: 2`. Keep the string free of quotes, because it isn't run through a shell.
- **The startup probe** gives the server up to about 30 minutes (180 × 10 s, after a 30-second delay) to download and load the weights before liveness checks can restart it. `/health` isn't under `/v1`, so it doesn't need the API key.
- **`timesliced: false`**: the GPU docs say Northflank cloud GPUs don't support timeslicing.
- **`createOptions.volumesToAttach`** mounts `hf-cache` from the first deployment. If the API rejects it, or the volume shows no attachment afterwards, remove that line, re-run the `put`, and attach the volume explicitly. Attaching redeploys the service:
  ```bash
  northflank attach volume --projectId llm --volumeId hf-cache --noDefaults -o json \
    --input '{"nfObject": {"id": "vllm", "type": "service"}}'
  ```

## Step 7: Wait for the model to load

The deployment stays `IN_PROGRESS` until the startup probe passes. Poll every 10 seconds:

```bash
OK=0
for i in $(seq 1 200); do
  S=$(northflank get service --projectId llm --serviceId vllm --noDefaults -o json)
  D=$(echo "$S" | jq -r '(.data // .).status.deployment.status // "NONE"'); echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId llm --serviceId vllm --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId llm --serviceId vllm --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

Follow progress with `northflank get service logs --projectId llm --serviceId vllm --lineLimit 100`.

## How to know it worked

You're done when all of these hold:

1. Health returns 200 (no key needed):
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/health"
   ```
2. `/v1/models` rejects requests without the key, and lists the model with it:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/v1/models"                      # 401
   curl -s "https://$HOST/v1/models" -H "Authorization: Bearer $VLLM_API_KEY" | jq -r '.data[].id'   # Qwen/Qwen3-0.6B
   ```
3. A chat completion returns text:
   ```bash
   curl -s "https://$HOST/v1/chat/completions" -H "Authorization: Bearer $VLLM_API_KEY" -H "Content-Type: application/json" \
     -d '{"model": "Qwen/Qwen3-0.6B", "messages": [{"role": "user", "content": "Reply with the word ready."}], "max_tokens": 50}' \
     | jq -r '.choices[0].message.content'
   ```
4. Weights are on the volume: `northflank get volume --projectId llm --volumeId hf-cache --noDefaults -o json | jq '(.data // .).attachedObjects'` lists `vllm`. After a restart, the logs show no new download.

Clients use `base_url="https://$HOST/v1"` and `api_key=$VLLM_API_KEY`. Services in the same project can call `http://vllm:8000/v1` without leaving the private network.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Create rejects the plan or GPU | Plan, `gpuType`, `gpuCount` or region don't match, or the project is free | Rebuild `GPU_PLAN`; re-check regions; ask a human about billing |
| Logs show `401`/`403` from Hugging Face | Gated model without `HF_TOKEN`, or licence not accepted | Human step: accept the licence, add `HF_TOKEN` to `vllm-env`, restart |
| CUDA out-of-memory at start-up | Model or context too large for the GPU | Add `--max-model-len 4096`, pick a smaller model, or a bigger GPU |
| Errors about shared memory or `/dev/shm` | `deployment.storage.shmSize` set too low (or a 64 MiB default in your own cloud) | Remove it on Northflank cloud GPU plans, or raise it (MiB) |
| Restarts during download | Startup window too short | Increase the startup probe's `failureThreshold` |
| `401` with the key | Key changed or not injected | Check that `vllm-env` is restricted to `vllm`; restart the service |
| Weights re-download on every restart | Volume not attached | Check `attachedObjects`; see step 6's fallback |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write: team, region, GPU price per hour, and whether the project is new or reused.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs with `northflank get service logs --projectId llm --serviceId vllm --lineLimit 200` before changing anything.
- Show the user the GPU price per hour before step 6. Pause with `northflank pause service` when idle.
- Never print `VLLM_API_KEY` or `HF_TOKEN`, or put them in the custom command.
- `/metrics` and `/health` aren't covered by the API key. If only known clients should call the server, add `security.ip` `ALLOW` rules on the port (see [port security policies](https://northflank.com/docs/v1/application/network/add-security-policies-for-ports)), or set `public: false` and call it from inside the project.
- Confirm before deleting the volume or the service.

## FAQ

### Where should I deploy my LLM?
On a cloud host that gives a long-running container an NVIDIA GPU sized for the model, persistent storage for the weights cache and a way to keep the API key secret. A GPU VM or self-managed Kubernetes cluster works if you'll handle drivers and upgrades; a container platform with GPU plans, as in this guide, runs the vLLM Docker image with those pieces attached.

### How do I serve a different model?
Change the first word of `customCommand` to the Hugging Face repo ID, adjust the GPU and volume size, and re-run step 6. Use `--served-model-name` so clients don't depend on the repo ID.

### Can I use more than one GPU?
Yes. Set `gpuCount` and the matching `nf-gpu-…-<count>g` plan, check that the region's `countOptions` allows that count, and add `--tensor-parallel-size <count>` to the command.

### Can I put a UI or a key-management proxy in front?
Yes. Point [Open WebUI](deploy-open-webui-with-ollama.md) at `http://vllm:8000/v1`, or add vLLM as a model in a [LiteLLM proxy](deploy-litellm-proxy.md) to issue per-user keys.

## Conclusion

You now have an open-weights model on a cloud GPU behind an OpenAI-compatible API, with an API key in a secret group and weights cached on a volume.

**Next:** add a [LiteLLM proxy](deploy-litellm-proxy.md) for per-team keys and spend limits, or a chat UI with [Open WebUI](deploy-open-webui-with-ollama.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy LiteLLM proxy](deploy-litellm-proxy.md)
- [How to deploy Open WebUI with Ollama](deploy-open-webui-with-ollama.md)
- [How to host Ollama in the cloud](host-ollama-in-the-cloud.md)
- [How to deploy AI agents in production](deploy-ai-agents-in-production.md)
