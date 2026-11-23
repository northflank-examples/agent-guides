<!--
agent-guide-meta:
  id: NF-STK-DEEPSEEK
  description: "How to host DeepSeek on a cloud GPU with vLLM: R1-0528-Qwen3-8B on one A100 with a reasoning parser, a weights volume and API key, then V4-Flash on 8 H200s."
  sources:
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://huggingface.co/deepseek-ai/DeepSeek-R1-0528-Qwen3-8B
    - https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash
    - https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash-0731
    - https://github.com/vllm-project/recipes/blob/main/models/deepseek-ai/DeepSeek-V4-Flash.yaml
    - https://github.com/vllm-project/recipes/blob/main/models/deepseek-ai/DeepSeek-R1.yaml
    - https://github.com/vllm-project/vllm/blob/v0.34.0/vllm/reasoning/__init__.py
    - https://github.com/vllm-project/vllm/blob/v0.34.0/vllm/tool_parsers/__init__.py
    - https://docs.vllm.ai/en/latest/serving/online_serving/openai_compatible_server/
    - https://northflank.com/stacks/deepseek-v3-1
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
    - https://northflank.com/docs/v1/application/gpu-workloads/deploy-gpus-on-northflank-cloud
    - https://northflank.com/docs/v1/application/use-cases/host-a-model-on-gpus
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/volumes/attach-volume
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/miscellaneous/miscellaneous/list-regions
  related: [NF-STK-VLLM, NF-STK-QWEN, NF-STK-GPTOSS, NF-STK-OPENWEBUI, NF-STK-LITELLM]
-->

# How to host DeepSeek

To host DeepSeek, serve a DeepSeek checkpoint from Hugging Face with vLLM's OpenAI-compatible server on an NVIDIA GPU that fits the weights, cache the download on a persistent volume, enable the DeepSeek reasoning parser so the `<think>` trace lands in its own field, and protect `/v1` with an API key. Full V4 models need eight H200-class GPUs, so most teams start with a distilled R1 model on one GPU.

This guide does it on Northflank with the CLI. You deploy `deepseek-ai/DeepSeek-R1-0528-Qwen3-8B` on a single A100 40 GB with a cache volume and a key in a secret group, then see how to switch the same service to `DeepSeek-V4-Flash` on 8 × H200. You end with an HTTPS endpoint verified by `/v1/models` and a chat completion that returns reasoning. Every step is a command a developer or coding agent (Claude Code, Codex, Cursor) can run. It needs GPU billing.

## TL;DR: how to host DeepSeek

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check which regions offer `a100-40` with the public `curl -s https://api.northflank.com/v1/regions`, and confirm the team, region, hourly price and project before creating anything.
3. Create the project with `northflank create project`, store a generated `VLLM_API_KEY` in a secret group with `northflank put secret` (DeepSeek weights are ungated, so no Hugging Face token is needed), and create a 40 GB volume for `/root/.cache/huggingface` with `northflank create volume`.
4. Deploy `vllm/vllm-openai:v0.34.0` with `northflank put service deployment` on `nf-gpu-a100-40-1g`, serving `deepseek-ai/DeepSeek-R1-0528-Qwen3-8B` with `--reasoning-parser deepseek_r1` and a startup probe on `/health`.
5. Verify with an authenticated `GET /v1/models` and a `POST /v1/chat/completions` whose response has both `reasoning` and `content`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup, and which DeepSeek model?

"DeepSeek" covers models from 8 billion to 1.6 trillion parameters, and the right GPU follows directly from that number:

| Model (Hugging Face ID) | Size | What it needs | Licence |
|---|---|---|---|
| `deepseek-ai/DeepSeek-R1-0528-Qwen3-8B` | 8.2B dense, BF16, about 16 GB of weights | One A100 40 GB (or one L4 with a short context) | MIT |
| `deepseek-ai/DeepSeek-V4-Flash-0731` | 284B MoE, 13B active, FP4 + FP8, about 166 GB | The vLLM recipe sets a 200 GB VRAM minimum and verifies H200, B200 and 8 × RTX PRO 6000 | MIT |
| `deepseek-ai/DeepSeek-V4-Pro` | 1.6T MoE, 49B active | Multi-GPU Blackwell or multi-node | MIT |

`DeepSeek-R1-0528-Qwen3-8B` is R1-0528's chain-of-thought distilled into Qwen3-8B, with DeepSeek-R1-0528's tokenizer, so load it from the `deepseek-ai` repo. On an A100 40 GB, vLLM's default 90% memory budget leaves roughly 20 GB for the KV cache after the 16 GB of weights, enough for `--max-model-len 32768` with several requests in flight. All three are MIT-licensed and ungated, so no access request is needed.

Where to host DeepSeek follows from those sizes: a long-running container with an NVIDIA GPU, persistent storage for the weights and a protected HTTPS port. Serverless and CPU hosts can't hold the weights; a GPU VM works if you supervise vLLM yourself; a container platform with GPU plans runs the vLLM image with the GPU, volume and secret attached. If you'd rather click through it, Northflank's [DeepSeek V3.1 stack](https://northflank.com/stacks/deepseek-v3-1) deploys the older 671B V3.1 with vLLM on 8 × H200 plus Open WebUI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **`openssl`** (to generate the API key).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **GPU access.** Free projects can't run GPUs, and GPU deployments need account credit.

Names used below: project `deepseek`, secret group `deepseek-secrets`, volume `deepseek-cache`, service `deepseek-r1`.

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

Before creating anything, print the team, whether the project exists, and the regions that offer one NVIDIA A100 40 GB with its on-demand price in US cents per GPU-hour. The regions endpoint is public, so it needs no token:

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
if P=$(northflank get project --projectId deepseek -o json 2>/dev/null); then
  echo "project: deepseek exists in $(echo "$P" | jq -r '(.data // .).deployment.region') and will be reused"
else
  echo "project: deepseek will be created"
fi

GPU=a100-40; COUNT=1
curl -s https://api.northflank.com/v1/regions | jq -r --arg g "$GPU" --argjson n "$COUNT" '
  .data.regions[] | select(any(.gpuDevices[]?; .id == $g and (.countOptions | index($n))))
  | "\(.id)\t\(.gpuDevices[] | select(.id == $g) | .pricing.onDemand) cents/h"'
```

- If the GPU list is empty, no region offers 1 × `a100-40` right now. Pick another GPU (the `.data.regions[].gpuDevices[].id` values in the same response) and change `GPU`.
- If the project already exists, its region must be in the GPU list, because a project's region can't be changed. If it isn't, use a new project name throughout.
- **GPUs need a human.** Free projects can't run GPU workloads, and GPU deployments need account credit. Ask the account owner to confirm billing and the hourly price before you create anything.
- At the time of writing, `a100-40` is offered in `us-central`, `europe-west-netherlands`, `asia-southeast` and `asia-northeast`. GPU plan IDs aren't listed by `/v1/plans`; the resource plans docs give the rule `nf-gpu-<gpuType>-<gpuCount>g`, so one A100 40 GB is `nf-gpu-a100-40-1g`.

**If you're an agent:** show the user the team, the region (`us-central` below, unless they choose another from the GPU list), the GPU's hourly price and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services, and every service in it runs in the project's region. The region can't be changed later. Skip this step if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "deepseek", "region": "us-central", "description": "Self-hosted DeepSeek" }'
```

To use another region from the step 2 list, change `region`. REST: `POST https://api.northflank.com/v1/projects`. If you plan to move to V4-Flash on H200s, pick a region that also lists `h200-141` (`us-central` does; check with `GPU=h200-141; COUNT=8` in step 2).

## Step 4: Put the API key in a secret group

vLLM checks `VLLM_API_KEY` on `/v1` routes. Generate it and send it without echoing it:

```bash
export VLLM_API_KEY=$(openssl rand -hex 32)
jq -n --arg k "$VLLM_API_KEY" '{
  name: "deepseek-secrets", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "deepseek-r1", type: "service" } ] },
  secrets: { variables: { VLLM_API_KEY: $k } }
}' > deepseek-secrets.json
northflank put secret --projectId deepseek --noDefaults -o json --file deepseek-secrets.json >/dev/null && rm deepseek-secrets.json
```

If the API refuses a restriction to a service that doesn't exist yet, re-run with `restrictions: { restricted: false }`.

## Step 5: Create the weights volume

```bash
northflank create volume --projectId deepseek --noDefaults -o json --input '{
  "name": "deepseek-cache",
  "mounts": [ { "containerMountPath": "/root/.cache/huggingface" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 40960 }
}'
```

The weights are about 16 GB; 40 GB (`storageSize` is in MB, and can only grow) leaves room for a second checkpoint.

## Step 6: Deploy vLLM with the DeepSeek reasoning parser

Save this as `deepseek-r1.json`:

```json
{
  "name": "deepseek-r1",
  "billing": { "deploymentPlan": "nf-gpu-a100-40-1g" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "vllm/vllm-openai:v0.34.0" },
    "docker": { "configType": "customCommand", "customCommand": "deepseek-ai/DeepSeek-R1-0528-Qwen3-8B --host 0.0.0.0 --port 8000 --served-model-name deepseek-r1 --max-model-len 32768 --reasoning-parser deepseek_r1" },
    "gpu": { "enabled": true, "configuration": { "gpuType": "a100-40", "gpuCount": 1, "timesliced": false } }
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
  "createOptions": { "volumesToAttach": ["deepseek-cache"] }
}
```

```bash
northflank put service deployment --projectId deepseek --noDefaults -o json --file deepseek-r1.json
```

The image's entrypoint is `vllm serve`, and the custom command supplies its arguments without a shell, so keep quotes out of it.

- **`--reasoning-parser deepseek_r1`** is the parser the vLLM DeepSeek-R1 recipe uses. It moves the text between `<think>` and `</think>` into `message.reasoning` (vLLM v0.34.0 field name), so clients that only read `content` get the final answer.
- **`--served-model-name deepseek-r1`** gives clients a stable name.
- **`--max-model-len 32768`** caps prompt plus output; R1-style traces are long, so don't go much below 16,384.
- **The startup probe** allows about 30 minutes for the first download and load.

If `volumesToAttach` is rejected or the volume shows no attachment, drop that line, re-run the `put`, then attach it (this redeploys):

```bash
northflank attach volume --projectId deepseek --volumeId deepseek-cache --noDefaults -o json \
  --input '{"nfObject": {"id": "deepseek-r1", "type": "service"}}'
```

## Step 7: Wait for the model to load

```bash
OK=0
until D=$(northflank get service --projectId deepseek --serviceId deepseek-r1 --noDefaults -o json \
      | jq -r '(.data // .).status.deployment.status // "NONE"'); echo "$D"; [ "$D" = COMPLETED ] || [ "$D" = FAILED ]; do
  sleep 10
done
[ "$D" = COMPLETED ] && OK=1
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId deepseek --serviceId deepseek-r1 --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId deepseek --serviceId deepseek-r1 --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means the startup probe passed and vLLM is serving. On `FAILED`, read `northflank get service logs --projectId deepseek --serviceId deepseek-r1 --lineLimit 200`.

## How to know it worked

1. The key is enforced and the model is listed under its served name:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/v1/models"     # 401
   curl -s "https://$HOST/v1/models" -H "Authorization: Bearer $VLLM_API_KEY" | jq -r '.data[].id'   # deepseek-r1
   ```
2. A chat completion returns a reasoning trace and an answer. DeepSeek's card uses temperature 0.6 for R1 models:
   ```bash
   curl -s "https://$HOST/v1/chat/completions" -H "Authorization: Bearer $VLLM_API_KEY" -H "Content-Type: application/json" \
     -d '{"model": "deepseek-r1", "temperature": 0.6, "max_tokens": 2048,
          "messages": [{"role": "user", "content": "What is 17 * 23? Answer with the number only."}]}' \
     | jq '{reasoning: (.choices[0].message.reasoning // "" | .[0:200]), answer: .choices[0].message.content}'
   ```
   Expect a non-empty `reasoning` and `"391"` in `answer`.
3. `northflank get volume --projectId deepseek --volumeId deepseek-cache --noDefaults -o json | jq '(.data // .).attachedObjects'` lists `deepseek-r1`, and a restart doesn't trigger a new download.

Point clients at `base_url = https://$HOST/v1` with `api_key = $VLLM_API_KEY`. Other services in the project can use `http://deepseek-r1:8000/v1`.

## Scale to the full DeepSeek-V4-Flash

When the distilled model isn't enough, switch the same service to V4-Flash. The vLLM recipe for `deepseek-ai/DeepSeek-V4-Flash` uses the `DeepSeek-V4-Flash-0731` checkpoint (about 166 GB, FP4 experts and FP8 elsewhere), a 200 GB VRAM minimum, base flags `--trust-remote-code --kv-cache-dtype fp8 --block-size 256`, the `deepseek_v4` tokenizer mode, tool-call parser and reasoning parser, and tensor plus expert parallelism across eight GPUs. Northflank offers H200 only in 8-GPU plans, which matches:

```bash
northflank update volume --projectId deepseek --volumeId deepseek-cache --noDefaults -o json \
  --input '{"spec": {"storageSize": 409600}}'
northflank patch service deployment --projectId deepseek --serviceId deepseek-r1 --noDefaults -o json --input '{
  "billing": { "deploymentPlan": "nf-gpu-h200-141-8g" },
  "deployment": {
    "instances": 1,
    "docker": { "configType": "customCommand", "customCommand": "deepseek-ai/DeepSeek-V4-Flash-0731 --host 0.0.0.0 --port 8000 --served-model-name deepseek-v4-flash --trust-remote-code --kv-cache-dtype fp8 --block-size 256 --tensor-parallel-size 8 --enable-expert-parallel --tokenizer-mode deepseek_v4 --enable-auto-tool-choice --tool-call-parser deepseek_v4 --reasoning-parser deepseek_v4" },
    "gpu": { "enabled": true, "configuration": { "gpuType": "h200-141", "gpuCount": 8, "timesliced": false } }
  }
}'
```

This only works if the project's region offers `h200-141`, and it costs eight times the per-GPU price, so get the user's approval first. The recipe also lists speculative decoding and a `--reasoning-config` JSON; add them once you've confirmed how the command string handles JSON. DeepSeek's 0731 card recommends `temperature = 1.0`, with `top_p = 0.95` for agentic use and `1.0` otherwise. The newer `DeepSeek-V4.1-Flash` has its own recipe and `deepseek_v41` parsers.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Plan or GPU rejected | Region has no `a100-40`, or the project is on the free tier | Re-run the regions query; ask a human about billing |
| CUDA out of memory on start | Context too long for what's left after the weights | Lower `--max-model-len` to 16384, or use `a100-80` |
| `reasoning` is empty and `content` has `<think>` | Reasoning parser missing | Check `--reasoning-parser deepseek_r1` is in the command |
| V4-Flash fails with an unknown architecture | Image older than the recipe's minimum (0.25.0 for the 0731 checkpoint) | Keep `vllm/vllm-openai:v0.34.0` or newer |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're running these steps for a user:

- Confirm the target from step 2 with the user before any write: team, region, GPU price per hour, and whether the project is new or reused.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs with `northflank get service logs --projectId deepseek --serviceId deepseek-r1 --lineLimit 200` before changing anything.
- Read the per-GPU price from `/v1/regions` and tell the user the hourly cost before step 6, and again before any switch to 8 × H200.
- Never print `VLLM_API_KEY` or write it into the custom command.
- `/health` and `/metrics` aren't behind the key. For private use, set `public: false` and call `http://deepseek-r1:8000` from inside the project.
- Pause with `northflank pause service --projectId deepseek --serviceId deepseek-r1` when the user is done testing. Confirm before deleting the volume.

## FAQ

### Where should I host DeepSeek?
On infrastructure that gives a long-running container an NVIDIA GPU with enough memory for the checkpoint, persistent storage for the weights and a protected HTTPS endpoint. For distilled R1 models that's one A100 or L4; for V4-Flash it's a full 8 × H200 node. A container platform with GPU plans, as here, runs the vLLM image with those pieces attached.

### Can I host DeepSeek R1 671B instead?
Yes, but the vLLM recipe for `deepseek-ai/DeepSeek-R1` lists an 805 GB VRAM minimum, which an 8 × H200 node (1,128 GB) meets but most smaller nodes don't. V4-Flash, at a 200 GB minimum, fits the same node with far more room for the KV cache.

### Is it safe to use DeepSeek weights commercially?
The model cards for R1-0528 and V4 state the MIT License and say the R1 series "supports commercial use and distillation". Self-hosting also means prompts never leave your infrastructure. Review the licence with your own counsel for regulated use.

## Conclusion

You now host a DeepSeek reasoning model behind an OpenAI-compatible API on your own GPU, with weights cached on a volume, the key in a secret group and a clear path to V4-Flash when you need it.

**Next:** put a [LiteLLM proxy](deploy-litellm-proxy.md) in front for per-team keys, or compare with [Qwen](deploy-qwen.md) and [gpt-oss](host-gpt-oss.md) on the same setup. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)
- [How to deploy Qwen](deploy-qwen.md)
- [How to host gpt-oss](host-gpt-oss.md)
- [How to deploy LiteLLM proxy](deploy-litellm-proxy.md)
- [How to deploy Open WebUI with Ollama](deploy-open-webui-with-ollama.md)
