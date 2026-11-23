<!--
agent-guide-meta:
  id: NF-STK-GRADIO
  description: "How to deploy a Gradio app with Docker: server_name 0.0.0.0 on port 7860, the queue and streaming over HTTPS, password auth and an optional GPU for model demos."
  sources:
    - https://www.gradio.app/guides/deploying-gradio-with-docker
    - https://www.gradio.app/guides/environment-variables
    - https://www.gradio.app/guides/sharing-your-app
    - https://www.gradio.app/guides/setting-up-a-demo-for-maximum-performance
    - https://www.gradio.app/docs/gradio/blocks
    - https://www.gradio.app/docs/python-client/client
    - https://pypi.org/project/gradio/
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/network/load-balancing
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
    - https://northflank.com/docs/v1/application/gpu-workloads/deploy-gpus-on-northflank-cloud
  related: [NF-STK-STREAMLIT, NF-STK-VLLM, NF-STK-SHINY, NF-STK-DASH]
-->

# How to deploy a Gradio app

To deploy a Gradio app, run it in a long-lived Docker container with `GRADIO_SERVER_NAME=0.0.0.0` and `GRADIO_SERVER_PORT=7860`, enable the queue so long predictions stream progress instead of timing out, protect it with `launch(auth=...)` using credentials from environment variables, and add a GPU only if the model needs one. Gradio binds to `127.0.0.1` by default, which is why a containerized app is often unreachable.

This guide does it on Northflank with the CLI: a `Dockerfile` built from Git, a password in a secret group, health checks on port 7860, and an optional GPU variant with a model cache volume. You end with a public HTTPS URL behind a login, verified with `curl` and a `gradio_client` prediction. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; GPUs need billing a human enables.

## TL;DR: how to deploy a Gradio app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Gradio app and confirm the team, repository, branch and project before changing anything, then give Northflank access to the repository (once).
3. Call `demo.queue(...).launch(auth=...)` in `app.py`, add a `Dockerfile` with `ENV GRADIO_SERVER_NAME=0.0.0.0 GRADIO_SERVER_PORT=7860`, and push.
4. Create a project, store `GRADIO_USERNAME` and `GRADIO_PASSWORD` in a secret group, and deploy with `northflank put service combined --file gradio.json`: HTTP port 7860, one instance, plus a GPU plan and cache volume if the model needs one.
5. Wait for the deploy, then check that the live build is your pushed commit and that `gradio_client` gets a prediction with the password.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why a long-running container for Gradio?

A Gradio app is a Python web server that usually holds a model in memory. Requests go through a queue, and the Gradio performance guide says results come back over server-sent events (SSE), which lets the server push progress and ETA updates without the request timing out. Older Gradio versions used WebSockets for the same job. Either way, the host has to keep a process running, keep the model loaded between requests and pass long-lived streaming connections through. The Gradio Docker guide also notes that with several replicas, a user's requests must reach the same instance, because the frontend opens several connections per event.

So when choosing where to deploy a Gradio app, Hugging Face Spaces is the quickest for public demos. A cloud VM works if you manage the proxy, TLS and drivers yourself. Serverless functions don't fit a model that takes a minute to load. A container platform runs your own image, with a GPU when the model needs one. On Northflank, a **combined service** builds the `Dockerfile` from Git on every push and serves it on an HTTPS URL; a **volume** keeps downloaded model weights across restarts.

## Prerequisites

- **A Gradio app that runs locally.** This guide assumes `app.py` at the repo root and a `requirements.txt` that pins `gradio` (and `torch`, `transformers`… if it runs a model). Run it, then open `http://localhost:7860`:
  ```bash
  pip install -r requirements.txt && python app.py
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **GPU variant only: billing that allows GPUs.** API-backed demos and small CPU models run on a compute plan. Large PyTorch models need a GPU, and free projects don't support GPU workloads, so a human confirms billing first.

Names used below: project `gradio-app`, secret group `gradio-secrets`, service `gradio`, volume `hf-cache`.

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

Before changing anything, confirm that this guide fits the app and print the deployment target. Run this from the repository root:

```bash
grep -hiE '^[[:space:]"]*gradio\b' $(ls requirements.txt pyproject.toml 2>/dev/null) /dev/null || echo "gradio: missing"
ls app.py requirements.txt pyproject.toml poetry.lock uv.lock Pipfile.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId gradio-app -o json >/dev/null 2>&1 \
  && echo "project: gradio-app exists and will be reused" \
  || echo "project: gradio-app will be created"
```

For the GPU variant, also list the regions that offer the L4 GPU used below (public endpoint, no token needed). The project's region must be one of them:

```bash
curl -s https://api.northflank.com/v1/regions | jq -r '.data.regions[] | select(any(.gpuDevices[]?; .id == "l4-24")) | .id'
```

- If it prints `gradio: missing`, this isn't a Gradio app. Stop and use a guide for your framework.
- The lockfile tells you the package manager. Step 5 assumes `requirements.txt`; for Poetry, uv or Pipenv, export a `requirements.txt` or swap the install command.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them; for a GPU, one from step 2); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "gradio-app", "region": "europe-west" }'
export PROJECT_ID=gradio-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Set up `app.py` and the `Dockerfile`

```python
import os
import gradio as gr

def greet(name: str) -> str:
    return f"Hello, {name}!"

demo = gr.Interface(fn=greet, inputs="text", outputs="text")

auth = None
if os.environ.get("GRADIO_USERNAME"):
    auth = (os.environ["GRADIO_USERNAME"], os.environ["GRADIO_PASSWORD"])

# server_name and server_port come from GRADIO_SERVER_NAME / GRADIO_SERVER_PORT
demo.queue(default_concurrency_limit=2, max_size=50).launch(auth=auth)
```

- **`auth`** accepts one `(username, password)` tuple, a list of them, or a function that returns `True` for valid logins, per the Gradio sharing guide. It notes that login needs third-party cookies enabled in the browser, which Safari and Chrome Incognito block by default.
- **`default_concurrency_limit`** defaults to `1`, so one prediction runs at a time per event and everyone else waits in the queue. Raise it while performance and memory hold. A GPU model often wants 1; an API-backed demo can go higher.
- **`max_size`** caps the queue (default unlimited), so users see "queue full" instead of waiting forever.
- Load the model at module level, not inside `greet`, so it loads once per process.

Then the `Dockerfile`, based on Gradio's own Docker guide:

```dockerfile
FROM python:3.13-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 \
    GRADIO_SERVER_NAME=0.0.0.0 GRADIO_SERVER_PORT=7860 \
    GRADIO_ANALYTICS_ENABLED=False
WORKDIR /usr/src/app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 7860
CMD ["python", "app.py"]
```

The Gradio environment variables reference lists `GRADIO_SERVER_NAME` (default `127.0.0.1`), `GRADIO_SERVER_PORT` (default `7860`) and `GRADIO_ANALYTICS_ENABLED`. Values passed to `launch()` in code override them. Don't set `share=True`: that creates a temporary public tunnel through Gradio's servers and isn't needed here. Add a `.dockerignore` with `.venv`, `__pycache__`, `.env` and `.git`.

Northflank builds what's in your repository, not the files on your machine, so commit and push them:

```bash
git add app.py Dockerfile .dockerignore requirements.txt
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Store the login (and tokens) in a secret group

Generate a password locally and send it straight to the group. Add `HF_TOKEN` if the model is gated on Hugging Face:

```bash
export GRADIO_PASSWORD=$(openssl rand -base64 24)
jq -n --arg p "$GRADIO_PASSWORD" --arg hf "${HF_TOKEN:-}" '{
  name: "gradio-secrets", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "gradio", type: "service" } ] },
  secrets: { variables: ({ GRADIO_USERNAME: "admin", GRADIO_PASSWORD: $p } + (if $hf == "" then {} else { HF_TOKEN: $hf } end)) }
}' > gradio-secrets.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file gradio-secrets.json >/dev/null && rm gradio-secrets.json
```

Keep `GRADIO_PASSWORD` in this shell for the checks, and tell the user it's stored in the `gradio-secrets` group. Restricting the group to a service that doesn't exist yet hasn't been confirmed to work; if the call is rejected, use `"restricted": false`.

## Step 7: Create the service

Size memory for the model: `nf-compute-100-4` (4 GB) is a reasonable start for a small CPU model, `nf-compute-50` for an API-backed demo (`curl -s https://api.northflank.com/v1/plans` lists the others). Write `gradio.json` (CPU):

```json
{
  "name": "gradio",
  "billing": { "deploymentPlan": "nf-compute-100-4" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 7860, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "TCP", "type": "startupProbe", "port": 7860,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 60, "successThreshold": 1 },
    { "protocol": "TCP", "type": "readinessProbe", "port": 7860,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file gradio.json
```

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined` with the same body. If no build appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId gradio --noDefaults -o json --input '{}'`.

Gradio opens the port only after `app.py` has loaded the model, so the startup probe gives it up to about 10 minutes (60 × 10 s) before anything can restart it. TCP probes avoid depending on what the login page returns.

**GPU variant.** Change the plan and add a `deployment.gpu` block. `billing.buildPlan` is optional; leave it out to use the default build plan, and only set it if builds run out of memory, using a build plan your team has access to (an unavailable one fails with `Build plan not found`). CUDA-sized wheels make that more likely here. PyPI's Linux `torch` wheels include CUDA libraries, so the `Dockerfile` stays the same:

```json
  "billing": { "deploymentPlan": "nf-gpu-l4-24-1g" },
  "deployment": { "instances": 1, "docker": { "configType": "default" },
    "gpu": { "enabled": true, "configuration": { "gpuType": "l4-24", "gpuCount": 1, "timesliced": false } } },
```

GPU plan IDs follow `nf-gpu-<gpuType>-<count>g` in the resource plans docs and aren't listed by `/v1/plans`. Show the user the hourly price from `/v1/regions` before creating it. Then keep downloaded weights on a volume, so restarts don't download them again (the container runs as root, so the cache is under `/root`):

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId hf-cache --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "hf-cache",
  "mounts": [ { "containerMountPath": "/root/.cache/huggingface" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 20480 },
  "attachedObjects": [ { "id": "gradio", "type": "service" } ]
}'
```

`storageSize` is in MB and can only be increased. Attaching redeploys the service and limits it to one instance. For a full model server behind Gradio, see [how to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md).

**Instances.** Keep `instances: 1`. For more, Gradio's Docker guide requires session affinity; add `"loadBalancing": { "mode": "consistentHash", "consistentHash": { "mode": "ip" } }` and test it first. Raising `default_concurrency_limit` on one bigger instance is usually simpler.

## Step 8: Wait for the deploy and get your URL

The first build takes a few minutes, longer with model wheels. This loop checks every 10 seconds and prints your URL when the app is live. You can also watch the build in the Northflank dashboard.

```bash
OK=0
for i in $(seq 1 120); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId gradio --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId gradio --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId gradio --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId gradio`. The troubleshooting table covers the common causes.

`COMPLETED` means rolled out and serving. For the GPU variant, run the loop again after the volume attach.

## How to know it worked

You're done when all of these hold. Check 6 applies to the GPU variant only.

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId gradio --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. Gradio is listening on all interfaces:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId gradio --lineLimit 200 --textIncludes "Running on local URL"
   ```
   This prints `Running on local URL:  http://0.0.0.0:7860`.
3. The page (login page, when `auth` is set) loads:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"     # 200
   ```
4. A prediction goes through the queue with the password. Run this locally with `pip install gradio_client`:
   ```bash
   python3 - <<EOF
   from gradio_client import Client
   c = Client("https://$HOST/", auth=("admin", "$GRADIO_PASSWORD"), verbose=False)
   print(c.predict("Northflank", api_name="/greet"))
   EOF
   ```
   This prints `Hello, Northflank!`. Current Gradio names a `gr.Interface` endpoint after its function (`/greet` here; older versions used `/predict`). For other apps, `c.view_api()` lists the endpoint names.
5. In a browser, log in and run the demo; progress shows while it runs and file or image outputs display. Ask the user or use a browser tool.
6. GPU variant only: the GPU is visible. `exec` drops output without a TTY, so wrap it in `script` (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId gradio \
     --shell-cmd 'sh -c' --cmd '"python -c \"import torch; print(torch.cuda.is_available())\""'
   ```
   This prints `True`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |
| Logs show `http://127.0.0.1:7860`, 502/503 | `server_name` not set | Keep `GRADIO_SERVER_NAME=0.0.0.0`, and don't pass `server_name="127.0.0.1"` in `launch()` |
| Logs show port 7861 | Port 7860 was busy, or `server_port` set elsewhere | Set `GRADIO_SERVER_PORT=7860`; match `internalPort` |
| Container restarts before the app is up | Model load longer than the startup probe allows | Raise `failureThreshold` on the startup probe |
| Users wait a long time with "In queue" | `default_concurrency_limit` is 1 | Raise it if memory allows, or move to a bigger plan |
| Login page loops back to itself | Browser blocks third-party cookies | Allow cookies for the site, or try another browser |
| `torch.cuda.is_available()` is `False` | No GPU block, or a CPU-only `torch` wheel | Check `deployment.gpu`; don't install from the CPU wheel index |
| `CUDA out of memory` | Model too large for the GPU | Use a smaller model, half precision or a bigger GPU |

Logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId gradio` and `northflank get service logs --projectId "$PROJECT_ID" --serviceId gradio --lineLimit 200`.

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print `GRADIO_PASSWORD` or `HF_TOKEN` in logs or chat; tell the user where it's stored.
- Don't remove `auth` to "fix" a login problem without asking. A public model demo can run up GPU or API costs.
- Show the user the GPU price per hour before creating a GPU service, and pause it with `northflank pause service` when idle.
- Don't delete the service, volume or project without explicit confirmation.

## FAQ

### Where should I deploy my Gradio app?
For a quick public demo, Hugging Face Spaces. For a private app with your own secrets, a login, a database or a GPU you control, a container platform runs the same `Dockerfile` with HTTPS. A cloud VM works if you'll manage drivers, the proxy and TLS yourself.

### Why does my Gradio app say "running on 127.0.0.1" in Docker?
That's Gradio's default. Set `GRADIO_SERVER_NAME=0.0.0.0` or `launch(server_name="0.0.0.0")` so it accepts connections from outside the container.

### Do I need `share=True`?
No. `share=True` creates a temporary `gradio.live` tunnel for local testing. A deployed app already has its own public URL.

### Can I serve Gradio under a path or inside FastAPI?
Yes. Mount it with `gr.mount_gradio_app(app, demo, path="/gradio")` in a FastAPI app, or set `root_path` (`GRADIO_ROOT_PATH`) when a proxy serves it under a sub-path. See [how to deploy a FastAPI app](deploy-fastapi-app.md).

## Conclusion

You now have a Gradio app built from Git, listening on `0.0.0.0:7860` behind HTTPS and a login, with predictions going through the queue and streaming progress, and a GPU variant with a persistent model cache when the demo needs one.

**Next:** put a dedicated model server behind the UI with [how to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md), or add speech with [how to deploy Whisper](deploy-whisper.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Streamlit app](deploy-streamlit-app.md)
- [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)
- [How to deploy ComfyUI](deploy-comfyui.md)
- [How to deploy Whisper](deploy-whisper.md)
- [How to deploy a FastAPI app](deploy-fastapi-app.md)
