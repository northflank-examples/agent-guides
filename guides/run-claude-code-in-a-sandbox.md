<!--
agent-guide-meta:
  id: NF-SBX-CLAUDECODE
  description: "How to run Claude Code in a sandbox: an isolated cloud microVM with a workspace volume and secret API key, capped headless runs, and the diff copied back."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/sandboxes/quickstart
    - https://northflank.com/docs/v1/application/sandboxes/examples
    - https://northflank.com/docs/v1/application/sandboxes/configure-and-manage-sandboxes
    - https://northflank.com/docs/v1/application/sandboxes/sandboxes-on-northflank-cloud
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/application/secure/inject-secrets
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/execute-command
    - https://northflank.com/docs/v1/api/copy-files
    - https://northflank.com/docs/v1/application/cloud-harness/quickstart
    - https://code.claude.com/docs/en/headless
    - https://code.claude.com/docs/en/cli-reference
    - https://code.claude.com/docs/en/permission-modes
    - https://code.claude.com/docs/en/sandboxing
    - https://code.claude.com/docs/en/setup
  related: [NF-JOB-004, NF-AI-003, NF-DEP-002]
-->

# How to run Claude Code in a sandbox

To run Claude Code in a sandbox, start an isolated cloud environment with its own kernel and a workspace volume, inject `ANTHROPIC_API_KEY` as a secret, install Claude Code as a non-root user, run `claude -p` with `--dangerously-skip-permissions` and turn and budget caps, then copy the result and `git diff` back. This is a cloud sandbox, not Claude Code's built-in sandbox setting, which only restricts shell commands on your machine.

This guide does it on Northflank with the Python SDK and CLI: a microVM sandbox from `node:22-bookworm`, the key in a restricted secret group, the repository on a `/workspace` volume, and a detached run you poll, then pause or delete. You end with a JSON result and a diff to review before applying. Every step is a script a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human logs in and creates the API token once.

## TL;DR: how to run Claude Code in a sandbox

1. Install the CLI with `npm install -g @northflank/cli`, run `nf login`, and install the SDK with `pip install northflank`.
2. Check that `NF_API_TOKEN` and `ANTHROPIC_API_KEY` are set, confirm the team and the `claude-sandbox` project, and create the project with `northflank create project` if it doesn't exist.
3. Store `ANTHROPIC_API_KEY` in a secret group restricted to the sandbox, then create a volume mounted at `/workspace` and a sandbox from `node:22-bookworm` with `sleep infinity` as its command.
4. Install Claude Code as the non-root `node` user, clone the repo, and run `claude -p "<task>" --output-format json --dangerously-skip-permissions` detached, polling for an exit-code file.
5. Copy `/workspace/out` (result JSON and `git diff`) back with `client.files.download`, then pause the sandbox (or delete it and its volume).
6. Verify that the exit code is `0`, `result.json` has a `result` and a `session_id`, and `changes.diff` holds the changes you expected.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why run Claude Code in a sandbox

Claude Code is most useful when it can act without asking. Its documentation says the mode that skips every prompt, `bypassPermissions` (`--dangerously-skip-permissions`), should only be used "in isolated environments like containers, VMs, or dev containers without internet access, where Claude Code cannot damage your host system", and that it "offers no protection against prompt injection or unintended actions". So the question is where that isolated environment lives.

- **Your own machine with the built-in sandbox setting.** Good for interactive work. It wraps shell commands only, and the agent still runs next to your files and credentials.
- **A local Docker container.** Better separation, but it shares your host's kernel and stops when your laptop sleeps.
- **A VM you manage.** Strong isolation, but you build images, rotate keys and clean up yourself.
- **A managed sandbox.** A disposable environment created by API, with its own kernel, that you can pause, resume and delete from a script.

On Northflank cloud, CPU sandboxes run as microVMs, each against its own kernel rather than the host's (GPU sandboxes use gVisor). A sandbox is a deployment service, so secret groups, volumes, logs and pause/resume come from the same SDK. For a managed version that also handles agent authentication and repository connection, see [Cloud Harness](https://northflank.com/docs/v1/application/cloud-harness/quickstart).

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **Python 3** with the SDK (`pip install northflank`), which the docs use for the copy-files and wait helpers.
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **A Northflank API token in `NF_API_TOKEN`**, for the SDK. **This needs a human:** create it in the Northflank UI (see the [sandbox quickstart](https://northflank.com/docs/v1/application/sandboxes/quickstart)) and export it, from your password manager, never pasted into chat.
- **An Anthropic API key** exported locally as `ANTHROPIC_API_KEY`, created in the Claude Console.
- **A repository to clone** over HTTPS (for a private one, a read-only token).

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
if P=$(northflank get project --projectId claude-sandbox -o json 2>/dev/null); then
  echo "project: claude-sandbox exists in $(echo "$P" | jq -r '(.data // .).deployment.region') and will be reused"
else
  echo "project: claude-sandbox will be created"
fi
```

- If a key says `missing`, stop and ask the user to export it in this shell. A human creates the Northflank API token in the UI ([new API token](https://app.northflank.com/s/account/settings/api/tokens/new)). Never print a key or paste it into chat.
- The Python SDK authenticates with `NF_API_TOKEN`, not with your `nf login` session, so the token must belong to the team printed above.
- Keep sandboxes in a project of their own, with nothing sensitive in it: code inside a sandbox is untrusted.

**If you're an agent:** show the user the team, the project and its region (`europe-west` below, unless they choose another), and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your sandboxes, and every sandbox in it runs in the project's region. The region can't be changed later. Skip this step if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "claude-sandbox", "region": "europe-west", "description": "Claude Code sandboxes" }'
```

To use another region, change `region` (`northflank list regions` lists them); to run on your own cluster instead, pass `"clusterId"` in place of `"region"`.

All Python snippets below share this setup:

```python
import os, shlex, time
from northflank import ApiClient

client = ApiClient(api_token=os.environ["NF_API_TOKEN"])
project_id = "claude-sandbox"  # the project from step 3
sandbox_id = os.environ.get("SANDBOX_ID", "claude-sbx-1")
volume_name = f"ws-{sandbox_id}"

def sh(cmd):
    """Run a command in the sandbox as root and fail loudly."""
    r = client.exec.run_service_command(
        project_id=project_id, service_id=sandbox_id,
        command=cmd, shell="/bin/bash -c",
    )
    if not r.ok:
        raise RuntimeError(f"exit {r.exit_code}: {r.stderr[-2000:]}")
    return r.stdout
```

## Step 4: Store the API key in a secret group

Don't pass the key in the create payload or bake it into an image. Put it in a secret group that only this sandbox can read:

```bash
export SANDBOX_ID=claude-sbx-1
umask 077
jq -n --arg k "$ANTHROPIC_API_KEY" --arg sid "$SANDBOX_ID" '{
  name: "claude-sandbox-secrets",
  type: "secret",
  secretType: "environment",
  priority: 10,
  restrictions: { restricted: true, nfObjects: [ {id: $sid, type: "service"} ] },
  secrets: { variables: { ANTHROPIC_API_KEY: $k } }
}' > /tmp/claude-secrets.json
northflank put secret --projectId claude-sandbox --file /tmp/claude-secrets.json -o json | jq '(.data // .).id'
rm -f /tmp/claude-secrets.json
```

`put` creates or updates, so re-running is safe. For a private repository, add a read-only `GIT_TOKEN` to the same `variables` map. See [manage secret groups](https://northflank.com/docs/v1/application/secure/manage-secret-groups).

## Step 5: Create the workspace volume and the sandbox

Ephemeral storage is wiped when a sandbox pauses or restarts, so the repository and results go on a volume mounted at `/workspace`. Volumes are attached at creation time through `createOptions.volumesToAttach`:

```python
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
# Read the id back rather than assuming it matches the name.
volume_id = (volume.data or {}).get("id") or volume_name

client.create.service.deployment(
    project_id=project_id,
    data={
        "name": sandbox_id,
        # 2 vCPU, 4 GB RAM; Claude Code lists 4 GB+ RAM as a requirement.
        "billing": {"deploymentPlan": "nf-compute-200"},
        "deployment": {
            "instances": 1,
            "external": {"imagePath": "node:22-bookworm"},
            "docker": {
                "configType": "customEntrypointCustomCommand",
                "customEntrypoint": "/bin/bash",
                "customCommand": "-c 'sleep infinity'",
            },
            "storage": {"ephemeralStorage": {"storageSize": 2048}},
        },
        "createOptions": {"volumesToAttach": [volume_id]},
    },
)

client.helpers.wait_for_service_ready(
    project_id=project_id, service_id=sandbox_id,
    timeout_s=300, poll_interval_s=2,
)
print("volume_id:", volume_id)
```

The `node:22-bookworm` image has Node.js 22, `git`, `curl`, `tar` (needed for file copy) and a non-root `node` user. Any public or private image works, so you can pre-bake your toolchain later.

## Step 6: Install Claude Code and clone the repository

Claude Code refuses `--dangerously-skip-permissions` when it runs as root, so install and run it as `node`. Prepare the workspace as root first:

```python
sh("mkdir -p /workspace/out && chown -R node:node /workspace")
sh("runuser -u node -- bash -c 'curl -fsSL https://claude.ai/install.sh | bash'")
print(sh("runuser -u node -- /home/node/.local/bin/claude --version"))

repo = "https://github.com/OWNER/REPO.git"
sh(f"runuser -u node -- git clone --depth 50 {shlex.quote(repo)} /workspace/repo")
```

The native installer puts the launcher at `~/.local/bin/claude`. If you'd rather use npm, `npm install -g @anthropic-ai/claude-code@latest` works on Node.js 22+. Installs live on ephemeral storage, so after a resume you rerun the install line; the clone on `/workspace` survives.

For a local-only project, upload it instead with `client.files.upload(project_id=..., service_id=..., local_path="./myproject", remote_path="/workspace")`, then `chown` it to `node`.

## Step 7: Check the key is visible without printing it

```python
print(sh("runuser -u node -- bash -c 'test -n \"$ANTHROPIC_API_KEY\" && echo key-present || echo key-missing'"))
```

If it prints `key-missing`, check the secret group's `restrictions` name this sandbox, then pause and resume it.

## Step 8: Run Claude Code non-interactively

`claude -p` runs one task and exits. `--output-format json` returns the result, session ID and an estimated cost; `--dangerously-skip-permissions` is the same as `--permission-mode bypassPermissions`. `--max-turns` and `--max-budget-usd` cap a runaway run (both work in print mode only).

Start it detached, so a long run doesn't depend on one exec connection (the SDK's `run_service_command` `timeout`, 60 seconds by default, only limits opening the connection), and write the exit code to a file at the end:

```python
task = "Run the test suite, fix any failing tests, and summarise what you changed."
run = (
    "cd /workspace/repo && export PATH=$HOME/.local/bin:$PATH && "
    f"claude -p {shlex.quote(task)} --output-format json "
    "--dangerously-skip-permissions --max-turns 40 --max-budget-usd 5 "
    "> /workspace/out/result.json 2> /workspace/out/stderr.log; "
    "echo $? > /workspace/out/exit_code"
)
sh("rm -f /workspace/out/exit_code")
sh(f"runuser -u node -- nohup bash -c {shlex.quote(run)} > /dev/null 2>&1 &")

for _ in range(360):  # up to an hour
    code = sh("cat /workspace/out/exit_code 2>/dev/null || true").strip()
    if code:
        print("claude exit code:", code)
        break
    time.sleep(10)
else:
    raise TimeoutError("Claude Code still running; keep polling or check the logs")
```

Notes on the flags:

- **Tighter alternative to bypass.** If you want an allow-list even inside the sandbox, replace `--dangerously-skip-permissions` with `--allowedTools "Bash,Read,Edit"` or `--permission-mode acceptEdits`. In a `-p` run, a call that needs approval and isn't allowed is denied, not prompted.
- **Reproducible runs.** Without `--bare`, a `-p` run loads the repository's `CLAUDE.md`, hooks in `.claude/settings.json` and servers in `.mcp.json` with no trust prompt. That's contained by the sandbox, but add `--bare` if you want the same behaviour on every repository; bare mode reads `ANTHROPIC_API_KEY` and skips that auto-discovery.

## Step 9: Copy the result and the diff back

Write the diff to the volume, including new files, then download the whole `out` directory:

```python
sh("runuser -u node -- bash -c 'cd /workspace/repo && git add -A && git diff --cached > /workspace/out/changes.diff'")

result = client.files.download(
    project_id=project_id, service_id=sandbox_id,
    remote_path="/workspace/out", local_path="./claude-out",
)
print("saved to", result.target)
```

Locally, read the summary with `jq -r '.result' claude-out/result.json`, check `jq '.total_cost_usd'`, and review `claude-out/changes.diff` before applying it with `git apply`. The download `timeout` defaults to 300 seconds; pass a larger one for big artefacts.

## Step 10: Pause or delete the sandbox

Pausing stops compute billing and ends running processes. The volume (repository and results) survives; ephemeral storage, including the Claude Code install, does not:

```python
client.pause.service(project_id=project_id, service_id=sandbox_id)
client.helpers.wait_for_service_paused(project_id=project_id, service_id=sandbox_id)

# Later, to continue on the same workspace:
# client.resume.service(project_id=project_id, service_id=sandbox_id, data={})
# client.helpers.wait_for_service_ready(project_id=project_id, service_id=sandbox_id)
# then rerun the install line from step 6.
```

A volume is billed while it exists, even when the sandbox is paused. When you're finished, and only after the user confirms and the results are copied out, detach the volume, delete the sandbox, then delete the volume. This can't be undone:

```python
client.detach.volume(
    project_id=project_id, volume_id=volume_id,
    data={"nfObject": {"id": sandbox_id, "type": "service"}},
)
client.delete.service(project_id=project_id, service_id=sandbox_id)
# Detaching settles asynchronously: retry on 409; 404 or 410 means it's gone.
client.delete.volume(project_id=project_id, volume_id=volume_id)
```

CLI equivalents: `northflank pause service --projectId claude-sandbox --serviceId "$SANDBOX_ID"`, `northflank resume service …`, and `northflank delete service …`. From a terminal, `northflank exec service --projectId claude-sandbox --serviceId "$SANDBOX_ID"` opens a shell in the sandbox; for one-off commands without a TTY, wrap it in `script -q /dev/null …` (macOS) so output isn't lost.

## How to know it worked

1. `wait_for_service_ready` returned and `claude --version` printed a version.
2. Step 7 printed `key-present`.
3. `/workspace/out/exit_code` contains `0`, and `result.json` has a `result` field and a `session_id`.
4. `claude-out/changes.diff` exists locally and contains the changes you expected.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `--dangerously-skip-permissions cannot be used with root/sudo privileges` | Claude Code ran as root | Run it with `runuser -u node --` as in step 8 |
| `key-missing` in step 7 | Secret group not applied to this service | Check `restrictions.nfObjects`, then restart the sandbox |
| `result.json` reports an authentication error | Key invalid or not exported to the `node` user | Rotate the key in the secret group; recheck step 7 |
| `claude: command not found` after resume | Install lived on ephemeral storage | Rerun the install line from step 6 |
| Copy fails before transferring | Missing path, or no `tar` in the image | Check `remote_path`; use an image with `tar` |
| Run stops early | `--max-turns` or `--max-budget-usd` reached | Raise the caps or split the task |
| An SDK call returns 401 or 403 | `NF_API_TOKEN` missing, expired, or from another team | Ask a human for a team API token for the team from step 2, export it and retry |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write: team, project, region, and whether the project is new or reused.
- If `wait_for_service_ready` raises, read the sandbox logs (`northflank get service logs --projectId claude-sandbox --serviceId claude-sbx-1`) before retrying.
- Never print `ANTHROPIC_API_KEY` or any Git token, and never write them to the repository, logs, PR descriptions or chat. Check presence only, as in step 7.
- Treat the sandbox's output as untrusted: show the user the diff and summary before applying, committing or pushing anything.
- Set `--max-turns` and `--max-budget-usd` on every run, and tell the user the reported `total_cost_usd` (an estimate).
- Pause the sandbox when a run finishes. Delete the sandbox or volume only after the user confirms that specific resource.

## FAQ

### Where should I run Claude Code when it needs full permissions?
Somewhere it can't reach anything you'd mind losing: a container, a VM, or a managed sandbox. For one-off local work, the built-in sandbox setting or a dev container is enough. For unattended, long-running or parallel runs, a cloud sandbox with its own kernel keeps the work off your machine and lets you pause or delete it from a script.

### What's the difference between Claude Code's built-in sandbox and this?
Claude Code has a built-in **sandboxed Bash tool**, off by default, that you turn on with `/sandbox` or `"sandbox": {"enabled": true}` in settings. It uses Seatbelt on macOS and bubblewrap on Linux and WSL2 (native Windows isn't supported) to limit where shell commands can write and to route network access through a domain allow-list. It covers shell commands only; Claude's file tools, MCP servers and hooks run outside it, on your machine. This guide moves the whole Claude Code process, its tools and the repository into a separate microVM, so nothing runs on your machine at all. You can use both: turn the built-in sandbox on inside the cloud sandbox too.

### Can I run Claude Code in Docker instead?
Yes, a local container works, with the trade-offs above: a shared host kernel and a run that stops with your machine. The image in this guide is a regular Docker image, so it works in both places.

### Can I run many Claude Code tasks in parallel?
Yes: create one sandbox and volume per task with a unique `sandbox_id`, and list each one in the secret group's `restrictions`.

## Conclusion

You now have Claude Code running in a disposable microVM with its own kernel, with the key in a restricted secret group, the repository on a volume that survives pauses, capped non-interactive runs, and results returned as a JSON summary and a diff you review before applying. 
**Next:** run a different agent the same way with [How to sandbox Codex](sandbox-codex.md), or move sandboxes into your own cloud account with [How to run self-hosted AI agent sandboxes in your own cloud](self-hosted-ai-agent-sandboxes.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to sandbox Codex](sandbox-codex.md)
- [How to run OpenCode in a sandbox](run-opencode-in-a-sandbox.md)
- [How to create a code execution sandbox for AI agents](create-code-execution-sandbox-for-ai-agents.md)
- [How to run self-hosted AI agent sandboxes in your own cloud](self-hosted-ai-agent-sandboxes.md)
- [How to deploy AI agents in production](deploy-ai-agents-in-production.md)
