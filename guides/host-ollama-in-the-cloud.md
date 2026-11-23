<!--
agent-guide-meta:
  id: NF-STK-OLLAMA
  description: "How to host Ollama in the cloud on an NVIDIA GPU with the official Docker image, a persistent model volume and a private API port, with every step scripted."
  sources:
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/stacks/deploy-ollama
    - https://docs.ollama.com/docker
    - https://docs.ollama.com/faq
    - https://docs.ollama.com/api/pull
    - https://docs.ollama.com/api/tags
    - https://docs.ollama.com/api/authentication
    - https://docs.ollama.com/api/openai-compatibility
    - https://github.com/ollama/ollama/blob/main/Dockerfile
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
    - https://northflank.com/docs/v1/application/gpu-workloads/deploy-gpus-on-northflank-cloud
    - https://northflank.com/docs/v1/application/billing/credits
    - https://northflank.com/docs/v1/application/gpu-workloads/configure-and-optimise-workloads-for-gpus
    - https://northflank.com/docs/v1/application/sandboxes/sandboxes-on-northflank-cloud
    - https://northflank.com/docs/v1/api/miscellaneous/miscellaneous/list-regions
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/network/networking-on-northflank
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-STK-OPENWEBUI, NF-STK-VLLM, NF-STK-LITELLM, NF-DEP-002]
-->

# How to host Ollama in the cloud

To host Ollama in the cloud, run the official `ollama/ollama` Docker image on a server with an NVIDIA GPU, listen on `OLLAMA_HOST=0.0.0.0:11434`, mount a persistent volume at `/root/.ollama` so models survive restarts, pull each model once, and keep port 11434 on a private network. Ollama's API has no authentication, so anyone who can reach that port can run, pull and delete models.

This guide does it on Northflank with the CLI: an NVIDIA L4 GPU service (with a CPU fallback) running the official image, a volume for the model cache, and a private port that other services call at `http://ollama:11434`. You end with a pulled model answering on the GPU, verified through a private tunnel. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; GPUs need account credit, which a human confirms first.

## TL;DR: how to host Ollama in the cloud

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check which regions offer an NVIDIA L4 with the public `curl -s https://api.northflank.com/v1/regions`, and confirm the team, region and project before creating anything.
3. Create the project with `northflank create project`, then run `ollama/ollama:<version>` with `northflank put service deployment`: plan `nf-gpu-l4-24-1g`, a `deployment.gpu` block and a **private** port 11434.
4. Attach a volume at `/root/.ollama` with `northflank create volume`, wait for the rollout, and pull a model with `script -q /dev/null northflank exec service … --cmd '"ollama pull llama3.2"'`.
5. Verify with `ollama ps` (expect `100% GPU`) and `curl …/api/tags` through `northflank forward service`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Ollama's defaults suit a laptop. Moving it to a cloud server means picking a host that can give it what it needs: an NVIDIA GPU with enough memory for your models, a persistent volume for the model cache, and a long-running container with a private network. Serverless and static hosts don't offer GPUs or keep a multi-gigabyte cache around. A raw GPU VM or your own Kubernetes cluster works, but you install drivers and the container runtime yourself. A container platform with GPU plans runs the official Docker image with the GPU attached for you.

Three of Ollama's defaults need changing on a server:

- **Bind address.** Ollama "binds 127.0.0.1 port 11434 by default" according to its FAQ. The official image already sets `OLLAMA_HOST=0.0.0.0:11434` in its Dockerfile. This guide sets it again explicitly so the port is reachable from other containers.
- **Model storage.** The Docker docs mount a volume at `/root/.ollama`, and the Northflank GPU docs list `/root/.ollama` as Ollama's model path. Containers are stateless, so without a volume every restart downloads the models again.
- **Authentication.** Ollama's API docs say the local API "does not require authentication". Anyone who can reach port 11434 can run, pull and delete models. So this guide keeps the port private: other services in the project call `http://ollama:11434`, and you reach it from your laptop with a port-forward.

If you'd rather click through it, Northflank has an [Ollama stack template](https://northflank.com/stacks/deploy-ollama) that deploys the same image with a GPU and a volume at `/root/.ollama` from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **GPU access.** Free projects can't run GPUs, and the GPU docs ask for at least $50 of account credit. Without it, use the CPU fallback in step 5.
- **A pinned Ollama version.** List recent tags with `curl -s "https://hub.docker.com/v2/repositories/ollama/ollama/tags?page_size=20&ordering=last_updated" | jq -r '.results[].name'`, and use a plain version such as `0.40.16`, not an `-rc` or `rocm` tag.

Names used below: project `llm`, service `ollama`, volume `ollama-models`.

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
```

- If the GPU list is empty, no region offers 1 × `l4-24` right now. Pick another GPU (the `.data.regions[].gpuDevices[].id` values in the same response) and change `GPU`.
- If the project already exists, its region must be in the GPU list, because a project's region can't be changed. If it isn't, use a new project name throughout.
- **GPUs need a human.** Free projects can't run GPU workloads, and GPU deployments need account credit. Ask the account owner to confirm billing and the hourly price before you create anything. Without GPU access, use the CPU fallback in step 5 and any region.
- An L4 runs 7B–8B models comfortably. For larger models set `GPU=a100-80` or `GPU=h100-80`.

**If you're an agent:** show the user the team, the region (`europe-west` below, unless they choose another from the GPU list), the GPU's hourly price and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services, and every service in it runs in the project's region. The region can't be changed later. Skip this step if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "llm", "region": "europe-west", "description": "Self-hosted LLMs" }'
```

To use another region from the step 2 list, change `region`. REST: `POST https://api.northflank.com/v1/projects`.

## Step 4: Build the GPU plan ID

GPU plans aren't listed by `/v1/plans`, which returns compute plans only. The resource plans docs give the rule: `nf-gpu-<gpuType>-<gpuCount>g`, with underscores in the type replaced by hyphens, and list the resulting IDs in a table (`nf-gpu-l4-24-1g` is 12 vCPU and 48 GB of system RAM with one L4). GPU plans bundle CPU and system RAM for the chosen GPU and count.

```bash
GPU_PLAN=$(jq -rn --arg g "$GPU" --argjson n "$COUNT" '"nf-gpu-\($g | gsub("_"; "-"))-\($n)g"'); echo "$GPU_PLAN"   # nf-gpu-l4-24-1g
```

The plan ID, `gpuType` (which keeps any underscore, as in `rtx_pro_6000-96`) and `gpuCount` must all match. If the API rejects the plan and names a different one, use the one it names.

## Step 5: Deploy the Ollama service

Write `ollama.json`. Replace `<OLLAMA_VERSION>`:

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

```bash
northflank put service deployment --projectId llm --noDefaults -o json --file ollama.json
```

REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/deployment`.

What the settings do:

- The `ollama/ollama` image's default command is `serve`, so no command override is needed.
- `timesliced: false` matches the GPU docs, which say Northflank cloud GPUs don't currently support timeslicing.
- `OLLAMA_KEEP_ALIVE` keeps a loaded model in GPU memory for 24 hours. The FAQ's default is 5 minutes, which means a slow first request after every idle period. It accepts durations like `24h` or `-1` (forever).
- `public: false` gives the port no public URL. Other services reach it at `http://ollama:11434`. Private ports are only reachable inside the project.
- The probes call `GET /api/tags` (list local models), which answers without a model loaded.

**CPU fallback.** With no GPU access, delete the `gpu` block and use a compute plan with enough RAM for the model, such as `nf-compute-400-16` (4 vCPU, 16 GB). Small models like `llama3.2` run on CPU, only more slowly.

## Step 6: Attach a volume at `/root/.ollama`

```bash
northflank get volume --projectId llm --volumeId ollama-models --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId llm --noDefaults -o json --input '{
  "name": "ollama-models",
  "mounts": [ { "containerMountPath": "/root/.ollama" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 20480 },
  "attachedObjects": [ { "id": "ollama", "type": "service" } ]
}'
```

REST: `POST https://api.northflank.com/v1/projects/{projectId}/volumes`. Things to know:

- `storageSize` is in MB and can only be increased later. `llama3.2` is about 2 GB, but 70B models are tens of GB, so size for what the user plans to pull.
- Attaching redeploys the service. A `ReadWriteOnce` volume limits the service to one instance.
- `create volume` returns 409 if the volume exists, hence the `get` guard.

## Step 7: Wait for the rollout

`COMPLETED` means rolled out and serving. GPU nodes can take a few minutes to schedule.

```bash
OK=0
for i in $(seq 1 90); do
  D=$(northflank get service --projectId llm --serviceId ollama --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
```

## Step 8: Pull a model

Ollama doesn't pull models on start. Pull one from inside the container with the `ollama` CLI. `exec` drops output without a TTY, so wrap it with `script` (macOS shown; on Linux use `script -qefc '<cmd>' /dev/null`):

```bash
script -q /dev/null northflank exec service --projectId llm --serviceId ollama \
  --shell-cmd 'sh -c' --cmd '"ollama pull llama3.2"'
script -q /dev/null northflank exec service --projectId llm --serviceId ollama \
  --shell-cmd 'sh -c' --cmd '"ollama list"'
```

Replace `llama3.2` with the model the user asked for (`gemma4`, `qwen3:8b`, …). It's written to the volume, so you only pull once. The REST equivalent is `POST /api/pull` with `{"model": "llama3.2", "stream": false}`, sent through the tunnel in the verify section.

## How to know it worked

You're done when all of these hold:

1. The model loads on the GPU. Run a prompt, then check the `PROCESSOR` column, which should read `100% GPU` (or `100% CPU` on the fallback):
   ```bash
   script -q /dev/null northflank exec service --projectId llm --serviceId ollama \
     --shell-cmd 'sh -c' --cmd '"ollama run llama3.2 hello && ollama ps"'
   ```
2. The API answers through a private tunnel. Read the log for the local address instead of assuming `127.0.0.1:11434`:
   ```bash
   northflank forward service --projectId llm --serviceId ollama --skipHostnames > /tmp/nf-ollama.log 2>&1 &
   FWD_PID=$!; sleep 10; cat /tmp/nf-ollama.log
   curl -s http://<LOCAL_ADDR>:<LOCAL_PORT>/api/tags | jq -r '.models[].name'          # lists llama3.2:latest
   curl -s http://<LOCAL_ADDR>:<LOCAL_PORT>/api/chat -d '{"model":"llama3.2","messages":[{"role":"user","content":"Say hi"}],"stream":false}' | jq -r '.message.content'
   kill $FWD_PID
   ```
3. No port is public:
   ```bash
   northflank get service --projectId llm --serviceId ollama --noDefaults -o json \
     | jq -e '[(.data // .).ports[] | select(.public == true)] | length == 0'
   ```
4. The volume is attached: `northflank get volume --projectId llm --volumeId ollama-models --noDefaults -o json | jq '(.data // .).attachedObjects'` lists `ollama`. After `northflank restart service --projectId llm --serviceId ollama`, `ollama list` still shows the model.

Other services in the project use `http://ollama:11434` for the native API, or `http://ollama:11434/v1` with any OpenAI SDK (Ollama's docs say the API key is "required but ignored", so pass any string).

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Create rejects the plan or GPU | Plan, `gpuType` and `gpuCount` don't match, the region lacks that GPU, or the project is free | Rebuild the plan in step 4; re-check regions; ask a human about billing |
| `GPU deployments are disabled due to insufficient credits` | No GPU credit | **Human step:** buy credit |
| `ollama ps` shows `100% CPU` on a GPU plan | `gpu` block missing or wrong image tag (e.g. `rocm`) | Check `deployment.gpu` with `get service`; use a plain version tag |
| Model gone after restart | Volume not attached, or mounted somewhere else | Check `attachedObjects` and `containerMountPath: /root/.ollama` |
| `connection refused` from another service | Wrong host or port | Use the service ID and port: `http://ollama:11434` |
| Out of memory loading a model | Model too large for the GPU | Pick a smaller or more quantised tag, or a bigger GPU |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write: team, region, GPU price per hour, and whether the project is new or reused.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs with `northflank get service logs --projectId llm --serviceId ollama` before changing anything.
- Show the user the GPU price per hour before step 5. GPU services bill while running; pause with `northflank pause service` when idle.
- Don't make port 11434 public. If the user insists, add `security.ip` `ALLOW` rules or `basic-auth` credentials to the port (see [port security policies](https://northflank.com/docs/v1/application/network/add-security-policies-for-ports)), and tell them the API has no auth of its own.
- Get explicit confirmation before deleting the volume, since it holds every downloaded model.

## FAQ

### Where should I host Ollama?
On a cloud host that can attach an NVIDIA GPU to a long-running container and keep `/root/.ollama` on a persistent volume, with the API kept on a private network. A GPU VM works if you're happy to manage drivers and Docker yourself. A container platform with GPU plans, as in this guide, runs the official Ollama image with the GPU and volume attached and lets your other services reach it privately.

### Can I run Ollama without a GPU?
Yes. Drop the `gpu` block and use a compute plan with enough RAM (step 5). Expect slower generation.

### How do I connect Open WebUI or my app?
Point it at `http://ollama:11434` from a service in the same project. For example, set Open WebUI's `OLLAMA_BASE_URL` to that address. See [how to deploy Open WebUI](deploy-open-webui-with-ollama.md).

### Where are the models stored?
Under `/root/.ollama` on the volume. To store them elsewhere, set `OLLAMA_MODELS` to another directory and mount the volume there.

### Should I use Ollama or vLLM?
Ollama is the simplest way to run many quantised models with one command each. For high-throughput serving of a single model, use [vLLM](deploy-llm-in-production-with-vllm.md).

## Conclusion

You now have Ollama on a cloud GPU, with models on a persistent volume and an API that only your own services and tunnels can reach.

**Next:** add a chat UI with [Open WebUI](deploy-open-webui-with-ollama.md), or put a [LiteLLM proxy](deploy-litellm-proxy.md) in front for API keys and spend tracking. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy Open WebUI with Ollama](deploy-open-webui-with-ollama.md)
- [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)
- [How to deploy LiteLLM proxy](deploy-litellm-proxy.md)
- [How to deploy AI agents in production](deploy-ai-agents-in-production.md)
