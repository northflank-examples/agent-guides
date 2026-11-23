<!--
agent-guide-meta:
  id: NF-STK-GPTOSS
  description: "How to host gpt-oss with vLLM: gpt-oss-120b in MXFP4 on one 80 GB A100, a weights volume, an API key, reasoning effort control and tool calls."
  sources:
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://huggingface.co/openai/gpt-oss-120b
    - https://huggingface.co/openai/gpt-oss-20b
    - https://github.com/vllm-project/recipes/blob/main/models/openai/gpt-oss-120b.yaml
    - https://github.com/vllm-project/recipes/blob/main/models/openai/gpt-oss-20b.yaml
    - https://docs.vllm.ai/en/latest/features/tool_calling/
    - https://github.com/vllm-project/vllm/blob/v0.34.0/vllm/entrypoints/openai/chat_completion/protocol.py
    - https://github.com/vllm-project/vllm/blob/v0.34.0/vllm/model_executor/model_loader/default_loader.py
    - https://northflank.com/stacks/gpt-oss-120b
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
    - https://northflank.com/docs/v1/application/gpu-workloads/deploy-gpus-on-northflank-cloud
    - https://northflank.com/docs/v1/application/use-cases/host-a-model-on-gpus
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/volumes/attach-volume
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/miscellaneous/miscellaneous/list-regions
  related: [NF-STK-VLLM, NF-STK-DEEPSEEK, NF-STK-MISTRAL, NF-STK-OPENWEBUI, NF-STK-LITELLM]
-->

# How to host gpt-oss

To host gpt-oss, serve `openai/gpt-oss-120b` or `openai/gpt-oss-20b` with vLLM's OpenAI-compatible server on NVIDIA GPU memory of at least 80 GB for 120b (the model card's figure; the vLLM recipe sets a 96 GB minimum) or 16 GB for 20b, keep the downloaded weights on a persistent volume, enable the `openai` tool-call parser and require an API key. The models only work with OpenAI's harmony format, which vLLM detects and applies for you.

This guide does it on Northflank with the CLI: gpt-oss-120b on a single A100 80 GB (below that 96 GB minimum, so treat it as the budget option), the key in a secret group, weights on a volume and a long startup probe. You end with an HTTPS endpoint verified by `/v1/models`, a chat completion at low reasoning effort and a tool call, plus how to move to two H100s or down to gpt-oss-20b. Every step is a command a developer or coding agent (Claude Code, Codex, Cursor) can paste and run.

## TL;DR: how to host gpt-oss

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check which regions offer `a100-80` with the public `curl -s https://api.northflank.com/v1/regions`, and confirm the team, region, hourly price and project before creating anything.
3. Create the project with `northflank create project`, store a generated `VLLM_API_KEY` in a secret group with `northflank put secret`, and create a 100 GB volume at `/root/.cache/huggingface` with `northflank create volume`.
4. Deploy `vllm/vllm-openai:v0.34.0` on `nf-gpu-a100-80-1g` with `northflank put service deployment`, serving `openai/gpt-oss-120b` with `--tool-call-parser openai --enable-auto-tool-choice` (one 80 GB card is below the vLLM recipe's 96 GB minimum; 2 × H100 is the safer layout).
5. Verify `GET /v1/models`, a chat completion with `"reasoning_effort": "low"` and a tool call, using `Authorization: Bearer $VLLM_API_KEY`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

gpt-oss is OpenAI's open-weights reasoning family, released under Apache 2.0 with a short usage policy that asks you to comply with applicable law. Two sizes exist:

| | gpt-oss-120b | gpt-oss-20b |
|---|---|---|
| Parameters | 117B total, 5.1B active (MoE) | 21B total, 3.6B active (MoE) |
| Weights | MXFP4 experts, about 61 GB | MXFP4 experts, about 14 GB |
| GPU memory | "a single 80GB GPU (like NVIDIA H100 or AMD MI300X)" | "within 16GB of memory" |
| Context | 131,072 tokens | 131,072 tokens |

The vLLM GPT-OSS recipe gives a single-card A100 launch (`vllm serve openai/gpt-oss-120b`, using Triton attention and the Marlin MXFP4 MoE kernel) and lists "A100 80GB for single-GPU" in its prerequisites, so this guide uses Northflank's `a100-80`, which costs less per hour than an H100. Switching to `h100-80` changes only the plan and GPU type.

> **Warning: one 80 GB GPU is below the recipe's stated minimum.** The same recipe's metadata sets a **96 GB VRAM minimum** for gpt-oss-120b, lists only H100, H200, B200 and AMD MI300-class GPUs as verified, and calls Ampere (A100) support ongoing work. One A100 80 GB is therefore an undersized, unverified choice that can run out of memory or leave little KV cache; a single H100 80 GB is also under the minimum, and the recipe's troubleshooting covers H100 single-GPU out-of-memory errors. For a configuration that clears the minimum, use the 2 × H100 layout in [Change size or GPU](#change-size-or-gpu) from the start.

Two vLLM behaviours shape the command:

- **Harmony is handled by vLLM.** The model card warns the models "should only be used with the harmony format". vLLM detects the gpt-oss model type and renders and parses harmony itself, so clients send normal OpenAI messages. `reasoning_effort` (`low`, `medium`, `high`) is a request field.
- **Tool calls need a parser.** The recipe and vLLM's tool-calling docs use `--tool-call-parser openai --enable-auto-tool-choice`. The recipe notes function calling supports `tool_choice: "auto"` only.

Where to host gpt-oss follows from the table: a container with one 80 GB NVIDIA GPU for 120b (or a 16 GB+ card for 20b), roughly 65 GB of persistent disk and an HTTPS port you can lock down. Raw GPU VMs work if you run and upgrade vLLM yourself; a container platform with GPU plans runs the image with the disk and secret attached. Northflank's [gpt-oss-120b stack](https://northflank.com/stacks/gpt-oss-120b) deploys the same model with vLLM on 2 × H100 plus Open WebUI in one click.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **`openssl`** (to generate the API key).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **GPU access.** Free projects can't run GPUs, and GPU deployments need account credit.
- **No Hugging Face token is needed.** Both repos are public and ungated.

Names used below: project `gpt-oss`, secret group `gpt-oss-secrets`, volume `gpt-oss-weights`, service `gpt-oss`.

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

Before creating anything, print the team, whether the project exists, and the regions that offer one NVIDIA A100 80 GB with its on-demand price in US cents per GPU-hour. The regions endpoint is public, so it needs no token:

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
if P=$(northflank get project --projectId gpt-oss -o json 2>/dev/null); then
  echo "project: gpt-oss exists in $(echo "$P" | jq -r '(.data // .).deployment.region') and will be reused"
else
  echo "project: gpt-oss will be created"
fi

GPU=a100-80; COUNT=1
curl -s https://api.northflank.com/v1/regions | jq -r --arg g "$GPU" --argjson n "$COUNT" '
  .data.regions[] | select(any(.gpuDevices[]?; .id == $g and (.countOptions | index($n))))
  | "\(.id)\t\(.gpuDevices[] | select(.id == $g) | .pricing.onDemand) cents/h"'
```

- If the GPU list is empty, no region offers 1 × `a100-80` right now. Pick another GPU (the `.data.regions[].gpuDevices[].id` values in the same response) and change `GPU`.
- If the project already exists, its region must be in the GPU list, because a project's region can't be changed. If it isn't, use a new project name throughout.
- **GPUs need a human.** Free projects can't run GPU workloads, and GPU deployments need account credit. Ask the account owner to confirm billing and the hourly price before you create anything.
- Set `GPU=h100-80; COUNT=2` to compare with the two-card layout. One A100 80 GB is plan `nf-gpu-a100-80-1g`, as listed on the resource plans docs page; GPU plans aren't returned by `/v1/plans`.

**If you're an agent:** show the user the team, the region (`us-central` below, unless they choose another from the GPU list), the GPU's hourly price and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services, and every service in it runs in the project's region. The region can't be changed later. Skip this step if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "gpt-oss", "region": "us-central", "description": "OpenAI gpt-oss on vLLM" }'
```

To use another region from the step 2 list, change `region`. REST: `POST https://api.northflank.com/v1/projects`. `us-central` lists both `a100-80` and `h100-80`, so you can move between them without a new project.

## Step 4: Create the secret group

```bash
export VLLM_API_KEY=$(openssl rand -hex 32)
cat > gpt-oss-secrets.json <<EOF
{ "name": "gpt-oss-secrets", "type": "secret", "secretType": "environment", "priority": 10,
  "restrictions": { "restricted": true, "nfObjects": [ { "id": "gpt-oss", "type": "service" } ] },
  "secrets": { "variables": { "VLLM_API_KEY": "$VLLM_API_KEY" } } }
EOF
northflank put secret --projectId gpt-oss --noDefaults -o json --file gpt-oss-secrets.json >/dev/null; shred -u gpt-oss-secrets.json 2>/dev/null || rm -f gpt-oss-secrets.json
```

The hex key needs no JSON escaping, so a heredoc is enough. vLLM picks `VLLM_API_KEY` up from the environment. If restricting the group to a service that isn't created yet fails, use `"restricted": false`.

## Step 5: Create the weights volume

```bash
northflank create volume --projectId gpt-oss --noDefaults -o json --input '{
  "name": "gpt-oss-weights",
  "mounts": [ { "containerMountPath": "/root/.cache/huggingface" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 102400 }
}'
```

The repo is about 190 GB in total, but vLLM only downloads the root `*.safetensors` files (about 61 GB): it skips `original/**` by default, and `metal/` holds a `model.bin` that doesn't match its download pattern. 100 GB (`storageSize` is in MB) leaves headroom. Volumes can be enlarged, never shrunk.

## Step 6: Deploy gpt-oss-120b

Write `gpt-oss.json`:

```json
{
  "name": "gpt-oss",
  "billing": { "deploymentPlan": "nf-gpu-a100-80-1g" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "vllm/vllm-openai:v0.34.0" },
    "docker": { "configType": "customCommand", "customCommand": "openai/gpt-oss-120b --host 0.0.0.0 --port 8000 --served-model-name gpt-oss-120b --tool-call-parser openai --enable-auto-tool-choice" },
    "gpu": { "enabled": true, "configuration": { "gpuType": "a100-80", "gpuCount": 1, "timesliced": false } }
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
  "createOptions": { "volumesToAttach": ["gpt-oss-weights"] }
}
```

```bash
northflank put service deployment --projectId gpt-oss --noDefaults -o json --file gpt-oss.json
```

About the command: it's the argument list for the image's `vllm serve` entrypoint, passed without a shell, so it contains no quotes. The recipe needs no context flag (the full 131,072 tokens are the default), and the 30-minute startup window matches what the Northflank stack page says a first load can take. If the volume isn't attached afterwards, attach it explicitly; this redeploys:

```bash
northflank attach volume --projectId gpt-oss --volumeId gpt-oss-weights --noDefaults -o json \
  --input '{"nfObject": {"id": "gpt-oss", "type": "service"}}'
```

## Step 7: Wait for the model

```bash
OK=0
for n in $(seq 1 200); do
  STATUS=$(northflank get service --projectId gpt-oss --serviceId gpt-oss --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  printf '%s %s\n' "$n" "$STATUS"
  case "$STATUS" in COMPLETED) OK=1; break ;; FAILED) echo "check logs"; break ;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId gpt-oss --serviceId gpt-oss --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId gpt-oss --serviceId gpt-oss --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

For progress or errors: `northflank get service logs --projectId gpt-oss --serviceId gpt-oss --lineLimit 150`.

## How to know it worked

```bash
H=(-H "Authorization: Bearer $VLLM_API_KEY" -H "Content-Type: application/json")
```

1. **Listed and protected.** `curl -s "https://$HOST/v1/models" "${H[@]}" | jq -r '.data[].id'` returns `gpt-oss-120b`; dropping the header returns `401`.
2. **Reasoning effort is honoured.** With `low`, the answer comes back quickly and the chain of thought is kept out of `content`:
   ```bash
   curl -s "https://$HOST/v1/chat/completions" "${H[@]}" -d '{
     "model": "gpt-oss-120b", "reasoning_effort": "low", "max_tokens": 512,
     "messages": [{"role": "user", "content": "Is 221 prime? Answer yes or no, then give its factors."}]}' \
   | jq '{answer: .choices[0].message.content, has_reasoning: (.choices[0].message.reasoning != null)}'
   ```
   Expect `no` with 13 × 17. The model card notes the chain of thought is "not intended to be shown to end users", so don't surface `reasoning` in a product UI.
3. **Tool calls are parsed.**
   ```bash
   curl -s "https://$HOST/v1/chat/completions" "${H[@]}" -d '{
     "model": "gpt-oss-120b", "tool_choice": "auto", "max_tokens": 512,
     "tools": [{"type": "function", "function": {"name": "create_ticket", "parameters": {"type": "object",
       "properties": {"title": {"type": "string"}, "priority": {"type": "string", "enum": ["low", "high"]}}, "required": ["title"]}}}],
     "messages": [{"role": "user", "content": "Open a high-priority ticket titled Checkout is down."}]}' \
   | jq '.choices[0].message.tool_calls[0].function'
   ```

The OpenAI SDKs work unchanged with `base_url="https://$HOST/v1"`. vLLM also serves `/v1/responses`, but the recipe lists limits there (basic streaming, no annotations, zero usage counts).

## Change size or GPU

- **More throughput for 120b:** two H100s with tensor parallelism, the layout Northflank's stack uses:
  ```bash
  northflank patch service deployment --projectId gpt-oss --serviceId gpt-oss --noDefaults -o json --input '{
    "billing": { "deploymentPlan": "nf-gpu-h100-80-2g" },
    "deployment": { "instances": 1,
      "docker": { "configType": "customCommand", "customCommand": "openai/gpt-oss-120b --host 0.0.0.0 --port 8000 --served-model-name gpt-oss-120b --tensor-parallel-size 2 --tool-call-parser openai --enable-auto-tool-choice" },
      "gpu": { "enabled": true, "configuration": { "gpuType": "h100-80", "gpuCount": 2, "timesliced": false } } } }'
  ```
- **Cheaper with gpt-oss-20b:** the same patch shape with plan `nf-gpu-a100-40-1g`, `gpuType` `a100-40`, `gpuCount` 1 and a command starting `openai/gpt-oss-20b`. Keep `--served-model-name` distinct so clients notice the switch. The recipe's verified GPUs for 20b are Hopper and newer; vLLM lists Ampere and Ada support as ongoing work.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| CUDA out of memory on one 80 GB GPU | Batch or context budget too high | The recipe's fix: add `--gpu-memory-utilization 0.95 --max-num-batched-tokens 1024`, or move to 2 × H100 |
| Error fetching the harmony vocabulary at start | tiktoken files couldn't be downloaded | Pre-download them and set `TIKTOKEN_ENCODINGS_BASE` (recipe advice) |
| Tool call returned as text | Parser flags missing, or `tool_choice` not `auto` | Keep `--tool-call-parser openai --enable-auto-tool-choice`; send `"tool_choice": "auto"` |
| Disk full during download | Volume shared with other checkpoints | Grow the volume with `northflank update volume` |
| Plan rejected | Region lacks `a100-80`, or a free project | Re-run the regions query; ask a human about billing |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

When running this on a user's behalf:

- Confirm the target from step 2 with the user before any write: team, region, GPU price per hour, and whether the project is new or reused.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs with `northflank get service logs --projectId gpt-oss --serviceId gpt-oss --lineLimit 200` before changing anything.
- Quote the A100 80 GB price from `/v1/regions` before step 6, and the 2 × H100 price before patching.
- Never print `VLLM_API_KEY`, and don't put it in the command string.
- `/health` and `/metrics` aren't behind the key. If only internal callers need the model, set the port to `public: false` and use `http://gpt-oss:8000/v1`.
- Pause the service when the user is done (`northflank pause service --projectId gpt-oss --serviceId gpt-oss`) and ask before deleting the volume.

## FAQ

### Where should I host gpt-oss?
Anywhere that gives a long-running container enough NVIDIA GPU memory for gpt-oss-120b (the model card says one 80 GB GPU; the vLLM recipe's minimum is 96 GB, so 2 × H100 is the safe choice) or a 16 GB+ GPU for gpt-oss-20b, persistent disk for the weights and a protected HTTPS endpoint. A container platform with GPU plans, as in this guide, runs the vLLM image with those attached.

### Can I use gpt-oss commercially?
Yes. Both models are Apache 2.0; the bundled usage policy asks you to comply with applicable law.

### Should I use Ollama instead?
For a laptop or a quick test, the model card also lists Ollama. For a multi-user API with batching and tool parsing, vLLM is the path the card and recipe document; Ollama in the cloud is covered in [How to host Ollama in the cloud](host-ollama-in-the-cloud.md).

## Conclusion

gpt-oss-120b now runs on your own 80 GB GPU behind an OpenAI-compatible API, with reasoning effort per request, parsed tool calls, cached weights and a key in a secret group.

**Next:** add a chat UI with [Open WebUI](deploy-open-webui-with-ollama.md), or route several models through a [LiteLLM proxy](deploy-litellm-proxy.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)
- [How to host DeepSeek](host-deepseek.md)
- [How to self-host Mistral](self-host-mistral.md)
- [How to host Ollama in the cloud](host-ollama-in-the-cloud.md)
- [How to deploy LiteLLM proxy](deploy-litellm-proxy.md)
