<!--
agent-guide-meta:
  id: NF-STK-QWEN
  description: "How to deploy Qwen with vLLM on a cloud H100: Qwen3.8-27B in FP8 with thinking and tool-call parsers, a weights volume, an API key and a verified endpoint."
  sources:
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://huggingface.co/Qwen/Qwen3.8-27B-FP8
    - https://huggingface.co/Qwen/Qwen3.8-27B
    - https://huggingface.co/Qwen/Qwen3.8-Flash-Next-FP8
    - https://github.com/vllm-project/recipes/blob/main/models/Qwen/Qwen3.8-27B.yaml
    - https://github.com/vllm-project/recipes/blob/main/models/Qwen/Qwen3.8-Flash-Next.yaml
    - https://github.com/vllm-project/vllm/blob/v0.34.0/vllm/tool_parsers/__init__.py
    - https://github.com/vllm-project/vllm/blob/v0.34.0/vllm/reasoning/__init__.py
    - https://docs.vllm.ai/en/latest/features/tool_calling/
    - https://northflank.com/stacks/deploy-qwen3-30b-instruct-32k
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
    - https://northflank.com/docs/v1/application/gpu-workloads/deploy-gpus-on-northflank-cloud
    - https://northflank.com/docs/v1/application/gpu-workloads/configure-and-optimise-workloads-for-gpus
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/volumes/attach-volume
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/miscellaneous/miscellaneous/list-regions
  related: [NF-STK-VLLM, NF-STK-DEEPSEEK, NF-STK-LLAMA, NF-STK-LITELLM, NF-STK-OPENWEBUI]
-->

# How to deploy Qwen

To deploy Qwen, run a Qwen3.8 checkpoint from Hugging Face on vLLM's OpenAI-compatible server with an NVIDIA GPU large enough for the weights, enable the `qwen3` reasoning parser and `qwen3_coder` tool-call parser, mount a volume for the model cache and set an API key. Qwen3.8 thinks by default, so without the reasoning parser its `<think>` block ends up inside the answer text.

This guide does it on Northflank with the CLI: `Qwen/Qwen3.8-27B-FP8` on one H100 80 GB, weights on a volume, the key in a secret group and a 30-minute startup window. You end with a public HTTPS endpoint verified by `/v1/models`, a chat completion with thinking switched off and a tool call. Every step is a copy-pasteable command for a developer or coding agent (Claude Code, Codex, Cursor). It needs a GPU-enabled account.

## TL;DR: how to deploy Qwen

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. List the regions that offer single H100s with the public `curl -s https://api.northflank.com/v1/regions`, and confirm the team, region, hourly price and project before creating anything.
3. Create the project with `northflank create project`, save a generated `VLLM_API_KEY` to a secret group with `northflank put secret`, and create a 60 GB volume at `/root/.cache/huggingface` with `northflank create volume`.
4. Deploy `vllm/vllm-openai:v0.34.0` on `nf-gpu-h100-80-1g` with `northflank put service deployment`, serving `Qwen/Qwen3.8-27B-FP8` with `--reasoning-parser qwen3 --enable-auto-tool-choice --tool-call-parser qwen3_coder`.
5. Check `GET /v1/models`, a non-thinking chat completion and a tool call, all with `Authorization: Bearer $VLLM_API_KEY`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Qwen3.8 is the current generation of Alibaba's open models on Hugging Face. The three text checkpoints differ by two orders of magnitude:

- **`Qwen/Qwen3.8-27B`**: a dense 27B vision-language model with a 262,144-token native context (extensible to 1M with YaRN). Apache 2.0. The vLLM recipe puts BF16 weights at 51.7 GiB (67 GB VRAM minimum) and the official FP8 checkpoint, `Qwen/Qwen3.8-27B-FP8`, at 28.7 GiB (38 GB minimum).
- **`Qwen/Qwen3.8-Flash-Next`**: a 176B MoE with 6B active parameters, released under the Qwen Community License (`qwen-community-1.0`), not Apache 2.0.
- **`Qwen/Qwen3.8-2.4T-A95B`**: a 2.4T MoE for multi-node clusters.

The FP8 27B model is the sweet spot for one GPU: on an 80 GB H100 it leaves about 40 GB for the KV cache, and only 16 of its 64 layers use full attention, so long contexts stay cheap. The recipe lists the H100 among the FP8 checkpoint's supported GPUs.

Each vLLM flag here comes from the Qwen3.8 recipe:

- `--reasoning-parser qwen3` puts the thinking text in `message.reasoning` instead of `message.content`.
- `--enable-auto-tool-choice --tool-call-parser qwen3_coder` turns the model's tool-call markup into OpenAI `tool_calls`.
- `--language-model-only` skips the vision encoder for text-only workloads, freeing memory for the KV cache. Drop it if you want image input.

Where to deploy Qwen follows from the weights: an NVIDIA GPU with at least 38 GB for the FP8 27B model, a disk cache that survives restarts, and a long-running HTTPS service. A GPU VM or your own Kubernetes cluster works if you'll run vLLM and its upgrades yourself; a container platform with GPU plans runs the image with these pieces attached. Northflank also has one-click Qwen stack templates, such as [Qwen3 30B Instruct (32K)](https://northflank.com/stacks/deploy-qwen3-30b-instruct-32k), which deploys the earlier Qwen3-30B-A3B with vLLM and Open WebUI on one H100.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **`openssl`** (to generate the API key).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **GPU access.** Free projects can't run GPUs, and GPU deployments need account credit.
- **No Hugging Face token is needed.** The Qwen repos are public and ungated.

Names used below: project `qwen`, secret group `qwen-secrets`, volume `qwen-models`, service `qwen`.

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

Before creating anything, print the team, whether the project exists, and the regions that offer one NVIDIA H100 80 GB with its on-demand price in US cents per GPU-hour. The regions endpoint is public, so it needs no token:

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
if P=$(northflank get project --projectId qwen -o json 2>/dev/null); then
  echo "project: qwen exists in $(echo "$P" | jq -r '(.data // .).deployment.region') and will be reused"
else
  echo "project: qwen will be created"
fi

GPU=h100-80; COUNT=1
curl -s https://api.northflank.com/v1/regions | jq -r --arg g "$GPU" --argjson n "$COUNT" '
  .data.regions[] | select(any(.gpuDevices[]?; .id == $g and (.countOptions | index($n))))
  | "\(.id)\t\(.gpuDevices[] | select(.id == $g) | .pricing.onDemand) cents/h"'
```

- If the GPU list is empty, no region offers 1 × `h100-80` right now. Pick another GPU (the `.data.regions[].gpuDevices[].id` values in the same response) and change `GPU`.
- If the project already exists, its region must be in the GPU list, because a project's region can't be changed. If it isn't, use a new project name throughout.
- **GPUs need a human.** Free projects can't run GPU workloads, and GPU deployments need account credit. Ask the account owner to confirm billing and the hourly price before you create anything.
- The plan ID follows the docs' rule `nf-gpu-<gpuType>-<gpuCount>g`: `nf-gpu-h100-80-1g`. `/v1/plans` doesn't list GPU plans. For the 4 × H100 upgrade later, the region also needs `4` in the H100's `countOptions` (`COUNT=4`).

**If you're an agent:** show the user the team, the region (`us-central` below, unless they choose another from the GPU list), the GPU's hourly price and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services, and every service in it runs in the project's region. The region can't be changed later. Skip this step if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "qwen", "region": "us-central", "description": "Qwen on vLLM" }'
```

To use another region from the step 2 list, change `region`. REST: `POST https://api.northflank.com/v1/projects`.

## Step 4: Create the secret group with the API key

```bash
export VLLM_API_KEY=$(openssl rand -hex 32)
TMP=$(umask 077; mktemp)
jq -n --arg k "$VLLM_API_KEY" '{name: "qwen-secrets", type: "secret", secretType: "environment", priority: 10,
  restrictions: {restricted: true, nfObjects: [{id: "qwen", type: "service"}]},
  secrets: {variables: {VLLM_API_KEY: $k}}}' > "$TMP"
northflank put secret --projectId qwen --noDefaults -o json --file "$TMP" >/dev/null && echo SECRET_OK; rm -f "$TMP"
```

vLLM reads `VLLM_API_KEY` from the environment, so the key never appears in the service spec or on a command line. If the restriction to a not-yet-created service is rejected, re-run with `restricted: false`.

## Step 5: Create the model volume

```bash
northflank create volume --projectId qwen --noDefaults -o json --input '{
  "name": "qwen-models",
  "mounts": [ { "containerMountPath": "/root/.cache/huggingface" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 61440 }
}'
```

The FP8 repo is about 30 GB. 60 GB gives room for the BF16 variant later; size is in MB and can only increase.

## Step 6: Deploy Qwen3.8-27B on vLLM

Write `qwen.json`:

```json
{
  "name": "qwen",
  "billing": { "deploymentPlan": "nf-gpu-h100-80-1g" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "vllm/vllm-openai:v0.34.0" },
    "docker": { "configType": "customCommand", "customCommand": "Qwen/Qwen3.8-27B-FP8 --host 0.0.0.0 --port 8000 --served-model-name qwen3.8-27b --max-model-len 65536 --language-model-only --reasoning-parser qwen3 --enable-auto-tool-choice --tool-call-parser qwen3_coder" },
    "gpu": { "enabled": true, "configuration": { "gpuType": "h100-80", "gpuCount": 1, "timesliced": false } }
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
  "createOptions": { "volumesToAttach": ["qwen-models"] }
}
```

```bash
northflank put service deployment --projectId qwen --noDefaults -o json --file qwen.json
```

Notes on the spec:

- The custom command is passed to the image's `vllm serve` entrypoint as arguments, with no shell, so it can't contain quotes. That rules out the recipe's JSON-valued options (speculative decoding, YaRN overrides) until you've confirmed how the string is split.
- `--max-model-len 65536` is a conservative start. The native window is 262,144; raise it once you've seen the KV cache size vLLM reports in the logs.
- Northflank cloud GPU plans default shared memory to the plan's RAM, so `--ipc=host` from the Docker examples needs no equivalent.
- Fallback if the volume isn't attached after the create: remove `createOptions`, re-run the `put`, then `northflank attach volume --projectId qwen --volumeId qwen-models --noDefaults -o json --input '{"nfObject": {"id": "qwen", "type": "service"}}'`.

## Step 7: Wait until the deployment completes

```bash
OK=0
while :; do
  D=$(northflank get service --projectId qwen --serviceId qwen --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "$(date +%T) $D"; case "$D" in COMPLETED) OK=1; break;; FAILED) echo DEPLOY_FAILED; break;; esac; sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId qwen --serviceId qwen --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId qwen --serviceId qwen --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

The first run is the slowest, because the 30 GB download and CUDA graph capture both happen before `/health` turns green. Tail progress with `northflank get service logs --projectId qwen --serviceId qwen --tail`.

## How to know it worked

```bash
AUTH=(-H "Authorization: Bearer $VLLM_API_KEY" -H "Content-Type: application/json")
```

1. **Model listed, key enforced.** `curl -s "https://$HOST/v1/models" "${AUTH[@]}" | jq -r '.data[].id'` prints `qwen3.8-27b`; the same call without the header returns 401.
2. **A direct answer with thinking switched off.** The model card disables thinking per request through `chat_template_kwargs`:
   ```bash
   curl -s "https://$HOST/v1/chat/completions" "${AUTH[@]}" -d '{
     "model": "qwen3.8-27b", "max_tokens": 100, "temperature": 0.7, "top_p": 0.8,
     "chat_template_kwargs": {"enable_thinking": false},
     "messages": [{"role": "user", "content": "Name the capital of Japan in one word."}]}' \
   | jq -r '.choices[0].message.content'
   ```
3. **Tool calling works.** The response should contain a `get_weather` call, not prose:
   ```bash
   curl -s "https://$HOST/v1/chat/completions" "${AUTH[@]}" -d '{
     "model": "qwen3.8-27b", "max_tokens": 1024,
     "tools": [{"type": "function", "function": {"name": "get_weather", "parameters": {"type": "object",
       "properties": {"city": {"type": "string"}}, "required": ["city"]}}}],
     "messages": [{"role": "user", "content": "What is the weather in Paris?"}]}' \
   | jq '.choices[0].message.tool_calls[0].function'
   ```

With thinking on (the default), the reasoning arrives in `message.reasoning`. The card recommends `temperature=1.0, top_p=0.95, top_k=20` for thinking mode and `0.7 / 0.8 / 20` without it.

## Scaling up to Qwen3.8-Flash-Next

The 176B MoE needs about 185 GB of FP8 weights, and the vLLM recipe sets a 250 GB VRAM minimum with H100 and H200 verified. Four H100s (320 GB) meet that. Grow the volume, then patch the service:

```bash
northflank update volume --projectId qwen --volumeId qwen-models --noDefaults -o json --input '{"spec": {"storageSize": 409600}}'
northflank patch service deployment --projectId qwen --serviceId qwen --noDefaults -o json --input '{
  "billing": { "deploymentPlan": "nf-gpu-h100-80-4g" },
  "deployment": { "instances": 1,
    "docker": { "configType": "customCommand", "customCommand": "Qwen/Qwen3.8-Flash-Next-FP8 --host 0.0.0.0 --port 8000 --served-model-name qwen3.8-flash-next --tensor-parallel-size 4 --enable-expert-parallel --max-model-len 131072 --language-model-only --reasoning-parser qwen3 --enable-auto-tool-choice --tool-call-parser qwen3_coder" },
    "gpu": { "enabled": true, "configuration": { "gpuType": "h100-80", "gpuCount": 4, "timesliced": false } } } }'
```

The recipe recommends the vLLM nightly image for Flash-Next but says stable 0.30.0 or later also runs it. Read the Qwen Community License before using Flash-Next commercially; the 27B model is Apache 2.0.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `content` starts with `<think>` | Reasoning parser not set | Add `--reasoning-parser qwen3` |
| Tool calls come back as plain text | Missing `--enable-auto-tool-choice` or wrong parser | Use `--tool-call-parser qwen3_coder` as the recipe does |
| Unknown architecture or processor error at start | vLLM or transformers too old for Qwen3.8 | Keep `v0.34.0` or newer; the recipe asks for transformers 5.8.0 or later |
| CUDA out of memory | Context too long or vision encoder loaded | Lower `--max-model-len`; keep `--language-model-only` |
| Replies loop or repeat | Sampling outside the card's ranges | Use the card's settings; tune `presence_penalty` between 0 and 2 |
| `401` on every call | Key not injected | Check `qwen-secrets` is restricted to `qwen`; restart the service |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're doing this on a user's behalf:

- Confirm the target from step 2 with the user before any write: team, region, GPU price per hour, and whether the project is new or reused.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs with `northflank get service logs --projectId qwen --serviceId qwen --lineLimit 200` before changing anything.
- State the H100 hourly price before step 6 and the 4 × H100 price before scaling.
- Keep `VLLM_API_KEY` out of logs, files and the custom command. Tell the user it lives in `qwen-secrets`.
- `/health` and `/metrics` are open without the key. Add an IP allow-list on the port, or set `public: false` for in-project callers only.
- Stop the meter with `northflank pause service --projectId qwen --serviceId qwen` when testing ends; ask before deleting the volume.

## FAQ

### Where should I deploy Qwen?
Anywhere that gives a long-running container an NVIDIA GPU with enough memory for the checkpoint (38 GB or more for Qwen3.8-27B in FP8), a persistent model cache and an HTTPS port you can protect. This guide uses a container platform with H100 plans so the vLLM image, volume and key are wired together.

### Can I run a smaller Qwen on a cheaper GPU?
Yes. Earlier sizes such as `Qwen/Qwen3-4B`, or `Qwen/Qwen3-8B` with a reduced `--max-model-len`, fit a 24 GB L4; the [vLLM production guide](deploy-llm-in-production-with-vllm.md) serves `Qwen/Qwen3-0.6B` on one L4.

### Does Qwen3.8 accept images through this endpoint?
Not with `--language-model-only`. Remove that flag and send OpenAI-style `image_url` content parts; the vision encoder then uses part of the GPU memory.

### Is Qwen free to use commercially?
`Qwen3.8-27B` and its FP8 variant are Apache 2.0. `Qwen3.8-Flash-Next` and the 2.4T model use other licence terms, so check each model card.

## Conclusion

Qwen3.8-27B now runs on your own H100 behind an OpenAI-compatible API, with thinking and tool calls parsed into standard fields, weights cached on a volume and access gated by a key.

**Next:** issue per-team keys with a [LiteLLM proxy](deploy-litellm-proxy.md), or compare against [Llama](deploy-llama.md) and [DeepSeek](host-deepseek.md) on the same pattern. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)
- [How to host DeepSeek](host-deepseek.md)
- [How to deploy Llama](deploy-llama.md)
- [How to deploy LiteLLM proxy](deploy-litellm-proxy.md)
- [How to deploy Open WebUI with Ollama](deploy-open-webui-with-ollama.md)
