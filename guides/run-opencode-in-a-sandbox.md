<!--
agent-guide-meta:
  id: NF-SBX-OPENCODE
  description: "How to run OpenCode in a sandbox: a headless opencode run in an isolated cloud microVM with a workspace volume and secret provider key, and a patch copied back."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/sandboxes/quickstart
    - https://northflank.com/docs/v1/application/sandboxes/examples
    - https://northflank.com/docs/v1/application/sandboxes/configure-and-manage-sandboxes
    - https://northflank.com/docs/v1/application/sandboxes/sandboxes-on-northflank-cloud
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/application/network/configure-network-policies
    - https://opencode.ai/docs/
    - https://opencode.ai/docs/cli/
    - https://opencode.ai/docs/providers/
    - https://opencode.ai/docs/permissions/
    - https://models.dev/api.json
  related: [NF-SBX-CODEX, NF-JOB-004, NF-AI-003]
-->

# How to run OpenCode in a sandbox

To run OpenCode in a sandbox, start an isolated cloud environment with its own kernel and a workspace volume, inject your model provider key as a secret, install OpenCode and clone the repository inside it, run `opencode run --standalone --auto` headless, then copy the result back as a `git diff`. This is a cloud sandbox, not OpenCode's permission rules, which only limit what it may do on your machine.

This guide does it on Northflank with the Python SDK and CLI: a microVM sandbox from `node:22-bookworm`, the provider key in a restricted secret group, OpenCode installed onto a `/workspace` volume so it survives pauses, and a background run you poll, then pause or delete. You end with a patch to review before applying. Every step is a script a developer or coding agent (Claude Code, Codex, Cursor) can run as-is; a human logs in and creates the API token once.

## TL;DR: how to run OpenCode in a sandbox

1. Install the CLI with `npm install -g @northflank/cli`, run `nf login`, and install the SDK with `pip install northflank`.
2. Check that `NF_API_TOKEN` and your provider key (for example `ANTHROPIC_API_KEY`) are set, confirm the team and the `opencode-sandbox` project, and create the project with `northflank create project` if it doesn't exist.
3. Store the provider key in a secret group restricted to the sandbox, then create a volume mounted at `/workspace` and a sandbox (a deployment service) running `sleep infinity`.
4. Install OpenCode onto the volume with `npm install -g --prefix /workspace/.tools @opencode/cli`, clone the repo, and run `opencode run --standalone --auto --format json -m <provider/model> "<task>"` in the background, polling for an exit-code file.
5. Write a `git diff` to `/workspace/out`, download it with `client.files.download`, then pause the sandbox (or delete it and the volume).
6. Verify that the exit code is `0`, `events.jsonl` holds JSON events and `changes.patch` contains the expected edits.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why run OpenCode in a sandbox

OpenCode's default permission policy allows nearly every action (it only asks before reading `.env` files), and `--auto` approves anything not explicitly denied. That's what makes it useful unattended, and also why it shouldn't run with your user's access to your files, SSH keys and cloud credentials. Your options, roughly:

- **Permission rules on your machine.** OpenCode's `permissions` config can `deny` actions, but commands it runs still share your kernel, filesystem and network.
- **A local Docker container.** Separates the filesystem. Containers share the host kernel, so a kernel-level escape reaches the host, and you manage images, volumes and cleanup yourself.
- **A VM you manage.** Strong isolation, more setup and teardown.
- **A managed sandbox platform.** An API creates an isolated environment per task, runs it, and tears it down.

On Northflank, each sandbox runs against its own kernel: CPU sandboxes run as microVMs and GPU sandboxes under gVisor on Northflank's cloud, with nothing to configure. The lifecycle is scriptable (create, exec, copy files, pause, resume, delete), secrets are injected at runtime, and a volume keeps the workspace across pauses. On your own (BYOC) cluster you can add [network policies](https://northflank.com/docs/v1/application/network/configure-network-policies) with egress rules to IPs, CIDRs and FQDNs.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output), **`openssl`** (to name the sandbox) and **Python 3** with the SDK (`pip install northflank`).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **A Northflank API token in `NF_API_TOKEN`**, for the SDK. **This needs a human:** create it in the Northflank UI (see the [sandbox quickstart](https://northflank.com/docs/v1/application/sandboxes/quickstart)) and export it, from your password manager, never pasted into chat.
- **A model provider key**, here `ANTHROPIC_API_KEY`. OpenCode picks up provider keys from environment variables; for other providers, use that provider's variable (for example `OPENROUTER_API_KEY`) and change the secret, the step 2 check and the model accordingly. Use a dedicated, spend-limited key: code inside the sandbox can read it.
- **A repository URL** the sandbox can clone. For a private repo, add a token (for example `GITHUB_TOKEN`) to the same secret group.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Check the keys and confirm where the sandbox will run

Before creating anything, check that the keys are set (without printing them) and print the team and whether the project exists:

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
[ -n "${NF_API_TOKEN:-}" ] && echo "NF_API_TOKEN: set" || echo "NF_API_TOKEN: missing"
[ -n "${ANTHROPIC_API_KEY:-}" ] && echo "ANTHROPIC_API_KEY: set" || echo "ANTHROPIC_API_KEY: missing"
if P=$(northflank get project --projectId opencode-sandbox -o json 2>/dev/null); then
  echo "project: opencode-sandbox exists in $(echo "$P" | jq -r '(.data // .).deployment.region') and will be reused"
else
  echo "project: opencode-sandbox will be created"
fi
```

- If a key says `missing`, stop and ask the user to export it in this shell. A human creates the Northflank API token in the UI ([new API token](https://app.northflank.com/s/account/settings/api/tokens/new)). Never print a key or paste it into chat.
- The Python SDK authenticates with `NF_API_TOKEN`, not with your `nf login` session, so the token must belong to the team printed above.
- Keep sandboxes in a project of their own, with nothing sensitive in it: code inside a sandbox is untrusted.

**If you're an agent:** show the user the team, the project and its region (`europe-west` below, unless they choose another), and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your sandboxes, and every sandbox in it runs in the project's region. The region can't be changed later. Skip this step if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "opencode-sandbox", "region": "europe-west", "description": "OpenCode sandboxes" }'
```

To use another region, change `region` (`northflank list regions` lists them); to run on your own cluster instead, pass `"clusterId"` in place of `"region"`.

## Step 4: Store the provider key as a secret

A secret group restricted to the sandbox injects the key as an environment variable at runtime, so it never goes into the image or the create payload ([manage secret groups](https://northflank.com/docs/v1/application/secure/manage-secret-groups)):

```bash
export SANDBOX_ID="opencode-$(openssl rand -hex 4)"
umask 077
jq -n --arg k "$ANTHROPIC_API_KEY" --arg sid "$SANDBOX_ID" '{
  name: ("secrets-" + $sid),
  type: "secret",
  secretType: "environment",
  priority: 10,
  restrictions: { restricted: true, nfObjects: [ {id: $sid, type: "service"} ] },
  secrets: { variables: { ANTHROPIC_API_KEY: $k } }
}' > /tmp/opencode-secrets.json
northflank put secret --projectId opencode-sandbox --file /tmp/opencode-secrets.json -o json | jq -r '(.data // .).id'
rm -f /tmp/opencode-secrets.json
```

## Step 5: Create the workspace volume and the sandbox

The rest uses the Python SDK calls from the sandbox docs. Ephemeral storage is wiped when a sandbox pauses or restarts, so attach a volume at `/workspace` for the repo, the OpenCode install and the outputs ([persistent storage example](https://northflank.com/docs/v1/application/sandboxes/examples#create-a-sandbox-with-persistent-storage)):

```python
import os
from northflank import ApiClient

client = ApiClient(api_token=os.environ["NF_API_TOKEN"])
project_id = "opencode-sandbox"  # the project from step 3
sandbox_id = os.environ["SANDBOX_ID"]
volume_name = f"data-{sandbox_id}"

volume = client.create.volume(
    project_id=project_id,
    data={
        "name": volume_name,
        "mounts": [{"containerMountPath": "/workspace"}],
        "spec": {
            "accessMode": "ReadWriteMany",
            "storageClassName": "nf-multi-rw",
            "storageSize": 10240,  # 10 GiB
        },
    },
)
volume_id = (volume.data or {}).get("id") or volume_name

client.create.service.deployment(
    project_id=project_id,
    data={
        "name": sandbox_id,
        "billing": {"deploymentPlan": "nf-compute-200"},  # 2 vCPU, 4 GB RAM
        "deployment": {
            "instances": 1,
            "docker": {
                "configType": "customEntrypointCustomCommand",
                "customEntrypoint": "/bin/bash",
                "customCommand": "-c 'sleep infinity'",
            },
            "external": {"imagePath": "node:22-bookworm"},
            "storage": {"ephemeralStorage": {"storageSize": 2048}},
        },
        "createOptions": {"volumesToAttach": [volume_id]},
    },
)

client.scale.service(project_id=project_id, service_id=sandbox_id, data={"instances": 1})
client.helpers.wait_for_service_ready(
    project_id=project_id, service_id=sandbox_id, timeout_s=300, poll_interval_s=1
)
```

`node:22-bookworm` ships Node.js, `git`, `bash` and `tar` (needed for file copies). `COMPLETED`, which `wait_for_service_ready` waits for, means the container is running.

## Step 6: Install OpenCode and clone the repository

OpenCode's docs give `npm install -g @opencode/cli` (or `curl -fsSL https://opencode.ai/v2/install | bash`). Installing with an npm prefix on the volume means it survives a pause:

```python
def sh(cmd: str):
    r = client.exec.run_service_command(
        project_id=project_id, service_id=sandbox_id,
        command=cmd, shell="/bin/bash -c",
    )
    if not r.ok:
        raise RuntimeError(f"exit {r.exit_code}: {r.stderr[-2000:]}")
    return r.stdout

sh("npm install -g --prefix /workspace/.tools @opencode/cli")
sh("/workspace/.tools/bin/opencode --version")
sh("git clone --depth 50 https://github.com/<OWNER>/<REPO>.git /workspace/repo")
sh("mkdir -p /workspace/out")
```

For a private repo, clone with `https://x-access-token:${GITHUB_TOKEN}@github.com/...` and the token in the secret group, so it's expanded inside the sandbox and never appears in your script. Check which models your key unlocks with `opencode models --standalone` and pick a `provider/model` ID, for example `anthropic/claude-sonnet-5-5`.

## Step 7: Run OpenCode headless

`opencode run` submits a prompt without opening the interactive interface and is meant for scripts and CI. Flags used here, checked against `opencode run --help` (v2.3.2):

- `--standalone` runs with a private server instead of the shared background service OpenCode otherwise discovers or starts.
- `--auto` auto-approves permissions that are not explicitly denied. Without it, an action that resolves to `ask` waits for a client decision that never comes.
- `--format json` prints raw JSON events instead of formatted output.
- `-m, --model` takes `provider/model` (optionally `#variant`); `--agent`, `--title` and `-f, --file` are also available.

There's no directory flag on `run` in v2, so `cd` into the repo. Agent runs take minutes, so start OpenCode in the background and record its exit code on the volume:

```python
import base64

task = "Fix the failing unit tests and explain the root cause."
# Write the prompt to a file so quotes in it can't break the shell command.
b64 = base64.b64encode(task.encode()).decode()
sh(f"echo {b64} | base64 -d > /workspace/out/task.md")

run = """
cd /workspace/repo
export PATH=/workspace/.tools/bin:$PATH
rm -f /workspace/out/exit_code
setsid nohup bash -c '
  opencode run --standalone --auto --format json \
    -m anthropic/claude-sonnet-5-5 --title sandbox-run \
    "$(cat /workspace/out/task.md)" \
    > /workspace/out/events.jsonl 2> /workspace/out/opencode.log
  echo $? > /workspace/out/exit_code
' >/dev/null 2>&1 &
"""
sh(run)
```

To keep some actions off-limits even with `--auto`, add `deny` rules to an `opencode.json` in the repo. OpenCode v2 uses a `permissions` list of `action`/`resource`/`effect` rules, and `deny` rules stay enforced for non-interactive clients ([permissions](https://opencode.ai/docs/permissions/)).

## Step 8: Wait for the run to finish

```python
import time

for _ in range(360):  # up to 60 minutes
    code = sh("cat /workspace/out/exit_code 2>/dev/null || true").strip()
    if code:
        print("opencode exit code:", code)
        break
    time.sleep(10)
else:
    raise TimeoutError("OpenCode did not finish; check /workspace/out/opencode.log")

sh("cd /workspace/repo && git add -A && git diff --cached > /workspace/out/changes.patch")
```

## Step 9: Copy the results back

```python
result = client.files.download(
    project_id=project_id, service_id=sandbox_id,
    remote_path="/workspace/out", local_path="./opencode-out",
)
print(result.target)
```

The download streams over the exec connection, with a `timeout` that defaults to 300 seconds in Python. Review `opencode-out/changes.patch` and apply it with `git apply`. The CLI equivalent is `northflank download service file --localPath opencode-out --remotePath /workspace/out` ([copy files](https://northflank.com/docs/v1/api/copy-files)).

## Step 10: Pause or delete the sandbox

Pausing stops compute billing, ends running processes and wipes ephemeral storage. The volume survives and is billed while it exists:

```python
client.pause.service(project_id=project_id, service_id=sandbox_id)
client.helpers.wait_for_service_paused(project_id=project_id, service_id=sandbox_id)

# Later: resume and run the next task against the same workspace.
client.resume.service(project_id=project_id, service_id=sandbox_id, data={})
client.helpers.wait_for_service_ready(project_id=project_id, service_id=sandbox_id)
```

When you're done for good, delete it. **This is irreversible:** deleting the volume destroys the workspace. Detach the volume, delete the service, then delete the volume (retry on `409` while the detach settles):

```python
client.detach.volume(project_id=project_id, volume_id=volume_id,
                     data={"nfObject": {"id": sandbox_id, "type": "service"}})
client.delete.service(project_id=project_id, service_id=sandbox_id)
client.delete.volume(project_id=project_id, volume_id=volume_id)
```

Also delete the `secrets-<SANDBOX_ID>` secret group if it was only for this sandbox.

## How to know it worked

1. `wait_for_service_ready` returns, and `opencode --version` prints a version.
2. `opencode models --standalone` lists models for your provider.
3. `/workspace/out/exit_code` contains `0`, and `events.jsonl` holds JSON events.
4. `opencode-out/changes.patch` contains the expected edits.
5. After pausing and resuming, `/workspace/repo` and `/workspace/.tools/bin/opencode` are still there.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No models for your provider | Key not injected | Check the group's `restrictions` list the sandbox ID; restart the sandbox |
| Run hangs, `exit_code` never appears | An action resolved to `ask` | Add `--auto`, or allow the action in `opencode.json` |
| Errors about the background service | Shared service unavailable in the sandbox | Pass `--standalone` |
| `exit_code` missing, log empty | Background process ended with the exec call | Read the sandbox logs; rerun and poll sooner |
| File download fails | No `tar` in the image, or wrong path | Use an image with `tar`; check `remote_path` |
| Workspace empty after resume | Files on ephemeral storage | Keep everything under `/workspace` |
| An SDK call returns 401 or 403 | `NF_API_TOKEN` missing, expired, or from another team | Ask a human for a team API token for the team from step 2, export it and retry |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

- Confirm the target from step 2 with the user before any write: team, project, region, and whether the project is new or reused.
- If `wait_for_service_ready` raises, read the sandbox logs (`northflank get service logs --projectId opencode-sandbox --serviceId "$SANDBOX_ID"`) before retrying.
- Never print provider keys, `NF_API_TOKEN` or Git tokens, and keep them out of the prompt and `changes.patch`.
- Treat the repo inside the sandbox as able to read the key. Use a dedicated key for untrusted repos.
- Ask the user before deleting the volume. Prefer pause.
- Don't push branches or open PRs from the sandbox unless the user asked.

## FAQ

### Where should I run OpenCode if not on my laptop?
For unattended runs, untrusted repositories or several tasks in parallel, run it in an isolated remote environment with its own kernel, such as a microVM sandbox, so a bad command or malicious dependency can't reach your machine. For interactive work on your own code, local permission rules may be enough.

### Can I just run OpenCode in Docker?
Yes, the same commands work in any container with Node.js. A container shares the host kernel, though, and you handle volumes, secrets and cleanup yourself. A sandbox gives you a separate kernel and an API for that lifecycle.

### Does OpenCode have its own sandbox mode?
OpenCode's docs describe a permission policy (`allow`, `ask`, `deny` per action and resource) rather than an OS-level sandbox. Use `deny` rules for actions it should never take, and the outer sandbox for isolation.

### Can I run OpenCode as a server in the sandbox?
`opencode serve --hostname 0.0.0.0 --port 4096` starts the API and web server, and clients connect with `--server <url>`. Expose the port with `public: false` so only services in the same project reach it (see [expose a web server](https://northflank.com/docs/v1/application/sandboxes/examples#expose-a-web-server)). How to secure a public port for the v2 server hasn't been confirmed here, so don't make it public.

### I'm on OpenCode 1.x (`opencode-ai`). What changes?
The 1.x `opencode run` takes similar flags (`--auto`, `--format json`, `-m`) and also has `--dir` and `--attach`. Check `opencode run --help` for your version.

## Conclusion

OpenCode now runs headless in a microVM sandbox with its own kernel: the provider key comes from a restricted secret group, the workspace and tools live on a volume that survives pauses, and the result comes back as a patch you review first.

**Next:** run another agent the same way in [How to sandbox Codex](sandbox-codex.md), or read the [sandbox examples](https://northflank.com/docs/v1/application/sandboxes/examples) for GPUs and exposed ports. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to run Claude Code in a sandbox](run-claude-code-in-a-sandbox.md)
- [How to sandbox Codex](sandbox-codex.md)
- [How to create a code execution sandbox for AI agents](create-code-execution-sandbox-for-ai-agents.md)
- [How to run self-hosted AI agent sandboxes in your own cloud](self-hosted-ai-agent-sandboxes.md)
- [How to deploy AI agents in production](deploy-ai-agents-in-production.md)
