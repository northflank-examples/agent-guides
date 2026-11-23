<!--
agent-guide-meta:
  id: NF-STK-MISTRAL
  description: "How to self-host Mistral with vLLM: Ministral 3 14B in native FP8 on one L4 with Mistral-format loading, tool calling, a weights volume and an API key."
  sources:
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://docs.mistral.ai/deployment/self-deployment/vllm/
    - https://huggingface.co/mistralai/Ministral-3-14B-Instruct-2512
    - https://huggingface.co/mistralai/Mistral-Small-4-119B-2603
    - https://huggingface.co/api/models/mistralai/Ministral-3-14B-Instruct-2512
    - https://github.com/vllm-project/recipes/blob/main/models/mistralai/Ministral-3-14B-Instruct-2512.yaml
    - https://github.com/vllm-project/recipes/blob/main/models/mistralai/Mistral-Small-4-119B-2603.yaml
    - https://docs.vllm.ai/en/latest/features/tool_calling/
    - https://github.com/vllm-project/vllm/blob/v0.34.0/vllm/tool_parsers/__init__.py
    - https://github.com/vllm-project/vllm/blob/v0.34.0/vllm/model_executor/model_loader/default_loader.py
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
    - https://northflank.com/docs/v1/application/gpu-workloads/deploy-gpus-on-northflank-cloud
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/volumes/attach-volume
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/miscellaneous/miscellaneous/list-regions
  related: [NF-STK-VLLM, NF-STK-LLAMA, NF-STK-GPTOSS, NF-STK-LITELLM, NF-STK-OPENWEBUI]
-->

# How to self-host Mistral

To self-host Mistral, run an open-weights Mistral model on vLLM, the engine Mistral's own self-deployment docs point to, with `--tokenizer_mode mistral --config_format mistral --load_format mistral` so it reads Mistral's native files, on an NVIDIA GPU sized for the weights, with a persistent model cache and an API key. Recent vLLM releases already prefer those native files under the default `auto` modes; setting the flags explicitly matches the model card and pins that choice, which keeps Mistral's own tokenizer, chat template and tool-call format in use.

This guide does it on Northflank with the CLI: `mistralai/Ministral-3-14B-Instruct-2512`, an Apache 2.0 model shipped in FP8, on one L4 GPU with the Mistral tool-call parser, a cache volume and a key in a secret group. You end with a private-by-key HTTPS endpoint checked by `/v1/models`, a chat completion and a tool call, plus the patch that moves to Mistral Small 4. Every step is a command a developer or coding agent (Claude Code, Codex, Cursor) can run.

## TL;DR: how to self-host Mistral

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Find the regions with `l4-24` GPUs with the public `curl -s https://api.northflank.com/v1/regions`, and confirm the team, region, hourly price and project before creating anything.
3. Create the project with `northflank create project`, put a generated `VLLM_API_KEY` (and `HF_TOKEN` if you use a gated Mistral repo) in a secret group with `northflank put secret`, and create a 40 GB cache volume at `/root/.cache/huggingface` with `northflank create volume`.
4. Deploy `vllm/vllm-openai:v0.34.0` on `nf-gpu-l4-24-1g` with `northflank put service deployment`, running Ministral 3 14B with the three Mistral-format flags, `--max-model-len 16384` and `--tool-call-parser mistral`.
5. Confirm `GET /v1/models`, a chat completion and a tool call respond with `Authorization: Bearer $VLLM_API_KEY`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

**Why vLLM.** Mistral's self-deployment docs describe vLLM as "particularly appropriate as a target platform for deploying Mistral models", and the Ministral 3 card says "We recommend using this model with vLLM". Their examples run the `vllm/vllm-openai` image with the three `mistral` format flags, which load `params.json`, `tekken.json` and `consolidated.safetensors` instead of the Hugging Face conversions.

**Which Mistral.** Mistral publishes a wide range on Hugging Face. The ones that matter for a text endpoint:

| Model | Shape | Licence | Serving need (vLLM recipe) |
|---|---|---|---|
| `Ministral-3-14B-Instruct-2512` | 14B dense, FP8, vision, 256K context | Apache 2.0 | 17 GB VRAM minimum; card: fits 24 GB in FP8 |
| `Ministral-3-8B-Instruct-2512` / `3B` | Smaller siblings | Apache 2.0 | 12 GB / 6 GB minimum |
| `Mistral-Small-4-119B-2603` | 119B MoE, 6.5B active, FP8, reasoning | Apache 2.0 | 143 GB minimum |
| `Mistral-Medium-3.5-128B`, `Mistral-Large-4-1T-A52B` | Large dense and MoE | Not Apache 2.0 (marked `other`); read the card | Multi-GPU |

Ministral 3 14B is the largest that fits one 24 GB L4. Its weights are about 15.7 GB, leaving a few gigabytes for the KV cache (about 160 KB per token in BF16), which is why this guide caps the context at 16,384 tokens and loads the language model only. On an 80 GB GPU you can raise the context towards 262,144.

**Hugging Face access.** Mistral's docs say its repos require a read `HF_TOKEN` and accepting access conditions. At the time of writing the Ministral 3 and Mistral Small 4 repos report `gated: false` on the Hugging Face API, so no token is needed for them; older or larger repos may differ.

Where to self-host Mistral follows from those numbers: a long-running container with an NVIDIA GPU (FP8 needs Ada, Hopper or newer to run natively), disk that keeps the weights between restarts and a protected HTTPS port. A GPU VM or your own cluster works if you'll operate vLLM yourself; a container platform with GPU plans runs it with the volume and secret attached.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **`openssl`** (to generate the API key).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **GPU access.** Free projects can't run GPUs, and GPU deployments need account credit.
- **Optional:** `HF_TOKEN` exported in your shell if you pick a gated Mistral repo. **This needs a human** to accept that repo's conditions and create the token.

Names used below: project `mistral`, secret group `mistral-secrets`, volume `mistral-cache`, service `ministral`.

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
if P=$(northflank get project --projectId mistral -o json 2>/dev/null); then
  echo "project: mistral exists in $(echo "$P" | jq -r '(.data // .).deployment.region') and will be reused"
else
  echo "project: mistral will be created"
fi

GPU=l4-24; COUNT=1
curl -s https://api.northflank.com/v1/regions | jq -r --arg g "$GPU" --argjson n "$COUNT" '
  .data.regions[] | select(any(.gpuDevices[]?; .id == $g and (.countOptions | index($n))))
  | "\(.id)\t\(.gpuDevices[] | select(.id == $g) | .pricing.onDemand) cents/h"'
```

- If the GPU list is empty, no region offers 1 × `l4-24` right now. Pick another GPU (the `.data.regions[].gpuDevices[].id` values in the same response) and change `GPU`.
- If the project already exists, its region must be in the GPU list, because a project's region can't be changed. If it isn't, use a new project name throughout.
- **GPUs need a human.** Free projects can't run GPU workloads, and GPU deployments need account credit. Ask the account owner to confirm billing and the hourly price before you create anything.
- The plan for one L4 is `nf-gpu-l4-24-1g` (the docs' `nf-gpu-<gpuType>-<gpuCount>g` rule; `/v1/plans` doesn't list GPU plans).

**If you're an agent:** show the user the team, the region (`europe-west` below, unless they choose another from the GPU list), the GPU's hourly price and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services, and every service in it runs in the project's region. The region can't be changed later. Skip this step if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "mistral", "region": "europe-west", "description": "Self-hosted Mistral models" }'
```

To use another region from the step 2 list, change `region`. REST: `POST https://api.northflank.com/v1/projects`. `europe-west` also offers H100s, which the Mistral Small 4 upgrade below needs.

## Step 4: Add the secret group

```bash
export VLLM_API_KEY=$(openssl rand -hex 32)
umask 077
jq -n --arg k "$VLLM_API_KEY" --arg hf "${HF_TOKEN:-}" '{
  name: "mistral-secrets", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "ministral", type: "service" } ] },
  secrets: { variables: ({ VLLM_API_KEY: $k } + (if $hf == "" then {} else { HF_TOKEN: $hf } end)) }
}' > mistral-secrets.json
northflank put secret --projectId mistral --noDefaults -o json --file mistral-secrets.json >/dev/null
rm -f mistral-secrets.json
```

`HF_TOKEN` is only added when it's set. If the API won't restrict the group to a service that doesn't exist yet, re-run with `restricted: false`.

## Step 5: Create the cache volume

```bash
northflank create volume --projectId mistral --noDefaults -o json --input '{
  "name": "mistral-cache",
  "mounts": [ { "containerMountPath": "/root/.cache/huggingface" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 40960 }
}'
```

The repo holds the weights twice, as `consolidated.safetensors` (15.7 GB) and as Hugging Face shards, but with `--load_format mistral` vLLM downloads only `consolidated*.safetensors`. 40 GB leaves room for a second checkpoint. Size is in MB and can only go up.

## Step 6: Deploy Ministral 3 14B

Save as `ministral.json`:

```json
{
  "name": "ministral",
  "billing": { "deploymentPlan": "nf-gpu-l4-24-1g" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "vllm/vllm-openai:v0.34.0" },
    "docker": { "configType": "customCommand", "customCommand": "mistralai/Ministral-3-14B-Instruct-2512 --host 0.0.0.0 --port 8000 --served-model-name ministral-3-14b --tokenizer_mode mistral --config_format mistral --load_format mistral --max-model-len 16384 --language-model-only --enable-auto-tool-choice --tool-call-parser mistral" },
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
  "createOptions": { "volumesToAttach": ["mistral-cache"] }
}
```

```bash
northflank put service deployment --projectId mistral --noDefaults -o json --file ministral.json
```

What each part of the command does:

- The first word is the repo; everything is passed to the image's `vllm serve` entrypoint without a shell, so there are no quotes.
- `--tokenizer_mode mistral --config_format mistral --load_format mistral` match the card's launch command. vLLM's tool-calling docs call this "the official Mistral AI's format" and pair it with `--tool-call-parser mistral`.
- `--language-model-only` (from the recipe's text-only option) skips the vision encoder, leaving more memory for the KV cache. Remove it to accept images.
- `--max-model-len 16384` keeps the KV cache inside the L4's spare memory. Adding `--kv-cache-dtype fp8` roughly halves the KV footprint if you need more context.

If the volume isn't attached after the first create, attach it (this redeploys):

```bash
northflank attach volume --projectId mistral --volumeId mistral-cache --noDefaults -o json \
  --input '{"nfObject": {"id": "ministral", "type": "service"}}'
```

## Step 7: Wait for vLLM to come up

```bash
S=NONE; while [ "$S" != COMPLETED ] && [ "$S" != FAILED ]; do
  sleep 10
  S=$(northflank get service --projectId mistral --serviceId ministral --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "status: $S"
done
OK=0; [ "$S" = COMPLETED ] && OK=1
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId mistral --serviceId ministral --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId mistral --serviceId ministral --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

The startup probe gives the download and load about 30 minutes. Logs: `northflank get service logs --projectId mistral --serviceId ministral --lineLimit 100`.

## How to know it worked

1. **Auth and model name.**
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/v1/models"                       # 401
   curl -s "https://$HOST/v1/models" -H "Authorization: Bearer $VLLM_API_KEY" | jq -r '.data[].id'   # ministral-3-14b
   ```
2. **A chat answer.** Mistral recommends a temperature below 0.1 for production use:
   ```bash
   curl -s "https://$HOST/v1/chat/completions" -H "Authorization: Bearer $VLLM_API_KEY" -H "Content-Type: application/json" -d '{
     "model": "ministral-3-14b", "temperature": 0.05, "max_tokens": 80,
     "messages": [{"role": "system", "content": "You are a concise assistant for an internal help desk."},
                  {"role": "user", "content": "In one sentence, what does a reverse proxy do?"}]}' | jq -r '.choices[0].message.content'
   ```
3. **A tool call.** Expect `convert_currency` with `amount`, `from` and `to`:
   ```bash
   curl -s "https://$HOST/v1/chat/completions" -H "Authorization: Bearer $VLLM_API_KEY" -H "Content-Type: application/json" -d '{
     "model": "ministral-3-14b", "temperature": 0.05, "max_tokens": 200,
     "tools": [{"type": "function", "function": {"name": "convert_currency", "parameters": {"type": "object",
       "properties": {"amount": {"type": "number"}, "from": {"type": "string"}, "to": {"type": "string"}},
       "required": ["amount", "from", "to"]}}}],
     "messages": [{"role": "user", "content": "Convert 250 EUR to USD."}]}' | jq '.choices[0].message.tool_calls[0].function'
   ```

Clients use `https://$HOST/v1` as the base URL, or `http://ministral:8000/v1` from inside the project.

## Moving to Mistral Small 4

`mistralai/Mistral-Small-4-119B-2603` is an Apache 2.0 MoE (6.5B active) with optional reasoning. The vLLM recipe passes no format flags (vLLM's `auto` defaults then load the repo's Mistral-native `consolidated-*` files, about 121 GB), sets a 143 GB minimum and enables `--reasoning-parser mistral` alongside the `mistral` tool parser; reasoning appears in `message.reasoning` when a request sets `reasoning_effort` to `high`. Two H100s (160 GB) meet the minimum, so cap the context:

```bash
northflank update volume --projectId mistral --volumeId mistral-cache --noDefaults -o json --input '{"spec": {"storageSize": 307200}}'
northflank patch service deployment --projectId mistral --serviceId ministral --noDefaults -o json --input '{
  "billing": { "deploymentPlan": "nf-gpu-h100-80-2g" },
  "deployment": { "instances": 1,
    "docker": { "configType": "customCommand", "customCommand": "mistralai/Mistral-Small-4-119B-2603 --host 0.0.0.0 --port 8000 --served-model-name mistral-small-4 --tensor-parallel-size 2 --max-model-len 65536 --enable-auto-tool-choice --tool-call-parser mistral --reasoning-parser mistral" },
    "gpu": { "enabled": true, "configuration": { "gpuType": "h100-80", "gpuCount": 2, "timesliced": false } } } }'
```

For more context or concurrency, use four H100s with `--tensor-parallel-size 4`. The recipe needs `mistral_common` 1.11.0 or later, which it says vLLM 0.20.1+ bundles.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Garbled output or template errors | Model loaded through the Hugging Face conversion (for example `hf` format flags) | Use all three `--*_mode/--*_format mistral` flags for Ministral |
| CUDA out of memory on the L4 | Context too large for the leftover memory | Lower to `--max-model-len 8192`, add `--kv-cache-dtype fp8`, or use `h100-80` |
| `401` from Hugging Face in logs | Gated repo without `HF_TOKEN` | Human step: accept conditions, add `HF_TOKEN` to `mistral-secrets`, restart |
| Tool call not returned for a required tool | Mistral format doesn't guarantee calls for `required` or named choices | Use `tool_choice: "auto"` and a clear system prompt |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

When you run this for someone:

- Confirm the target from step 2 with the user before any write: team, region, GPU price per hour, and whether the project is new or reused.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs with `northflank get service logs --projectId mistral --serviceId ministral --lineLimit 200` before changing anything.
- Show the hourly GPU price before deploying and before the Mistral Small 4 patch.
- Don't print `VLLM_API_KEY` or `HF_TOKEN`; don't put either in the command string.
- `/health` and `/metrics` bypass the key. Restrict callers with [port security policies](https://northflank.com/docs/v1/application/network/add-security-policies-for-ports) or make the port private.
- Check the licence on any model other than Ministral 3 or Small 4 before deploying it for commercial use.
- Ask before deleting the volume; pause the service with `northflank pause service` when idle.

## FAQ

### Can you self-host Mistral models?
Yes. Mistral publishes open weights on Hugging Face, many under Apache 2.0, and documents vLLM as the self-deployment path.

### Where should I host Mistral?
On a cloud host that gives a long-running container an NVIDIA GPU (an L4 for Ministral 3 14B with a short context, H100s for Mistral Small 4), persistent storage for the weights and a secret store for the API key. A container platform with GPU plans, as used here, handles all three.

### Which Mistral model should I start with?
Ministral 3 14B for a single cheap GPU, Ministral 3 8B or 3B if latency matters more than quality, and Mistral Small 4 when you need reasoning and stronger tool use.

## Conclusion

You're self-hosting a Mistral model behind an OpenAI-compatible API: Ministral 3 14B in FP8 on an L4, loaded in Mistral's native format with tool calling, its weights cached on a volume and the endpoint protected by a key.

**Next:** front it with a [LiteLLM proxy](deploy-litellm-proxy.md) for per-team keys, or run [gpt-oss](host-gpt-oss.md) alongside it. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)
- [How to deploy Llama](deploy-llama.md)
- [How to host gpt-oss](host-gpt-oss.md)
- [How to deploy LiteLLM proxy](deploy-litellm-proxy.md)
- [How to deploy Open WebUI with Ollama](deploy-open-webui-with-ollama.md)
