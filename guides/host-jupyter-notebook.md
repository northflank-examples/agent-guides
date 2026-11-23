<!--
agent-guide-meta:
  id: NF-STK-JUPYTER
  description: "How to host a Jupyter notebook server: the official JupyterLab Docker image with a fixed token, a persistent work volume, HTTPS access and an optional GPU."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://jupyter-docker-stacks.readthedocs.io/en/latest/using/selecting.html
    - https://jupyter-docker-stacks.readthedocs.io/en/latest/using/common.html
    - https://github.com/jupyter/docker-stacks/blob/main/images/base-notebook/Dockerfile
    - https://github.com/jupyter-server/jupyter_server/blob/main/jupyter_server/auth/identity.py
    - https://jupyter-server.readthedocs.io/en/latest/operators/security.html
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/gpu-workloads/deploy-gpus-on-northflank-cloud
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-STK-STREAMLIT, NF-STK-VLLM, NF-STK-OLLAMA, NF-DEP-002]
-->

# How to host a Jupyter notebook

To host a Jupyter notebook server, run an official Jupyter Docker Stacks image such as `quay.io/jupyter/scipy-notebook` on port 8888, set a long random `JUPYTER_TOKEN` so the server doesn't print a new token on every restart, mount a persistent volume at `/home/jovyan/work`, and serve it over HTTPS. Notebooks saved anywhere outside that volume disappear when the container is replaced.

This guide does it on Northflank with the CLI: a secret group for the token, a JupyterLab service from a pinned tag, a volume for notebooks and an optional GPU variant. You end with JupyterLab on a public HTTPS URL whose API answers only with the token, and a notebook file that survives a restart. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; the GPU variant needs a paid account.

## TL;DR: how to host a Jupyter notebook

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, the image tag and (for a GPU later) the region, then create the `jupyter` project.
3. Store a generated `JUPYTER_TOKEN` in a secret group and create a volume for `/home/jovyan/work`.
4. Run `quay.io/jupyter/scipy-notebook:<date-tag>` with `northflank put service deployment` on port 8888, attaching the volume.
5. Wait for the rollout, read the public host and open JupyterLab with the token.
6. Verify that `/api/status` returns 200 with the token and 403 without, and that a file in `work/` survives `northflank restart service`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

`jupyter lab` on a laptop is fine for one person on one machine. A hosted server needs three things that the defaults don't give you:

- **A fixed token.** Jupyter Server generates a random token at each start unless `JUPYTER_TOKEN` is set. Its source says that setting the token to an empty string disables authentication altogether, which it does not recommend. A notebook server runs arbitrary code, so an unauthenticated one on the internet is a remote shell.
- **Persistent storage.** The Docker Stacks images keep the user's files in `/home/jovyan`, and their docs mount host folders at `/home/jovyan/work`. On a platform, that becomes a volume.
- **A maintained image.** The Jupyter Docker Stacks images on `quay.io/jupyter` come in layers: `base-notebook` (JupyterLab), `minimal-notebook`, `scipy-notebook` (common scientific packages), `pytorch-notebook` and `tensorflow-notebook` (with CUDA variants), and `datascience-notebook` (Python, R and Julia). Pin a date tag, as the docs recommend for reproducibility.

So where should you host a Jupyter notebook? On something that keeps a container running with a persistent disk, terminates TLS, passes WebSockets (kernels talk over them) and, for model work, offers GPUs. Notebook SaaS products are quick but limit runtime, packages or hardware. A VM gives full control, but you set up TLS, auth and backups yourself. A container platform with volumes and GPU plans runs the official image with those pieces attached. For many users with separate servers, look at JupyterHub instead (see the FAQ).

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`**, to generate the token.
- **For the GPU variant only:** a paid account, because GPUs aren't available on free projects.

Names used below: project `jupyter`, secret group `jupyter-env`, volume `jupyter-work`, service `jupyter`.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Confirm where it will deploy

Before changing anything, pick an image tag and print the deployment target. This lists recent date tags for `scipy-notebook`; use one from before today (UTC):

```bash
curl -s "https://quay.io/api/v1/repository/jupyter/scipy-notebook/tag/?limit=50&onlyActiveTags=true" \
  | jq -r '.tags[].name' | grep -E '^[0-9]{4}-[0-9]{2}-[0-9]{2}$' | sort | tail -n 3
export JUPYTER_TAG=<DATE_TAG>

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId jupyter -o json >/dev/null 2>&1 \
  && echo "project: jupyter exists and will be reused" \
  || echo "project: jupyter will be created"
echo "image:   quay.io/jupyter/scipy-notebook:$JUPYTER_TAG"
```

A project's region is fixed. If you might want a GPU notebook later, check that the region offers one. This prints `l4-24` if `europe-west` (the region used below) has L4 GPUs:

```bash
curl -s https://api.northflank.com/v1/regions \
  | jq -r '.data.regions[] | select(.id == "europe-west") | .gpuDevices[]?.id' | grep -x l4-24
```

**If you're an agent:** show the user the team, the image tag, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. If you might want a GPU later, pick one that offers it (step 2 and the GPU section). Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "jupyter", "region": "europe-west" }'
export PROJECT_ID=jupyter
```

## Step 4: Create the token secret

```bash
JUPYTER_TOKEN=$(openssl rand -hex 32)
jq -n --arg t "$JUPYTER_TOKEN" '{
  name: "jupyter-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  secrets: { variables: { JUPYTER_TOKEN: $t } }
}' > jupyter-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file jupyter-env.json >/dev/null && rm jupyter-env.json
```

Keep `JUPYTER_TOKEN` in this shell for the checks, and **tell the user to copy it from the secret group into their password manager.** Put any API keys your notebooks need (`HF_TOKEN`, `OPENAI_API_KEY`…) in the same group so they never end up in notebook cells.

## Step 5: Create the work volume

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId jupyter-work --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "jupyter-work",
  "mounts": [ { "containerMountPath": "/home/jovyan/work" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 }
}'
```

`storageSize` is in MB and can only be increased. The volume covers `work/` only, so the image's own files in `/home/jovyan` (config, the conda environment) still come from the image and update with it.

## Step 6: Deploy JupyterLab

Write `jupyter.json`, replacing `<JUPYTER_TAG>`:

```json
{
  "name": "jupyter",
  "billing": { "deploymentPlan": "nf-compute-200-8" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "quay.io/jupyter/scipy-notebook:<JUPYTER_TAG>" },
    "docker": { "configType": "default" }
  },
  "ports": [ { "name": "http", "internalPort": 8888, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "TCP", "type": "readinessProbe", "port": 8888,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 6, "successThreshold": 1 }
  ],
  "createOptions": { "volumesToAttach": ["jupyter-work"] }
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file jupyter.json
```

Notes:

- **The default command** (`start-notebook.py`) runs `jupyter lab` on `JUPYTER_PORT`, which the image sets to 8888. The token comes from `JUPYTER_TOKEN` in the secret group.
- **A TCP probe** checks the port without needing the token. Most Jupyter API routes require it.
- **`nf-compute-200-8`** (2 vCPU, 8 GB) suits pandas-sized work. Kernel memory is the usual limit, so size the plan to your largest dataframe.
- **One instance.** A notebook server keeps kernels in memory, and the volume is `ReadWriteOnce`.

If the volume isn't attached after the first rollout, attach it explicitly (this redeploys the service):

```bash
northflank attach volume --projectId "$PROJECT_ID" --volumeId jupyter-work --noDefaults -o json \
  --input '{"nfObject": {"id": "jupyter", "type": "service"}}'
```

## Step 7: Wait for the rollout and open JupyterLab

```bash
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId jupyter --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId jupyter --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId jupyter --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST/lab"
fi
```

`COMPLETED` means rolled out and serving. If it prints `ROLLOUT_NOT_COMPLETE`, read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId jupyter`. **This step needs a human:** open `https://$HOST/lab` and paste the token into the login form. Don't share links with `?token=` in them, because the token ends up in browser history and logs.

## How to know it worked

You're done when all of these hold:

1. The API answers with the token and refuses without it (Jupyter Server accepts `Authorization: token <TOKEN>`):
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: token $JUPYTER_TOKEN" "https://$HOST/api/status"   # 200
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/api/status"                                         # 403
   ```
2. A file in `work/` survives a restart. Create one through the contents API, restart, and read it back:
   ```bash
   curl -s -X PUT -H "Authorization: token $JUPYTER_TOKEN" -H 'Content-Type: application/json' \
     "https://$HOST/api/contents/work/persist-check.txt" -d '{"type":"file","format":"text","content":"ok"}' >/dev/null
   northflank restart service --projectId "$PROJECT_ID" --serviceId jupyter --noDefaults -o json >/dev/null
   # wait for COMPLETED with the loop from step 7, then:
   curl -s -H "Authorization: token $JUPYTER_TOKEN" "https://$HOST/api/contents/work/persist-check.txt" | jq -r .content   # ok
   ```

## Optional: a GPU notebook

For deep learning, use the CUDA build of `pytorch-notebook` on a GPU plan. GPUs aren't available on free projects, and the project's region must offer the GPU. List regions that offer an L4 without a token:

```bash
curl -s https://api.northflank.com/v1/regions | jq -r '.data.regions[] | select(any(.gpuDevices[]?; .id == "l4-24")) | .id'
```

Create a second volume the same way as step 5, named `jupyter-gpu-work`. Then deploy with the CUDA tag (`cuda12-<JUPYTER_TAG>`), a GPU plan and a `deployment.gpu` block:

```json
{
  "name": "jupyter-gpu",
  "billing": { "deploymentPlan": "nf-gpu-l4-24-1g" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "quay.io/jupyter/pytorch-notebook:cuda12-<JUPYTER_TAG>" },
    "docker": { "configType": "default" },
    "gpu": { "enabled": true, "configuration": { "gpuType": "l4-24", "gpuCount": 1, "timesliced": false } }
  },
  "ports": [ { "name": "http", "internalPort": 8888, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "TCP", "type": "readinessProbe", "port": 8888,
      "initialDelaySeconds": 15, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 12, "successThreshold": 1 }
  ],
  "createOptions": { "volumesToAttach": ["jupyter-gpu-work"] }
}
```

GPU plan IDs follow `nf-gpu-<gpuType>-<gpuCount>g`; they aren't listed by `/v1/plans`. Check that PyTorch sees the GPU (Linux: `script -qefc '<cmd>' /dev/null`):

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId jupyter-gpu \
  --shell-cmd 'sh -c' --cmd '"python -c \"import torch; print(torch.cuda.is_available())\""'   # True
```

A GPU bills while the service runs, idle or not. Pause it when you're not using it with `northflank pause service --projectId "$PROJECT_ID" --serviceId jupyter-gpu`. For serving a model rather than experimenting, see [how to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md).

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Login page rejects the token | `JUPYTER_TOKEN` not injected, or a password was configured | Check the `jupyter-env` group; restart |
| `Permission denied` saving to `work/` | Volume not writable by `jovyan` (UID 1000) | Save once in `/home/jovyan` to confirm the server works, then see [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) |
| Notebooks gone after a restart | Saved outside `work/`, or volume not attached | Keep files under `work/`; check `northflank get volume … --volumeId jupyter-work` |
| Kernel dies on large data | Out of memory | Move to a larger plan |
| Kernel keeps reconnecting | WebSocket dropped by a proxy or network | Reload the tab; long silent cells keep running server-side |
| `torch.cuda.is_available()` is `False` | Non-CUDA tag, or no `deployment.gpu` block | Use `cuda12-…` and a GPU plan |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Never print `JUPYTER_TOKEN` or put it in a URL you log. Never set it to an empty string to "fix" auth. That turns the server into an open remote shell.
- Install packages from a notebook with `%pip install` for experiments, but tell the user these vanish on redeploy. For lasting packages, build a custom image `FROM` the same tag.
- Ask before deploying the GPU variant, because it bills by the hour.
- Get explicit confirmation before deleting the volume; it holds the user's notebooks.

## FAQ

### Where should I host my Jupyter notebook?
On a host that keeps a container running with a persistent volume, HTTPS and WebSocket support, plus GPUs if you train models. A container platform running the official Docker Stacks image covers all of that. A VM works if you manage TLS, auth and backups yourself, and hosted notebook services are fine until you hit their runtime or hardware limits.

### Should I use a password instead of a token?
You can. The Docker Stacks docs show `--PasswordIdentityProvider.hashed_password` with a hash from `jupyter_server.auth.passwd()`. A long random token in a secret group does the same job with less setup. Either way, don't run without one.

### What about JupyterHub for a team?
JupyterHub spawns a separate server per user with central login. It's a different deployment (a hub plus per-user containers). One shared JupyterLab with one token is only suitable for a single user or a small trusted group.

### How do I add Python packages permanently?
Build your own image `FROM quay.io/jupyter/scipy-notebook:<JUPYTER_TAG>` with `RUN pip install …` (or `mamba install`), push it, and change the service's `imagePath`. The volume keeps your notebooks across the change.

## Conclusion

You now have JupyterLab on a pinned official image, behind a fixed token, with notebooks on a persistent volume and an optional GPU variant for deep learning.

**Next:** add a custom domain by [linking a domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port), and back up the work volume. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Streamlit app](deploy-streamlit-app.md)
- [How to deploy an LLM in production with vLLM](deploy-llm-in-production-with-vllm.md)
- [How to host Ollama in the cloud](host-ollama-in-the-cloud.md)
- [How to deploy a Gradio app](deploy-gradio-app.md)
