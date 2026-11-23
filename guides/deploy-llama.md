<!--
agent-guide-meta:
  id: NF-STK-LLAMA
  description: "How to deploy Llama on a cloud GPU with vLLM: Llama 3.1 8B Instruct on an L4 with a gated-model HF token, tool calling, a weights volume and an API key."
  sources:
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://huggingface.co/meta-llama/Llama-3.1-8B-Instruct
    - https://huggingface.co/meta-llama/Llama-4-Scout-17B-16E-Instruct
    - https://huggingface.co/api/models/meta-llama/Llama-3.1-8B-Instruct
    - https://github.com/vllm-project/recipes/blob/main/models/meta-llama/Llama-3.1-8B-Instruct.yaml
    - https://github.com/vllm-project/recipes/blob/main/models/meta-llama/Llama-4-Scout-17B-16E-Instruct.yaml
    - https://docs.vllm.ai/en/latest/features/tool_calling/
    - https://github.com/vllm-project/vllm/blob/v0.34.0/docker/Dockerfile
    - https://github.com/vllm-project/vllm/blob/v0.34.0/vllm/tool_parsers/__init__.py
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
    - https://northflank.com/docs/v1/application/gpu-workloads/deploy-gpus-on-northflank-cloud
    - https://northflank.com/docs/v1/application/use-cases/host-a-model-on-gpus
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/volumes/attach-volume
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/miscellaneous/miscellaneous/list-regions
  related: [NF-STK-VLLM, NF-STK-MISTRAL, NF-STK-QWEN, NF-STK-OLLAMA, NF-STK-LITELLM]
-->

# How to deploy Llama

To deploy Llama, accept Meta's licence on the model's Hugging Face page, then run vLLM's OpenAI-compatible server on an NVIDIA GPU with your Hugging Face token as a secret, a persistent volume for the weights and an API key on `/v1`. Llama repos are gated, so a missing or unapproved token is the most common reason the server fails at download.

This guide does it on Northflank with the CLI, serving `meta-llama/Llama-3.1-8B-Instruct` on one L4 GPU with tool calling enabled, then shows the switch to Llama 4 Scout on four H100s. You end with an HTTPS endpoint answering `/v1/models`, a chat completion and a tool call. Each step is a command a developer or coding agent (Claude Code, Codex, Cursor) can run, except one: a human must request model access on Hugging Face.

## TL;DR: how to deploy Llama

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Have a human accept the Llama 3.1 licence on Hugging Face and create a read token, then check the token returns 200 on the repo's `config.json`, find an L4 region with the public `curl -s https://api.northflank.com/v1/regions`, and confirm the team, region, hourly price and project.
3. Create the project with `northflank create project`, store `HF_TOKEN` and a generated `VLLM_API_KEY` in a secret group with `northflank put secret`, and create a 40 GB volume for `/root/.cache/huggingface` with `northflank create volume`.
4. Deploy `vllm/vllm-openai:v0.34.0` on `nf-gpu-l4-24-1g` with `northflank put service deployment`, using `--max-model-len 16384 --tool-call-parser llama3_json` and the Llama 3.1 JSON chat template.
5. Verify `GET /v1/models`, a chat completion and a tool call with `Authorization: Bearer $VLLM_API_KEY`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Llama is Meta's open-weights family. The current instruct models on Hugging Face are Llama 3.1 (8B, 70B, 405B), Llama 3.3 70B and the Llama 4 mixture-of-experts models, Scout (109B total, 17B active) and Maverick (400B total). Picking one is a GPU-sizing decision:

- **Llama 3.1 8B Instruct**: 8B parameters, 16 GB of BF16 weights, 131,072-token context. The vLLM recipe sets a 20 GB VRAM minimum. On a 24 GB L4 that leaves a few gigabytes for the KV cache (about 128 KB per token), so the context is capped at 16,384 here. An A100 40 GB serves the full context.
- **Llama 3.3 70B Instruct**: about 141 GB in BF16, so multiple 80 GB GPUs.
- **Llama 4 Scout 17B-16E Instruct**: about 217 GB in BF16; the recipe's minimum is 262 GB, which four H100s (320 GB) meet.

**Licence and access.** Every `meta-llama` repo is gated with manual review. You agree to the Llama Community License (3.1, 3.3 or 4), which requires a "Built with Llama" notice if you distribute the model or a product built on it, compliance with Meta's Acceptable Use Policy, and a separate licence from Meta if your products had more than 700 million monthly active users on the release date. The deployment therefore needs an `HF_TOKEN` from an account Meta has approved.

**Tool calling.** vLLM's tool-calling docs pair Llama 3.1 with `--tool-call-parser llama3_json` and the `examples/tool_chat_template_llama3.1_json.jinja` template. The `vllm/vllm-openai` image copies `examples/` into its `/vllm-workspace` working directory, so the template is referenced by absolute path. Llama 3 doesn't support parallel tool calls; Llama 4 does.

Where to deploy Llama follows from that: a container host with an NVIDIA GPU, a secret store for the gated-model token, persistent disk for the weights and HTTPS in front. A GPU VM or a Kubernetes cluster you run yourself works if you maintain drivers and restarts; a container platform with GPU plans runs vLLM with those attached. For CPU-friendly quantised Llama builds, [Ollama](host-ollama-in-the-cloud.md) is the simpler path.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **`openssl`** (to generate the API key).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **GPU access.** Free projects can't run GPUs, and GPU deployments need account credit.
- **Hugging Face access to Llama.** **This needs a human:** sign in to Hugging Face, open `meta-llama/Llama-3.1-8B-Instruct`, accept the licence form and wait for approval, then create a read token and export it as `HF_TOKEN` in the agent's shell (never in a file). Step 2 checks it.

Names used below: project `llama`, secret group `llama-secrets`, volume `llama-weights`, service `llama`.

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
if P=$(northflank get project --projectId llama -o json 2>/dev/null); then
  echo "project: llama exists in $(echo "$P" | jq -r '(.data // .).deployment.region') and will be reused"
else
  echo "project: llama will be created"
fi

GPU=l4-24; COUNT=1
curl -s https://api.northflank.com/v1/regions | jq -r --arg g "$GPU" --argjson n "$COUNT" '
  .data.regions[] | select(any(.gpuDevices[]?; .id == $g and (.countOptions | index($n))))
  | "\(.id)\t\(.gpuDevices[] | select(.id == $g) | .pricing.onDemand) cents/h"'

curl -s -o /dev/null -w 'hf token: %{http_code}\n' -H "Authorization: Bearer $HF_TOKEN" \
  https://huggingface.co/meta-llama/Llama-3.1-8B-Instruct/resolve/main/config.json
```

- If the GPU list is empty, no region offers 1 × `l4-24` right now. Pick another GPU (the `.data.regions[].gpuDevices[].id` values in the same response) and change `GPU`.
- If the project already exists, its region must be in the GPU list, because a project's region can't be changed. If it isn't, use a new project name throughout.
- **GPUs need a human.** Free projects can't run GPU workloads, and GPU deployments need account credit. Ask the account owner to confirm billing and the hourly price before you create anything.
- The plan ID is `nf-gpu-l4-24-1g`, as listed on the resource plans docs page; GPU plans don't appear in `/v1/plans`.
- `hf token: 200` means the token is approved for the repo. `401` means it's missing or wrong, and `403` means access hasn't been granted yet: stop and ask the user.

**If you're an agent:** show the user the team, the region (`europe-west` below, unless they choose another from the GPU list), the GPU's hourly price and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services, and every service in it runs in the project's region. The region can't be changed later. Skip this step if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "llama", "region": "europe-west", "description": "Llama on vLLM" }'
```

To use another region from the step 2 list, change `region`. REST: `POST https://api.northflank.com/v1/projects`. If you might move to Llama 4 later, choose a region that also lists `h100-80` with a count of 4.

## Step 4: Store the Hugging Face token and API key

Both values go into one secret group restricted to the service, and neither is printed:

```bash
export VLLM_API_KEY=$(openssl rand -hex 32)
jq -n --arg hf "$HF_TOKEN" --arg k "$VLLM_API_KEY" '{
  name: "llama-secrets", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "llama", type: "service" } ] },
  secrets: { variables: { HF_TOKEN: $hf, VLLM_API_KEY: $k } }
}' > llama-secrets.json
northflank put secret --projectId llama --noDefaults -o json --file llama-secrets.json >/dev/null; rm -f llama-secrets.json
```

vLLM reads `HF_TOKEN` for the download and `VLLM_API_KEY` for request auth. If the API rejects a restriction to a service that isn't created yet, set `restricted: false`.

## Step 5: Create the weights volume

```bash
northflank create volume --projectId llama --noDefaults -o json --input '{
  "name": "llama-weights",
  "mounts": [ { "containerMountPath": "/root/.cache/huggingface" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 40960 }
}'
```

vLLM pulls the 16 GB of root-level safetensors. 40 GB (`storageSize` is in MB) covers that with headroom; volumes can only grow.

## Step 6: Deploy Llama 3.1 8B with tool calling

Create `llama.json`:

```json
{
  "name": "llama",
  "billing": { "deploymentPlan": "nf-gpu-l4-24-1g" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "vllm/vllm-openai:v0.34.0" },
    "docker": { "configType": "customCommand", "customCommand": "meta-llama/Llama-3.1-8B-Instruct --host 0.0.0.0 --port 8000 --served-model-name llama-3.1-8b --max-model-len 16384 --enable-auto-tool-choice --tool-call-parser llama3_json --chat-template /vllm-workspace/examples/tool_chat_template_llama3.1_json.jinja" },
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
  "createOptions": { "volumesToAttach": ["llama-weights"] }
}
```

```bash
northflank put service deployment --projectId llama --noDefaults -o json --file llama.json
```

The command string becomes the arguments to the image's `vllm serve` entrypoint (no shell, so no quotes). If the server exits with a missing-template error, remove `--chat-template …` to fall back to the model's own chat template; plain chat still works and tool calls may be less reliable. If the volume isn't attached after creation, attach it, which triggers a redeploy:

```bash
northflank attach volume --projectId llama --volumeId llama-weights --noDefaults -o json \
  --input '{"nfObject": {"id": "llama", "type": "service"}}'
```

## Step 7: Wait for the rollout and get the URL

```bash
OK=0
for i in $(seq 1 200); do
  D=$(northflank get service --projectId llama --serviceId llama --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"; [ "$D" = "COMPLETED" ] && { OK=1; break; }; [ "$D" = FAILED ] && { echo FAILED; break; }; sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId llama --serviceId llama --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId llama --serviceId llama --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

A `403` or `GatedRepoError` in `northflank get service logs --projectId llama --serviceId llama --lineLimit 100` means the token isn't approved for this repo.

## How to know it worked

1. `curl -s "https://$HOST/v1/models" -H "Authorization: Bearer $VLLM_API_KEY" | jq -r '.data[].id'` prints `llama-3.1-8b`, and the same request without the header gets `401`.
2. A plain completion answers:
   ```bash
   curl -s "https://$HOST/v1/chat/completions" -H "Authorization: Bearer $VLLM_API_KEY" -H "Content-Type: application/json" \
     -d '{"model": "llama-3.1-8b", "max_tokens": 60, "messages": [{"role": "user", "content": "Say hello in French."}]}' \
     | jq -r '.choices[0].message.content'
   ```
3. A tool call comes back structured:
   ```bash
   curl -s "https://$HOST/v1/chat/completions" -H "Authorization: Bearer $VLLM_API_KEY" -H "Content-Type: application/json" -d '{
     "model": "llama-3.1-8b", "max_tokens": 200, "tool_choice": "auto",
     "tools": [{"type": "function", "function": {"name": "lookup_order", "description": "Look up an order by ID",
       "parameters": {"type": "object", "properties": {"order_id": {"type": "string"}}, "required": ["order_id"]}}}],
     "messages": [{"role": "user", "content": "Where is order A-1042?"}]}' | jq '.choices[0].message.tool_calls'
   ```
   Expect one call to `lookup_order` with `{"order_id": "A-1042"}` as arguments.

## Move up to Llama 4 Scout

Scout needs separate Hugging Face approval for `meta-llama/Llama-4-Scout-17B-16E-Instruct` under the Llama 4 Community License (same token, new request). Grow the volume to fit 217 GB of weights, then patch the service to four H100s with the `llama4_pythonic` parser:

```bash
northflank update volume --projectId llama --volumeId llama-weights --noDefaults -o json --input '{"spec": {"storageSize": 409600}}'
northflank patch service deployment --projectId llama --serviceId llama --noDefaults -o json --input '{
  "billing": { "deploymentPlan": "nf-gpu-h100-80-4g" },
  "deployment": { "instances": 1,
    "docker": { "configType": "customCommand", "customCommand": "meta-llama/Llama-4-Scout-17B-16E-Instruct --host 0.0.0.0 --port 8000 --served-model-name llama-4-scout --tensor-parallel-size 4 --max-model-len 65536 --enable-auto-tool-choice --tool-call-parser llama4_pythonic --chat-template /vllm-workspace/examples/tool_chat_template_llama4_pythonic.jinja" },
    "gpu": { "enabled": true, "configuration": { "gpuType": "h100-80", "gpuCount": 4, "timesliced": false } } } }'
```

The recipe also lists an NVIDIA FP8 build, `nvidia/Llama-4-Scout-17B-16E-Instruct-FP8` (131 GB minimum), which fits two H100s with `--kv-cache-dtype fp8`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `401`/`403`/`GatedRepoError` in logs | Token missing, wrong, or access not yet approved | Re-run the step 2 `curl`; update `HF_TOKEN` in `llama-secrets`; restart |
| CUDA out of memory on the L4 | 16,384 tokens still too long for remaining memory | Lower `--max-model-len` to 8192, or move to `a100-40` |
| Tool calls returned as text | Parser or template missing | Keep `--enable-auto-tool-choice --tool-call-parser llama3_json` and the template |
| Only one of several requested tools is called | Llama 3 has no parallel tool calls | Expected; use Llama 4 for parallel calls |
| `401` from vLLM with the right key | Secret group not applied | Check the restriction lists `llama`; restart the service |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're executing this for a user:

- Confirm the target from step 2 with the user before any write: team, region, GPU price per hour, and whether the project is new or reused.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs with `northflank get service logs --projectId llama --serviceId llama --lineLimit 200` before changing anything.
- Stop and ask for the human Hugging Face step; don't try to accept licences on the user's behalf.
- Never echo `HF_TOKEN` or `VLLM_API_KEY`, and keep both out of the custom command.
- Quote the L4 hourly price from `/v1/regions` before deploying, and the 4 × H100 price before the Scout patch.
- Remind the user about the "Built with Llama" notice if they ship a product on it.
- Pause idle services with `northflank pause service`; confirm before deleting volumes.

## FAQ

### Where should I deploy Llama?
On a host that gives a long-running container an NVIDIA GPU sized for the model (an L4 for 8B with a modest context, multiple H100s for Llama 4), a secret for the gated-model token and persistent storage for weights. A container platform with GPU plans, as used here, wires those to the vLLM image.

### Can I deploy Llama without a Hugging Face token?
Not from the `meta-llama` repos: they're gated, and vLLM downloads with `HF_TOKEN`. Mirrors and quantised community builds exist, but they're still covered by Meta's licence.

### Llama 3.1 or Llama 4?
Llama 3.1 8B is the cheapest to run and handles chat and simple tools. Llama 4 Scout adds parallel tool calls, image input and a much longer context, at the cost of a multi-GPU plan.

## Conclusion

You now have Llama running on your own GPU behind an OpenAI-compatible API, with the gated-model token and API key in a secret group, weights on a volume and tool calling switched on.

**Next:** add per-user keys with a [LiteLLM proxy](deploy-litellm-proxy.md), or try [Mistral](self-host-mistral.md) with the same layout. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)
- [How to self-host Mistral](self-host-mistral.md)
- [How to deploy Qwen](deploy-qwen.md)
- [How to host Ollama in the cloud](host-ollama-in-the-cloud.md)
- [How to deploy LiteLLM proxy](deploy-litellm-proxy.md)
