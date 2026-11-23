<!--
agent-guide-meta:
  id: NF-STK-WHISPER
  description: "How to deploy Whisper as an OpenAI-compatible speech-to-text API with Speaches (faster-whisper) on a cloud GPU, with a model volume, API key and CPU fallback."
  sources:
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://speaches.ai/installation/
    - https://speaches.ai/configuration/
    - https://speaches.ai/usage/speech-to-text/
    - https://github.com/speaches-ai/speaches/blob/v0.8.3/Dockerfile
    - https://github.com/speaches-ai/speaches/blob/v0.8.3/src/speaches/main.py
    - https://github.com/speaches-ai/speaches/blob/v0.8.3/src/speaches/config.py
    - https://github.com/speaches-ai/speaches/blob/v0.8.3/src/speaches/routers/models.py
    - https://github.com/SYSTRAN/faster-whisper
    - https://huggingface.co/Systran/faster-whisper-large-v3
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
    - https://northflank.com/docs/v1/application/gpu-workloads/deploy-gpus-on-northflank-cloud
    - https://northflank.com/docs/v1/application/gpu-workloads/configure-and-optimise-workloads-for-gpus
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/volumes/attach-volume
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
  related: [NF-STK-VLLM, NF-STK-OLLAMA, NF-STK-LITELLM, NF-DEP-002]
-->

# How to deploy Whisper

To deploy Whisper as a speech-to-text API, run an OpenAI-compatible server such as Speaches (faster-whisper) from its CUDA Docker image on an NVIDIA GPU, persist the Hugging Face cache on a volume, set an `API_KEY`, download a model like `Systran/faster-whisper-large-v3`, and send audio to `POST /v1/audio/transcriptions`. Without a GPU, the same server runs from its CPU image with a smaller int8 model.

This guide does it on Northflank with the CLI: Speaches on an L4 GPU, a model cache volume and the API key in a secret group, with a CPU plan as fallback. You end with a public HTTPS endpoint that rejects requests without the key and transcribes a sample clip with it. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run; GPU billing needs a human.

## TL;DR: how to deploy Whisper

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check which regions offer L4 GPUs with the public `curl -s https://api.northflank.com/v1/regions`, and confirm the team, region, hourly price and project before creating anything.
3. Create the project with `northflank create project`, store a generated `API_KEY` in a secret group with `northflank put secret`, and create a volume for `/home/ubuntu/.cache/huggingface/hub` with `northflank create volume`.
4. Run `ghcr.io/speaches-ai/speaches:0.8.3-cuda` with `northflank put service deployment` on plan `nf-gpu-l4-24-1g` with a `deployment.gpu` block, port 8000 and `ENABLE_UI=false` (or `0.8.3-cpu` on a compute plan), then download the model with `POST /v1/models/Systran/faster-whisper-large-v3`.
5. Verify that `/v1/models` is rejected without the key, and that `POST /v1/audio/transcriptions` returns text with it.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why Speaches and faster-whisper?

OpenAI's Whisper repository is a Python library and CLI, not a server. To expose it as an API you need a server around it. This guide uses [Speaches](https://speaches.ai/) (formerly `faster-whisper-server`), which serves faster-whisper, a CTranslate2 reimplementation of Whisper, behind OpenAI's API shape:

- **`/v1/audio/transcriptions` and `/v1/audio/translations`** accept the same multipart requests as OpenAI's API, so the OpenAI SDKs work with a changed `base_url`.
- **Separate GPU and CPU images**: `ghcr.io/speaches-ai/speaches:<version>-cuda` and `:<version>-cpu`, both listening on port 8000.
- **Models load on demand.** `POST /v1/models/{id}` downloads a model into the Hugging Face cache at `/home/ubuntu/.cache/huggingface/hub`, which the Speaches install docs mount as a volume. Loaded models unload after `WHISPER__TTL` seconds (default 300) of inactivity (renamed `STT_MODEL_TTL` in the 0.9 release candidates).
- **`API_KEY`** turns on bearer-token checks. Speaches also mounts a Gradio UI at `/` by default, which isn't covered by the key, so this guide sets `ENABLE_UI=false`.

vLLM can also serve Whisper checkpoints on `/v1/audio/transcriptions`. Use it if you already run vLLM (see [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)); Speaches is the simpler choice when speech-to-text is the only job and you want a CPU fallback.

Where to deploy Whisper follows from that: a long-running container with an NVIDIA GPU (or several CPU cores for small models), persistent storage for model weights, a secret API key and an HTTPS endpoint that accepts audio uploads. Serverless functions struggle with multi-gigabyte models and cold starts. A GPU VM works if you install drivers and run the proxy yourself. A container platform with GPU plans runs the image with the GPU, volume and secret attached.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **`openssl`** (to generate the API key).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **For the GPU path, GPU access.** Free projects can't run GPUs, and GPU deployments need account credit. Without it, use the CPU fallback in step 6.
- **A model.** `Systran/faster-whisper-large-v3` (about 3 GB) fits easily on an L4's 24 GB. On CPU, use `Systran/faster-whisper-small`; distilled and English-only variants such as `Systran/faster-distil-whisper-large-v3` are faster.

Names used below: project `speech`, secret group `whisper-env`, volume `whisper-models`, service `whisper`.

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
if P=$(northflank get project --projectId speech -o json 2>/dev/null); then
  echo "project: speech exists in $(echo "$P" | jq -r '(.data // .).deployment.region') and will be reused"
else
  echo "project: speech will be created"
fi

GPU=l4-24; COUNT=1
curl -s https://api.northflank.com/v1/regions | jq -r --arg g "$GPU" --argjson n "$COUNT" '
  .data.regions[] | select(any(.gpuDevices[]?; .id == $g and (.countOptions | index($n))))
  | "\(.id)\t\(.gpuDevices[] | select(.id == $g) | .pricing.onDemand) cents/h"'
```

- If the GPU list is empty, no region offers 1 × `l4-24` right now. Pick another GPU (the `.data.regions[].gpuDevices[].id` values in the same response) and change `GPU`.
- If the project already exists, its region must be in the GPU list, because a project's region can't be changed. If it isn't, use a new project name throughout.
- **GPUs need a human.** Free projects can't run GPU workloads, and GPU deployments need account credit. Ask the account owner to confirm billing and the hourly price before you create anything. Without GPU access, use the CPU fallback in step 6.
- GPU plans aren't listed by `/v1/plans`. The resource plans docs give the ID rule `nf-gpu-<gpuType>-<gpuCount>g`, so one L4 is `nf-gpu-l4-24-1g`.

**If you're an agent:** show the user the team, the region (`europe-west` below, unless they choose another from the GPU list), the GPU's hourly price and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services, and every service in it runs in the project's region. The region can't be changed later. Skip this step if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "speech", "region": "europe-west", "description": "Speech-to-text API" }'
```

To use another region from the step 2 list, change `region`. REST: `POST https://api.northflank.com/v1/projects`. Pick a GPU region even if you start on CPU, so you can switch later without a new project.

## Step 4: Store the API key

```bash
export WHISPER_API_KEY=$(openssl rand -hex 32)
jq -n --arg k "$WHISPER_API_KEY" '{
  name: "whisper-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  secrets: { variables: { API_KEY: $k } }
}' > whisper-env.json
northflank put secret --projectId speech --noDefaults -o json --file whisper-env.json >/dev/null && rm whisper-env.json
```

Keep `WHISPER_API_KEY` in this shell for the checks, and tell the user it is stored in the `whisper-env` secret group. Speaches reads `API_KEY` and rejects requests without `Authorization: Bearer <key>`.

## Step 5: Create the model cache volume

```bash
northflank get volume --projectId speech --volumeId whisper-models --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId speech --noDefaults -o json --input '{
  "name": "whisper-models",
  "mounts": [ { "containerMountPath": "/home/ubuntu/.cache/huggingface/hub" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 }
}'
```

`storageSize` is in MB and can only be increased. 10 GB holds a few Whisper models. The volume limits the service to one instance.

## Step 6: Deploy the Speaches service

Write `whisper.json`:

```json
{
  "name": "whisper",
  "billing": { "deploymentPlan": "nf-gpu-l4-24-1g" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "ghcr.io/speaches-ai/speaches:0.8.3-cuda" },
    "docker": { "configType": "default" },
    "gpu": { "enabled": true, "configuration": { "gpuType": "l4-24", "gpuCount": 1, "timesliced": false } }
  },
  "ports": [ { "name": "http", "internalPort": 8000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "ENABLE_UI": "false", "LOG_LEVEL": "info", "WHISPER__TTL": "-1" },
  "healthChecks": [
    { "protocol": "TCP", "type": "startupProbe", "port": 8000,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 60, "successThreshold": 1 },
    { "protocol": "TCP", "type": "livenessProbe", "port": 8000,
      "initialDelaySeconds": 10, "periodSeconds": 30, "timeoutSeconds": 3, "failureThreshold": 3 }
  ],
  "createOptions": { "volumesToAttach": ["whisper-models"] }
}
```

```bash
northflank put service deployment --projectId speech --noDefaults -o json --file whisper.json
```

What the settings do:

- **The image tag** pins Speaches v0.8.3, the latest stable release at the time of writing. Check the [releases](https://github.com/speaches-ai/speaches/releases) for newer `-cuda` and `-cpu` tags.
- **`ENABLE_UI=false`** removes the Gradio UI that would otherwise sit at `/` without the key check. **`WHISPER__TTL=-1`** (the v0.8.3 name; 0.9 renames it `STT_MODEL_TTL`) keeps a loaded model in GPU memory instead of unloading it after five idle minutes.
- **TCP probes**: in v0.8.3, `/health` is behind the API key too, so an HTTP probe without the token would fail.
- **`timesliced: false`**: the GPU docs say Northflank cloud GPUs don't support timeslicing.

If the volume shows no attachment after the deploy, remove `createOptions`, re-run the `put`, and attach it explicitly (this redeploys the service):

```bash
northflank attach volume --projectId speech --volumeId whisper-models --noDefaults -o json \
  --input '{"nfObject": {"id": "whisper", "type": "service"}}'
```

**CPU fallback.** Change four things and re-run the `put`: `"deploymentPlan": "nf-compute-400-16"` (4 vCPU, 16 GB), `"imagePath": "ghcr.io/speaches-ai/speaches:0.8.3-cpu"`, delete the `gpu` block, and add `"WHISPER__COMPUTE_TYPE": "int8"` to `runtimeEnvironment`. Then use `Systran/faster-whisper-small` below. Expect transcription to run slower than real time for large models on CPU.

## Step 7: Wait for the rollout and read the URL

```bash
OK=0
for i in $(seq 1 90); do
  D=$(northflank get service --projectId speech --serviceId whisper --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId speech --serviceId whisper --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId speech --serviceId whisper --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. Follow start-up with `northflank get service logs --projectId speech --serviceId whisper --lineLimit 100`.

## Step 8: Download the model

```bash
MODEL=Systran/faster-whisper-large-v3     # CPU: Systran/faster-whisper-small
curl -s -w '\n%{http_code}\n' -X POST "https://$HOST/v1/models/$MODEL" -H "Authorization: Bearer $WHISPER_API_KEY"
```

Speaches returns 200 with `Model '…' downloaded`, or 201 if it is already on the volume. The download lands in the cache volume, so it happens once. A large model can take a few minutes; if the request times out at the proxy, wait and re-run it until you get 201.

## How to know it worked

You're done when all of these hold:

1. Requests without the key are rejected, and the model is listed with it:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/v1/models"                       # 401 or 403
   curl -s "https://$HOST/v1/models" -H "Authorization: Bearer $WHISPER_API_KEY" | jq -r '.data[].id'
   ```
2. A sample clip transcribes. This uses the JFK test file from the faster-whisper repository:
   ```bash
   curl -sL -o jfk.flac https://raw.githubusercontent.com/SYSTRAN/faster-whisper/master/tests/data/jfk.flac
   curl -s "https://$HOST/v1/audio/transcriptions" -H "Authorization: Bearer $WHISPER_API_KEY" \
     -F "file=@jfk.flac" -F "model=$MODEL" | jq -r '.text'
   ```
   Expect a sentence starting "And so my fellow Americans, ask not what your country can do for you".
3. On the GPU path, the model runs on CUDA. The logs should mention the `cuda` device when the model loads; `nvidia-smi` also shows the process:
   ```bash
   script -q /dev/null northflank exec service --projectId speech --serviceId whisper --cmd 'nvidia-smi'
   ```
4. The volume is attached: `northflank get volume --projectId speech --volumeId whisper-models --noDefaults -o json | jq '(.data // .).attachedObjects'` lists `whisper`.

Clients use the OpenAI SDK with `base_url="https://$HOST/v1"` and `api_key=$WHISPER_API_KEY`. Services in the same project can call `http://whisper:8000/v1` privately.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Create rejects the plan or GPU | Plan, `gpuType` or region don't match, or the project is free | Re-check the regions list; ask a human about billing; or use the CPU fallback |
| `PermissionError` on `/home/ubuntu/.cache/huggingface/hub` | Volume not writable by UID 1000 | Restart once; if it persists, mount the volume at `/data` and set `HF_HUB_CACHE=/data` in `runtimeEnvironment` |
| `404 Model '…' not found` on download | Model isn't in Speaches' faster-whisper registry | Use a CTranslate2 model, e.g. the `Systran/faster-whisper-*` repos |
| `401`/`403` with the key | Wrong key or secret group not injected | Check `whisper-env`; restart the service |
| CUDA errors or very slow transcription on GPU | `-cpu` image or missing `gpu` block | Use the `-cuda` tag and check `deployment.gpu` |
| Out of memory on CPU | Model too large for the plan | Use `small` or `base`, or a bigger plan |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write: team, region, GPU price per hour, and whether the project is new or reused.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs with `northflank get service logs --projectId speech --serviceId whisper --lineLimit 200` before changing anything.
- Show the GPU price per hour before step 6, and pause the service with `northflank pause service` when it's idle.
- Never print `WHISPER_API_KEY`. Keep `ENABLE_UI=false` unless the user asks for the UI and protects the port (see [port security policies](https://northflank.com/docs/v1/application/network/add-security-policies-for-ports)).
- Audio may contain personal data. Don't send test files other than the sample without the user's say-so.
- Confirm before deleting the volume or the service.

## FAQ

### Where should I deploy Whisper?
On a host that runs a long-lived container with an NVIDIA GPU, persistent storage for the weights and a secret API key behind HTTPS. A GPU VM works if you'll manage drivers and the proxy; a container platform with GPU plans, as in this guide, runs the Speaches image with those pieces attached. For light or batch use, a CPU plan with a small model is enough.

### Can I use vLLM instead?
Yes. vLLM serves Whisper checkpoints on the same `/v1/audio/transcriptions` route. Follow [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md) with an `openai/whisper-*` model ID, and check vLLM's supported models list for your version.

### Which Whisper model should I use?
`large-v3` for accuracy, `distil-large-v3` for speed on English, `small` or `base` on CPU. Add more models with the download call in step 8; each one is loaded on its first request.

### Can I put it behind a key-management proxy?
Yes. Add the endpoint to a [LiteLLM proxy](deploy-litellm-proxy.md) to issue per-team keys and track usage.

## Conclusion

You now have Whisper behind an OpenAI-compatible transcription API on a cloud GPU, with models cached on a volume, an API key in a secret group and a CPU fallback.

**Next:** put a [LiteLLM proxy](deploy-litellm-proxy.md) in front for per-team keys, or pair it with a chat model from [vLLM](deploy-llm-in-production-with-vllm.md) for voice agents. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)
- [How to host Ollama in the cloud](host-ollama-in-the-cloud.md)
- [How to deploy LiteLLM proxy](deploy-litellm-proxy.md)
- [How to deploy ComfyUI](deploy-comfyui.md)
