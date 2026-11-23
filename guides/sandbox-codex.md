<!--
agent-guide-meta:
  id: NF-SBX-CODEX
  description: "How to sandbox Codex: run codex exec headless in an isolated cloud microVM with a workspace volume and a secret API key, then copy the patch back for review."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/sandboxes/quickstart
    - https://northflank.com/docs/v1/application/sandboxes/examples
    - https://northflank.com/docs/v1/application/sandboxes/configure-and-manage-sandboxes
    - https://northflank.com/docs/v1/application/sandboxes/sandboxes-on-northflank-cloud
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/application/network/configure-network-policies
    - https://developers.openai.com/codex/noninteractive
    - https://developers.openai.com/codex/agent-approvals-security
    - https://developers.openai.com/codex/auth
    - https://github.com/openai/codex
  related: [NF-SBX-OPENCODE, NF-JOB-004, NF-AI-003]
-->

# How to sandbox Codex

To sandbox Codex, run `codex exec` non-interactively inside an isolated cloud environment with its own kernel and a workspace volume, pass the OpenAI key as a secret mapped to `CODEX_API_KEY` for that run only, and copy the result back as a `git diff`. This is a cloud sandbox, not Codex's built-in sandbox mode, which only limits what Codex can do on your machine.

This guide does it on Northflank with the Python SDK and CLI: a microVM sandbox from `node:22-bookworm`, the key in a restricted secret group, Codex installed onto a `/workspace` volume, and a background run you poll, then pause or delete. You end with Codex's final message and a patch to review before applying. Every step is a script a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human logs in and creates the API token once.

## TL;DR: how to sandbox Codex

1. Install the CLI with `npm install -g @northflank/cli`, run `nf login`, and install the SDK with `pip install northflank`.
2. Check that `NF_API_TOKEN` and `OPENAI_API_KEY` are set, confirm the team and the `codex-sandbox` project, and create the project with `northflank create project` if it doesn't exist.
3. Store `OPENAI_API_KEY` in a secret group restricted to the sandbox, then create a volume mounted at `/workspace` and a sandbox (a deployment service) running `sleep infinity`.
4. Install Codex onto the volume with `npm install -g --prefix /workspace/.tools @openai/codex`, clone the repo, and run `CODEX_API_KEY="$OPENAI_API_KEY" codex exec --dangerously-bypass-approvals-and-sandbox -o … - < task.md` in the background, polling for an exit-code file.
5. Write a `git diff` to `/workspace/out`, download it with `client.files.download`, then pause the sandbox (or delete it and the volume).
6. Verify that the exit code is `0`, `last-message.md` holds Codex's summary and `changes.patch` contains the expected edits.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why sandbox Codex, and where

A coding agent runs commands it wrote a few seconds ago, often with `npm install`, test runners and build scripts that pull code from the internet. If that happens on your machine, a bad command or a malicious dependency runs with your user's access. Your options, roughly:

- **Codex's built-in sandbox on your machine.** Good default for interactive use. It restricts writes and network at the OS level, but the agent still shares your kernel, filesystem and logged-in sessions.
- **A local container (Docker, Dev Containers).** Separates the filesystem. Containers share the host kernel, so a kernel-level escape reaches the host, and OpenAI's docs note Codex's own sandbox may not work inside one.
- **A VM you manage.** Strong isolation, but you build the image, start, stop and clean up yourself.
- **A managed sandbox platform.** An API creates an isolated environment per task, runs it, and tears it down.

On Northflank, each sandbox runs against its own kernel: CPU sandboxes run as microVMs and GPU sandboxes under gVisor on Northflank's cloud, with nothing to configure. You also get a lifecycle you can script (create, exec, copy files, pause, resume, delete), secrets injected at runtime rather than baked into images, and persistent volumes so the workspace survives a pause. On your own (BYOC) cluster you can add [network policies](https://northflank.com/docs/v1/application/network/configure-network-policies) with egress rules to IPs, CIDRs and FQDNs.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output), **`openssl`** (to name the sandbox) and **Python 3** with the SDK (`pip install northflank`).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **A Northflank API token in `NF_API_TOKEN`**, for the SDK. **This needs a human:** create it in the Northflank UI (see the [sandbox quickstart](https://northflank.com/docs/v1/application/sandboxes/quickstart)) and export it, from your password manager, never pasted into chat.
- **An OpenAI API key** in your local `OPENAI_API_KEY`. Use a dedicated, spend-limited key: code inside the sandbox can read it.
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
[ -n "${OPENAI_API_KEY:-}" ] && echo "OPENAI_API_KEY: set" || echo "OPENAI_API_KEY: missing"
if P=$(northflank get project --projectId codex-sandbox -o json 2>/dev/null); then
  echo "project: codex-sandbox exists in $(echo "$P" | jq -r '(.data // .).deployment.region') and will be reused"
else
  echo "project: codex-sandbox will be created"
fi
```

- If a key says `missing`, stop and ask the user to export it in this shell. A human creates the Northflank API token in the UI ([new API token](https://app.northflank.com/s/account/settings/api/tokens/new)). Never print a key or paste it into chat.
- The Python SDK authenticates with `NF_API_TOKEN`, not with your `nf login` session, so the token must belong to the team printed above.
- Keep sandboxes in a project of their own, with nothing sensitive in it: code inside a sandbox is untrusted.

**If you're an agent:** show the user the team, the project and its region (`europe-west` below, unless they choose another), and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your sandboxes, and every sandbox in it runs in the project's region. The region can't be changed later. Skip this step if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "codex-sandbox", "region": "europe-west", "description": "Codex sandboxes" }'
```

To use another region, change `region` (`northflank list regions` lists them); to run on your own cluster instead, pass `"clusterId"` in place of `"region"`.

## Step 4: Store the API key as a secret

Keep the key out of the image and out of the create payload. A secret group restricted to the sandbox injects it as an environment variable at runtime ([manage secret groups](https://northflank.com/docs/v1/application/secure/manage-secret-groups)):

```bash
export SANDBOX_ID="codex-$(openssl rand -hex 4)"
umask 077
jq -n --arg k "$OPENAI_API_KEY" --arg sid "$SANDBOX_ID" '{
  name: ("secrets-" + $sid),
  type: "secret",
  secretType: "environment",
  priority: 10,
  restrictions: { restricted: true, nfObjects: [ {id: $sid, type: "service"} ] },
  secrets: { variables: { OPENAI_API_KEY: $k } }
}' > /tmp/codex-secrets.json
northflank put secret --projectId codex-sandbox --file /tmp/codex-secrets.json -o json | jq -r '(.data // .).id'
rm -f /tmp/codex-secrets.json
```

## Step 5: Create the workspace volume and the sandbox

From here on, the code is Python and uses the calls from the sandbox docs. Ephemeral storage is wiped when a sandbox pauses or restarts, so attach a volume at `/workspace` and keep the repo, the Codex install and the outputs on it ([persistent storage example](https://northflank.com/docs/v1/application/sandboxes/examples#create-a-sandbox-with-persistent-storage)):

```python
import os
from northflank import ApiClient

client = ApiClient(api_token=os.environ["NF_API_TOKEN"])
project_id = "codex-sandbox"  # the project from step 3
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

`node:22-bookworm` ships Node.js, `git`, `bash` and `tar`, which is everything this guide needs (`tar` is required for file copies). `COMPLETED` (what `wait_for_service_ready` waits for) means the container is running, not that anything exited.

## Step 6: Install Codex and clone the repository

Run setup commands with `run_service_command`. Installing Codex with an npm prefix on the volume means it survives a pause, so a resumed sandbox doesn't reinstall:

```python
def sh(cmd: str):
    r = client.exec.run_service_command(
        project_id=project_id, service_id=sandbox_id,
        command=cmd, shell="/bin/bash -c",
    )
    if not r.ok:
        raise RuntimeError(f"exit {r.exit_code}: {r.stderr[-2000:]}")
    return r.stdout

sh("npm install -g --prefix /workspace/.tools @openai/codex")
sh("/workspace/.tools/bin/codex --version")
sh("git clone --depth 50 https://github.com/<OWNER>/<REPO>.git /workspace/repo")
sh("mkdir -p /workspace/out")
```

For a private repo, clone with `https://x-access-token:${GITHUB_TOKEN}@github.com/...` and the token in the secret group, so it is expanded inside the sandbox and never appears in your script.

## Step 7: Run Codex headless

`codex exec` runs Codex non-interactively: progress goes to stderr, the final message to stdout. Key flags, checked against `codex exec --help`:

- `-o, --output-last-message <FILE>` writes the final message to a file.
- `--json` prints events to stdout as JSON Lines.
- `-s, --sandbox` picks Codex's own policy: `read-only` (the default for `exec`), `workspace-write` or `danger-full-access`.
- `--dangerously-bypass-approvals-and-sandbox` skips approvals and Codex's own sandbox. The help text says it is "intended solely for running in environments that are externally sandboxed", which is this setup.
- `-C, --cd <DIR>` sets the working root; Codex requires a Git repository unless you pass `--skip-git-repo-check`.

OpenAI documents `CODEX_API_KEY` for a single run and recommends setting it only for the Codex invocation that needs it, so map the secret onto it inline. Passing `-` makes Codex read the prompt from stdin. Agent runs take minutes, so start Codex in the background and record its exit code on the volume:

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
  CODEX_API_KEY="$OPENAI_API_KEY" codex exec \
    --dangerously-bypass-approvals-and-sandbox --ephemeral \
    --json -o /workspace/out/last-message.md \
    - < /workspace/out/task.md > /workspace/out/events.jsonl 2> /workspace/out/codex.log
  echo $? > /workspace/out/exit_code
' >/dev/null 2>&1 &
"""
sh(run)
```

If you'd rather keep Codex's inner sandbox as a second layer, replace the bypass flag with `--sandbox workspace-write`. OpenAI notes its Linux sandbox needs namespace, setuid `bwrap` and seccomp support that containerized environments may block; this hasn't been tested in a Northflank sandbox, so fall back to the bypass flag if commands fail with sandbox errors.

## Step 8: Wait for the run to finish

```python
import time

for _ in range(360):  # up to 60 minutes
    code = sh("cat /workspace/out/exit_code 2>/dev/null || true").strip()
    if code:
        print("codex exit code:", code)
        break
    time.sleep(10)
else:
    raise TimeoutError("Codex did not finish; check /workspace/out/codex.log")

sh("cd /workspace/repo && git add -A && git diff --cached > /workspace/out/changes.patch")
print(sh("tail -n 20 /workspace/out/last-message.md"))
```

## Step 9: Copy the results back

Download the output directory. Both SDKs stream over the exec connection, and the Python call takes a `timeout` that defaults to 300 seconds:

```python
result = client.files.download(
    project_id=project_id, service_id=sandbox_id,
    remote_path="/workspace/out", local_path="./codex-out",
)
print(result.target)
```

Review `codex-out/changes.patch` and apply it locally with `git apply`, or have the sandbox push a branch if it holds a Git token. The CLI equivalent is `northflank download service file --localPath codex-out --remotePath /workspace/out` ([copy files](https://northflank.com/docs/v1/api/copy-files)).

## Step 10: Pause or delete the sandbox

Pausing stops compute billing, ends running processes and wipes ephemeral storage. The volume (repo, Codex install, outputs) survives, and it is billed while it exists:

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

1. `wait_for_service_ready` returns, and `codex --version` prints a version.
2. `/workspace/out/exit_code` contains `0`.
3. `codex-out/last-message.md` has Codex's final summary, and `codex-out/events.jsonl` has one JSON event per line.
4. `codex-out/changes.patch` contains the expected edits.
5. After pausing and resuming, `/workspace/repo` and `/workspace/.tools/bin/codex` are still there.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `codex.log` shows an auth error | Secret not injected | Check the group's `restrictions` list the sandbox ID; restart the sandbox |
| Codex refuses to run: not a Git repository | Ran outside the clone | `cd /workspace/repo`, or pass `--skip-git-repo-check` |
| Commands fail with sandbox or namespace errors | Codex's inner `bwrap` sandbox blocked | Use `--dangerously-bypass-approvals-and-sandbox` (the outer sandbox is the boundary) |
| `exit_code` never appears | Background process ended with the exec call, or a long run | Read `codex.log`; use the sandbox logs; raise the poll limit |
| File download fails | No `tar` in the image, or wrong path | Use an image with `tar`; check `remote_path` |
| Workspace empty after resume | Files were on ephemeral storage | Keep everything under `/workspace` |
| An SDK call returns 401 or 403 | `NF_API_TOKEN` missing, expired, or from another team | Ask a human for a team API token for the team from step 2, export it and retry |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

- Confirm the target from step 2 with the user before any write: team, project, region, and whether the project is new or reused.
- If `wait_for_service_ready` raises, read the sandbox logs (`northflank get service logs --projectId codex-sandbox --serviceId "$SANDBOX_ID"`) before retrying.
- Never print `OPENAI_API_KEY`, `NF_API_TOKEN` or Git tokens, and don't put them in the task prompt, commit messages or `changes.patch`.
- Treat anything inside the sandbox, including the repo, as able to read the API key. Use a dedicated key and rotate it if the repo is untrusted.
- Ask the user before deleting the volume. Prefer pause.
- Don't push branches or open PRs from the sandbox unless the user asked for that.

## FAQ

### How do I configure Codex's own sandbox mode and approval policy?
On your own machine, Codex CLI enforces a sandbox itself: Seatbelt on macOS, `bwrap` plus seccomp on Linux, and a Windows-specific sandbox on Windows. Choose a mode with `--sandbox` (`read-only`, `workspace-write`, `danger-full-access`) or `sandbox_mode` in `~/.codex/config.toml`, and when Codex asks before running commands with `--ask-for-approval` (`on-request` or `never`) or `approval_policy`. Network access is off by default; for `workspace-write`, `network_access = true` under `[sandbox_workspace_write]` turns it on. See OpenAI's [agent approvals and security](https://developers.openai.com/codex/agent-approvals-security) page; `--full-auto` and the `untrusted` approval policy are no longer supported.

### Where should I run Codex if I don't want it on my laptop?
For interactive work, Codex's built-in sandbox on your machine is usually enough. For unattended runs, untrusted repositories or many tasks in parallel, run it in an isolated remote environment with its own kernel, such as a microVM sandbox, so a mistake or malicious dependency can't reach your machine.

### How do I remove or disable the Codex sandbox?
`--sandbox danger-full-access` disables Codex's sandbox, and `--dangerously-bypass-approvals-and-sandbox` also skips approvals. Only do this inside an environment that is already isolated, as in this guide.

### Can I use a ChatGPT login instead of an API key?
For automation, OpenAI recommends API key authentication. `printenv OPENAI_API_KEY | codex login --with-api-key` also works, but it caches credentials in plaintext at `~/.codex/auth.json`, so the inline `CODEX_API_KEY` approach keeps less on disk.

### Can I run several Codex tasks at once?
Yes: create one sandbox (and volume) per task with a different `SANDBOX_ID`, so runs don't share files or processes.

## Conclusion

Codex now runs headless in a microVM sandbox with its own kernel: the key comes from a restricted secret group, the workspace and tools live on a volume that survives pauses, and the result comes back as a patch you review before it touches your repo.

**Next:** run another agent the same way in [How to run OpenCode in a sandbox](run-opencode-in-a-sandbox.md), or read the [sandbox examples](https://northflank.com/docs/v1/application/sandboxes/examples) for GPUs and exposed ports. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to run Claude Code in a sandbox](run-claude-code-in-a-sandbox.md)
- [How to run OpenCode in a sandbox](run-opencode-in-a-sandbox.md)
- [How to create a code execution sandbox for AI agents](create-code-execution-sandbox-for-ai-agents.md)
- [How to run self-hosted AI agent sandboxes in your own cloud](self-hosted-ai-agent-sandboxes.md)
- [How to deploy AI agents in production](deploy-ai-agents-in-production.md)
