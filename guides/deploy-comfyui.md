<!--
agent-guide-meta:
  id: NF-STK-COMFYUI
  description: "How to deploy ComfyUI on a cloud GPU: build it from a small Dockerfile, keep models and outputs on a volume, protect the UI with basic auth and verify CUDA."
  sources:
    - https://github.com/Comfy-Org/ComfyUI
    - https://github.com/Comfy-Org/ComfyUI/blob/v0.46.1/comfy/cli_args.py
    - https://github.com/Comfy-Org/ComfyUI/blob/v0.46.1/folder_paths.py
    - https://github.com/Comfy-Org/ComfyUI/blob/v0.46.1/server.py
    - https://huggingface.co/Comfy-Org/stable-diffusion-1.5
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
    - https://northflank.com/docs/v1/application/gpu-workloads/deploy-gpus-on-northflank-cloud
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/volumes/attach-volume
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-STK-WHISPER, NF-STK-VLLM, NF-STK-OLLAMA, NF-DEP-001]
-->

# How to deploy ComfyUI

To deploy ComfyUI, build a container from the official repository with CUDA-enabled PyTorch, start it with `python main.py --listen 0.0.0.0 --port 8188`, run it on an NVIDIA GPU, keep models, custom nodes and outputs on a persistent volume through `--base-directory`, and put authentication in front of it. ComfyUI has no login of its own, so an open port lets anyone run jobs on your GPU and install custom nodes.

This guide does it on Northflank with the CLI: ComfyUI built from a short Dockerfile in your Git repository, on an L4 GPU, with a `/data` volume and basic auth on the public port. You end with an HTTPS URL that asks for a password, reports a CUDA device and lists a downloaded checkpoint. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run; a human logs in once, confirms GPU billing and shares the repository with Northflank.

## TL;DR: how to deploy ComfyUI

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that an L4 GPU is available in the region and confirm the team, repository, branch and project before changing anything.
3. Give Northflank access to your repository (once), create the project and check access, then push a `Dockerfile` that clones ComfyUI `v0.46.1`, installs PyTorch from the CUDA index and runs `main.py --listen 0.0.0.0 --port 8188 --base-directory /data`.
4. Create a `/data` volume and the service with `northflank put service combined`: GPU plan `nf-gpu-l4-24-1g`, port 8188 with basic auth, and a startup probe on `/system_stats`.
5. Download a checkpoint into `/data/models/checkpoints` with `northflank exec service`.
6. Verify that the live build is your pushed commit, the URL returns 401 without credentials, `/system_stats` shows a `cuda` device and the checkpoint is listed.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

The ComfyUI manual install is: install PyTorch for your GPU, then `pip install -r requirements.txt` and `python main.py`. A server changes four things:

- **Listen address.** `--listen` defaults to `127.0.0.1`; containers need `0.0.0.0`. The default port is 8188.
- **Storage.** `--base-directory` moves `models`, `custom_nodes`, `input`, `output`, `temp` and `user` under one path. Pointing it at a volume keeps checkpoints (often 2–20 GB each), installed nodes and generated images across restarts and redeploys.
- **GPU.** PyTorch wheels from `https://download.pytorch.org/whl/cu130` (the index the ComfyUI README gives for NVIDIA) bring their own CUDA runtime; the host supplies the driver.
- **Access control.** ComfyUI has no authentication. Anyone who reaches the port can queue prompts, upload files and, with ComfyUI-Manager, install code. Either keep the port private and reach it through a tunnel, or add basic auth at the edge, as this guide does.

Comfy-Org doesn't publish an official Docker image, so you build from the official repository at a pinned tag rather than trust a third-party image.

Where to host ComfyUI follows from those needs: an NVIDIA GPU with enough VRAM for your models (SD 1.5 runs in a few GB, SDXL and Flux need 12–24 GB), a long-running process, a large persistent disk and a way to keep strangers out. Serverless GPU functions fit API-style batch jobs but not an interactive editor with installed nodes. A GPU VM works if you handle drivers, TLS and auth yourself. A container platform with GPU plans runs your image with the GPU, volume and auth attached.

## Prerequisites

- **A GitHub, GitLab or Bitbucket repository you can push to.** It only needs to hold the `Dockerfile` you add in step 5; an empty repository is fine.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **GPU access.** Free projects don't support GPU workloads. **This needs a human:** the account owner confirms GPU billing.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Check the app and confirm where it will deploy

Before changing anything, check that the region has the GPU this guide uses and print the deployment target. Run this from the repository root. The first command is a public endpoint (no token needed) and prints the L4 price in US cents per GPU-hour for each region that has one:

```bash
curl -s https://api.northflank.com/v1/regions | jq -r '.data.regions[]
  | select(any(.gpuDevices[]?; .id == "l4-24")) | "\(.id)\t\(.gpuDevices[] | select(.id == "l4-24") | .pricing.onDemand) cents/h"'
ls Dockerfile 2>/dev/null && grep -inE '^(FROM|CMD)' Dockerfile

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId comfy -o json >/dev/null 2>&1 \
  && echo "project: comfy exists and will be reused" \
  || echo "project: comfy will be created"
```

- If `europe-west` isn't in the list, pick a listed region in step 4. If no region is listed, L4 GPUs aren't offered right now: stop.
- GPU plans aren't listed by `/v1/plans`; the resource plans docs give the ID rule `nf-gpu-<gpuType>-<gpuCount>g`. For SDXL or Flux at full precision, pick a larger GPU.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it, or use a separate repository for ComfyUI.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another), the GPU price per hour and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick a region with L4 GPUs from step 2, close to your users; it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "comfy", "region": "europe-west" }'
export PROJECT_ID=comfy

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

Names used below: project `comfy`, volume `comfyui-data`, service `comfyui`.

## Step 5: Add the Dockerfile

Commit this as `Dockerfile` at the repo root:

```dockerfile
FROM python:3.12-slim
ARG COMFYUI_VERSION=v0.46.1
ARG TORCH_INDEX=https://download.pytorch.org/whl/cu130
RUN apt-get update && apt-get install -y --no-install-recommends git curl \
 && rm -rf /var/lib/apt/lists/*
RUN git clone --depth 1 --branch "$COMFYUI_VERSION" https://github.com/Comfy-Org/ComfyUI.git /app
WORKDIR /app
RUN pip install --no-cache-dir torch torchvision torchaudio --extra-index-url "$TORCH_INDEX" \
 && pip install --no-cache-dir -r requirements.txt
EXPOSE 8188
CMD ["python", "main.py", "--listen", "0.0.0.0", "--port", "8188", "--base-directory", "/data"]
```

`v0.46.1` was the latest release tag at the time of writing; list newer ones with `git ls-remote --tags https://github.com/Comfy-Org/ComfyUI.git`. The ComfyUI README says Python 3.13 is well supported; 3.12 is used here for the widest custom-node compatibility.

Northflank builds what's on the remote branch, not your local working tree, so push it before step 8:

```bash
git add Dockerfile
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create the volume

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId comfyui-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "comfyui-data",
  "mounts": [ { "containerMountPath": "/data" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 51200 }
}'
```

`storageSize` is in MB and can only be increased. 50 GB holds a handful of checkpoints, LoRAs and outputs. The volume limits the service to one instance, which suits ComfyUI's single queue.

## Step 7: Generate the basic-auth password

```bash
COMFY_USER=comfy
COMFY_PASS=$(openssl rand -hex 16)
```

Keep `COMFY_PASS` in this shell for the checks. It's stored in the port's security settings, so **tell the user to save it in their password manager**; you can rotate it by re-running step 8 with a new value.

## Step 8: Create the ComfyUI service

Write the service definition with the password injected by `jq`, then create it:

```bash
jq -n --arg u "$COMFY_USER" --arg p "$COMFY_PASS" '{
  name: "comfyui",
  billing: { deploymentPlan: "nf-gpu-l4-24-1g" },
  deployment: {
    instances: 1,
    docker: { configType: "default" },
    gpu: { enabled: true, configuration: { gpuType: "l4-24", gpuCount: 1, timesliced: false } }
  },
  vcsData: { projectUrl: "https://github.com/<OWNER>/<REPO>", projectType: "github", projectBranch: "main" },
  buildSettings: { dockerfile: { buildEngine: "buildkit", dockerFilePath: "/Dockerfile", dockerWorkDir: "/" } },
  ports: [ { name: "http", internalPort: 8188, public: true, protocol: "HTTP",
    security: { credentials: [ { username: $u, password: $p, type: "basic-auth" } ] } } ],
  healthChecks: [
    { protocol: "HTTP", type: "startupProbe", path: "/system_stats", port: 8188,
      initialDelaySeconds: 20, periodSeconds: 10, timeoutSeconds: 5, failureThreshold: 60, successThreshold: 1 },
    { protocol: "HTTP", type: "livenessProbe", path: "/system_stats", port: 8188,
      initialDelaySeconds: 10, periodSeconds: 30, timeoutSeconds: 5, failureThreshold: 3 }
  ],
  createOptions: { volumesToAttach: ["comfyui-data"] }
}' > comfyui.json
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file comfyui.json >/dev/null && rm comfyui.json
```

Notes:

- **Repository.** Use the same repository URL, provider and branch as in step 4.
- **Build plan.** `billing.buildPlan` is optional; leave it out to use the default build plan, and only set it if builds run out of memory, using a build plan your team has access to (an unavailable one fails with `Build plan not found`). Installing PyTorch with CUDA is large, so this build is the likeliest to need one.
- **`timesliced: false`**: the GPU docs say Northflank cloud GPUs don't support timeslicing.
- **Basic auth** applies to the public URL; browsers show a login prompt. Services in the same project reach `http://comfyui:8188` without it.
- **Private alternative.** Set `public: false`, drop `security`, and open the UI through a tunnel: `northflank forward service --projectId "$PROJECT_ID" --serviceId comfyui`.

If no build starts within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId comfyui --noDefaults -o json --input '{}'`. If the volume shows no attachment afterwards, attach it explicitly (this redeploys):

```bash
northflank attach volume --projectId "$PROJECT_ID" --volumeId comfyui-data --noDefaults -o json \
  --input '{"nfObject": {"id": "comfyui", "type": "service"}}'
```

## Step 9: Wait for the deploy and get your URL

The first build downloads several GB of wheels; allow 10–20 minutes. This loop checks every 10 seconds and prints your URL when ComfyUI is live.

```bash
OK=0
for i in $(seq 1 120); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId comfyui --noDefaults -o json)
  B=$(echo "$S" | jq -r '(.data // .).status.build.status // "NONE"')
  D=$(echo "$S" | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "build=$B deployment=$D"
  case "$B" in FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo BUILD_FAILED; break;; esac
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  [ "$B" = "SUCCESS" ] && [ "$D" = "COMPLETED" ] && { OK=1; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId comfyui --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId comfyui --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId comfyui`. The troubleshooting table covers the common causes.

## Step 10: Download a checkpoint onto the volume

Stable Diffusion 1.5 (about 2 GB, no licence gate) is enough to prove the pipeline:

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId comfyui --shell-cmd 'sh -c' \
  --cmd '"mkdir -p /data/models/checkpoints && curl -fL -o /data/models/checkpoints/v1-5-pruned-emaonly-fp16.safetensors https://huggingface.co/Comfy-Org/stable-diffusion-1.5/resolve/main/v1-5-pruned-emaonly-fp16.safetensors && ls -lh /data/models/checkpoints"'
```

(Linux: `script -qefc '<cmd>' /dev/null`.) For gated models, **a human** must accept the licence on Hugging Face and provide a token; pass it as a header (`-H "Authorization: Bearer $HF_TOKEN"`) without printing it.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId comfyui --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The public URL requires credentials:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"                              # 401
   ```
3. ComfyUI sees the GPU:
   ```bash
   curl -s -u "$COMFY_USER:$COMFY_PASS" "https://$HOST/system_stats" | jq '.devices[] | {name, type, vram_total}'
   ```
   Expect `"type": "cuda"` and an NVIDIA L4 device name. `cpu` means PyTorch couldn't use the driver.
4. The checkpoint is visible to the loader node:
   ```bash
   curl -s -u "$COMFY_USER:$COMFY_PASS" "https://$HOST/object_info/CheckpointLoaderSimple" | grep -o 'v1-5-pruned-emaonly-fp16.safetensors' | head -n1
   ```
5. The volume is attached: `northflank get volume --projectId "$PROJECT_ID" --volumeId comfyui-data --noDefaults -o json | jq '(.data // .).attachedObjects'` lists `comfyui`.
6. **A human** opens `https://$HOST`, signs in, selects the checkpoint in the default workflow and clicks Run. The image appears, and is saved under `/data/output`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `/system_stats` shows `cpu`, logs mention the driver being too old | `cu130` wheels need a newer driver | Check `nvidia-smi` via exec, then set `TORCH_INDEX` to `…/whl/cu128` and rebuild (the ComfyUI README asks for cu130 or newer on 20-series and newer GPUs, so treat this as a fallback) |
| Build fails with out-of-memory | Default build plan too small for PyTorch | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| Create rejects the plan or GPU | Plan, `gpuType` or region don't match, or the project is free | Re-check regions; ask a human about billing |
| Startup probe fails, container restarts | Slow first start or custom node import error | Read `northflank get service logs`; raise `failureThreshold` |
| Checkpoint missing after restart | Volume not attached; file written to the container | Check `attachedObjects`; re-download into `/data` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |
| CUDA out of memory | Model too large for the GPU | Use fp16/fp8 weights or a larger GPU (`--lowvram` in the `CMD` only has an effect when dynamic VRAM is off) |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never deploy ComfyUI on a public port without `security.credentials` or an IP allow-list. Never print `COMFY_PASS` or Hugging Face tokens.
- Show the GPU price per hour from step 2 before step 8, and pause with `northflank pause service` when idle.
- Installing custom nodes runs third-party code with access to the volume. Ask before installing any.
- Confirm before deleting the volume; it holds models and every generated image.

## FAQ

### Where should I host ComfyUI?
On a cloud host that gives a long-running container an NVIDIA GPU with enough VRAM for your models, a large persistent disk and authentication in front of the UI. A GPU VM works if you'll set up drivers, TLS and auth yourself; a container platform with GPU plans, as here, attaches the GPU, volume and basic auth to your image.

### Can I install ComfyUI-Manager?
Yes. Add `pip install -r manager_requirements.txt` to the Dockerfile and `--enable-manager` to the `CMD`, as the ComfyUI README describes. Nodes it installs land in `/data/custom_nodes`, so they persist.

### Can I call ComfyUI as an API?
Yes. Export a workflow in API format and `POST` it to `/prompt`, then poll `/history/{prompt_id}`. Send the basic-auth credentials, or call `http://comfyui:8188` from another service in the project.

### How do I upgrade?
Change `COMFYUI_VERSION` in the Dockerfile and push; the service rebuilds. Models and outputs stay on the volume.

## Conclusion

You now have ComfyUI on a cloud GPU, built from the official repository, with models, nodes and outputs on a volume and the UI behind basic auth.

**Next:** add an IP allow-list on the port, or serve text models next to it with [vLLM](deploy-llm-in-production-with-vllm.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)
- [How to deploy Whisper](deploy-whisper.md)
- [How to host Ollama in the cloud](host-ollama-in-the-cloud.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
