<!--
agent-guide-meta:
  id: NF-SBX-MCP
  description: "How to build a code execution MCP server in Python that runs model code in isolated per-conversation sandboxes, served over Streamable HTTP with a bearer token."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/sandboxes/quickstart
    - https://northflank.com/docs/v1/application/sandboxes/configure-and-manage-sandboxes
    - https://northflank.com/docs/v1/application/sandboxes/sandboxes-on-northflank-cloud
    - https://northflank.com/docs/v1/application/sandboxes/examples
    - https://northflank.com/docs/v1/application/use-cases/run-code-in-sandboxes
    - https://northflank.com/docs/v1/application/secure/use-role-based-access-control
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/api/execute-command
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://pypi.org/project/mcp/ (2.3.0, `mcp.server.MCPServer`, `mcp.server.mcpserver.exceptions.ToolError`, `streamable_http_app`, `custom_route`; supports 2026-07-28 and 2025-11-25 and earlier)
    - https://modelcontextprotocol.io/specification/2026-07-28/changelog
    - https://pypi.org/project/northflank/ (2.1.0, `exec.run_service_command`, `list.services`)
    - https://github.com/modelcontextprotocol/python-sdk
    - https://code.claude.com/docs/en/mcp
  related: [NF-AI-003, NF-SBX-CODEEXEC, NF-SBX-CLAUDECODE, NF-SBX-BYOC, NF-JOB-004]
-->

# How to build a code execution MCP server

To build a code execution MCP server, expose a `run_code` tool and sandbox lifecycle tools over Streamable HTTP behind a bearer token, run each conversation's code in an isolated sandbox with its own kernel, pass state between calls through a server-minted `sandbox_id` handle, enforce a per-call timeout and delete idle sandboxes. Running model-written code next to the server's own API tokens is the usual mistake.

This guide writes the server in Python with the official MCP SDK and deploys it on Northflank with the CLI. You end with an HTTPS `/mcp` endpoint offering `create_sandbox`, `run_code`, `delete_sandbox` and `run_code_once`, verified with `curl` tool calls and connected to Claude Code. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end. A human only has to log in once, share the repository with Northflank and create the scoped API token.

## TL;DR: how to build a code execution MCP server

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, repository, branch and the two projects (`code-mcp` for the server, an empty `code-sandboxes` for sandboxes), then give Northflank access to the repository (once).
3. Write `server.py` with `MCPServer`: `create_sandbox`, `run_code`, `delete_sandbox` and `run_code_once` tools that create sandboxes with `client.create.service.deployment(...)`, run code with `client.exec.run_service_command(...)` behind a server-minted `sandbox_id`, and delete idle ones. Push.
4. **A human** creates an API role limited to `code-sandboxes`; its token and a generated `MCP_AUTH_TOKEN` go into a secret group. Deploy with `northflank put service combined`.
5. Check that the live build is your pushed commit, then call the tools with `curl`, run code twice with the same `sandbox_id`, and connect Claude Code with `claude mcp add --transport http sandbox "$URL/mcp" --header "Authorization: Bearer $MCP_AUTH_TOKEN"`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why a code execution MCP server needs a real sandbox

An MCP tool that calls `subprocess.run(code)` runs model-written code with the server's own permissions, network and secrets, which here include an API token. Code from a model or a prompt-injected document can read environment variables, write to disk or call internal services. What you need instead:

- **Isolation with its own kernel.** Containers share the host kernel. On Northflank cloud, CPU sandboxes run in microVMs and GPU sandboxes under gVisor, so each sandbox has its own kernel.
- **No secrets inside.** The sandbox gets only the code. The MCP server keeps both tokens.
- **A lifecycle.** A sandbox handle keeps state between calls (files, installed packages) for whoever holds it, and idle cleanup keeps costs bounded.

A local container shares your kernel; a client's built-in sandbox mode protects one laptop, not a shared server; a VM per conversation isolates well but is heavy to manage; a managed sandbox platform gives per-conversation isolation through an API. This guide uses the last.

## Prerequisites

- **Python 3.12 or later locally** (`python3 --version`). Step 5 writes the server and checks that it compiles.
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **`openssl`** (to generate the MCP bearer token).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **A human to create a least-privilege API token** in step 6, limited to the sandbox project.

Names used below: project `code-mcp` and service `code-mcp` for the server, project `code-sandboxes` for the sandboxes only, secret group `code-mcp-secrets`.

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

Before changing anything, confirm the repository and print the deployment target. Run this from the repository root:

```bash
grep -iE '^(mcp|northflank)([^a-z_-]|$)' requirements.txt 2>/dev/null
ls server.py requirements.txt pyproject.toml Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
for P in code-mcp code-sandboxes; do
  northflank get project --projectId "$P" -o json >/dev/null 2>&1 \
    && echo "project: $P exists and will be reused" \
    || echo "project: $P will be created"
done
northflank list services --projectId code-sandboxes -o json 2>/dev/null \
  | jq -r '"code-sandboxes services: \([(.data // .).services[]?.name] | join(", "))"'
[ -n "$NF_SANDBOX_TOKEN" ] && echo "NF_SANDBOX_TOKEN: set" || echo "NF_SANDBOX_TOKEN: not set yet (step 6)"
```

- This guide writes a Python server with the `mcp` and `northflank` packages. If the repository already holds another app, use a new repository (or a subfolder with its own `dockerWorkDir`) for the server. If an existing `server.py` or `Dockerfile` is listed, compare it with step 5 before replacing it.
- **`code-sandboxes` must hold nothing but sandboxes.** The server deletes every `sandbox-*` service there on startup. If the services line lists anything else, pick another, empty project name and use it everywhere `code-sandboxes` appears.
- The token check never prints the value. It's fine if it isn't set yet; a human creates it in step 6.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether each project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create the projects and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip a `create` line if step 2 said that project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch. The sandbox project stays empty; only the server creates services in it.

```bash
northflank create project -o json --input '{ "name": "code-mcp", "region": "europe-west" }'
northflank create project -o json --input '{ "name": "code-sandboxes", "region": "europe-west" }'
export PROJECT_ID=code-mcp

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Write the MCP server

`server.py`:

```python
import os, threading, time, uuid
import anyio, uvicorn
from mcp.server import MCPServer
from mcp.server.mcpserver.exceptions import ToolError
from northflank import ApiClient
from starlette.requests import Request
from starlette.responses import JSONResponse, PlainTextResponse

PROJECT = os.environ["SANDBOX_PROJECT_ID"]
IDLE_SECONDS = int(os.environ.get("IDLE_SECONDS", "600"))
RUN_TIMEOUT = int(os.environ.get("RUN_TIMEOUT", "60"))  # upper bound for one call
MAX_SANDBOXES = int(os.environ.get("MAX_SANDBOXES", "10"))
MAX_OUT = 20_000
nf = ApiClient(api_token=os.environ["NF_API_TOKEN"])
# Handles this process created: sandbox_id -> {"last", "lock"}. Nothing else is accepted.
sandboxes: dict[str, dict] = {}
registry_lock = threading.Lock()

def nf_create(sid: str) -> None:
    nf.create.service.deployment(project_id=PROJECT, data={
        "name": sid,
        "billing": {"deploymentPlan": "nf-compute-200"},
        "deployment": {
            "instances": 1,
            "docker": {"configType": "customEntrypointCustomCommand",
                       "customEntrypoint": "/bin/bash",
                       "customCommand": "-c 'sleep infinity'"},
            "external": {"imagePath": "python:3.12-slim"},
            "storage": {"ephemeralStorage": {"storageSize": 2048}},
        },
    })
    nf.helpers.wait_for_service_ready(project_id=PROJECT, service_id=sid, timeout_s=300, poll_interval_s=1)

def nf_delete(sid: str) -> None:
    try:
        nf.delete.service(project_id=PROJECT, service_id=sid)
    except Exception as e:
        print(f"delete {sid} failed: {e}", flush=True)

def lookup(sid: str) -> dict:
    with registry_lock:
        entry = sandboxes.get(sid)
    if entry is None:  # unknown, expired, or not created by this server
        raise ToolError(f"unknown sandbox_id {sid!r}; call create_sandbox first")
    return entry

def check(language: str, code: str, timeout: int) -> int:
    if language not in ("python", "bash"):
        raise ToolError("language must be 'python' or 'bash'")
    if len(code) > 100_000:
        raise ToolError("code too large; split it up")
    return max(1, min(int(timeout), RUN_TIMEOUT))

def create_sync() -> str:
    sid = f"sandbox-{uuid.uuid4().hex[:8]}"
    entry = {"last": time.time(), "lock": threading.Lock()}
    with registry_lock:
        if len(sandboxes) >= MAX_SANDBOXES:
            raise ToolError("sandbox limit reached; call delete_sandbox on one you no longer need")
        sandboxes[sid] = entry
    with entry["lock"]:  # run_code on this handle waits until the sandbox is ready
        try:
            nf_create(sid)
        except Exception:
            with registry_lock:
                sandboxes.pop(sid, None)
            nf_delete(sid)
            raise
    return sid

def run_sync(sid: str, language: str, code: str, timeout: int) -> dict:
    entry = lookup(sid)
    interp = ["python3", "-c"] if language == "python" else ["bash", "-c"]
    with entry["lock"]:  # one call at a time per sandbox
        if sandboxes.get(sid) is not entry:
            raise ToolError(f"sandbox {sid!r} was deleted")
        r = nf.exec.run_service_command(project_id=PROJECT, service_id=sid,
                                        command=["timeout", str(timeout), *interp, code])
        entry["last"] = time.time()
    return {"exit_code": r.exit_code, "timed_out": r.exit_code == 124,
            "stdout": r.stdout[-MAX_OUT:], "stderr": r.stderr[-MAX_OUT:], "sandbox_id": sid}

def delete_sync(sid: str) -> None:
    entry = lookup(sid)
    with entry["lock"]:
        with registry_lock:
            sandboxes.pop(sid, None)
    nf_delete(sid)

def reaper() -> None:
    # Startup: remove sandboxes left by a previous process (the in-memory map is gone).
    for s in (nf.list.services(project_id=PROJECT, per_page=100).data or {}).get("services", []):
        if s.get("name", "").startswith("sandbox-"):
            nf_delete(s["id"])
    while True:
        time.sleep(30)
        now = time.time()
        with registry_lock:
            idle = [k for k, e in sandboxes.items()
                    if now - e["last"] > IDLE_SECONDS and not e["lock"].locked()]
            gone = [k for k in idle if sandboxes.pop(k, None)]
        for sid in gone:
            nf_delete(sid)

mcp = MCPServer(name="code-sandbox")

@mcp.tool()
async def create_sandbox() -> dict:
    """Create an isolated sandbox and return its sandbox_id. Pass that sandbox_id to every
    run_code call that should share files and installed packages. A sandbox unused for
    idle_seconds is deleted; call delete_sandbox when you are done."""
    sid = await anyio.to_thread.run_sync(create_sync)
    return {"sandbox_id": sid, "idle_seconds": IDLE_SECONDS}

@mcp.tool()
async def run_code(sandbox_id: str, code: str, language: str = "python", timeout: int = 30) -> dict:
    """Run Python or bash code in the sandbox from create_sandbox. Returns exit_code, stdout,
    stderr. timeout is in seconds, capped at the server's RUN_TIMEOUT."""
    t = check(language, code, timeout)
    return await anyio.to_thread.run_sync(run_sync, sandbox_id, language, code, t)

@mcp.tool()
async def delete_sandbox(sandbox_id: str) -> dict:
    """Delete a sandbox from create_sandbox and everything in it."""
    await anyio.to_thread.run_sync(delete_sync, sandbox_id)
    return {"deleted": sandbox_id}

@mcp.tool()
async def run_code_once(code: str, language: str = "python", timeout: int = 30) -> dict:
    """Run code in a fresh sandbox that is deleted afterwards. Nothing persists between calls."""
    t = check(language, code, timeout)
    sid = await anyio.to_thread.run_sync(create_sync)
    try:
        return await anyio.to_thread.run_sync(run_sync, sid, language, code, t)
    finally:
        await anyio.to_thread.run_sync(delete_sync, sid)

@mcp.custom_route("/healthz", methods=["GET"])
async def healthz(request: Request):
    return PlainTextResponse("ok")

inner = mcp.streamable_http_app(host="0.0.0.0")
TOKEN = os.environ["MCP_AUTH_TOKEN"].encode()

async def app(scope, receive, send):
    if scope["type"] == "http" and scope["path"].startswith("/mcp"):
        auth = dict(scope["headers"]).get(b"authorization", b"")
        if auth != b"Bearer " + TOKEN:
            return await JSONResponse({"error": "unauthorized"}, status_code=401)(scope, receive, send)
    await inner(scope, receive, send)

if __name__ == "__main__":
    threading.Thread(target=reaper, daemon=True).start()
    uvicorn.run(app, host="0.0.0.0", port=int(os.environ.get("PORT", "3000")))
```

How it maps to the docs and SDKs:

- `create.service.deployment`, `helpers.wait_for_service_ready`, `exec.run_service_command` and `delete.service` are the calls from the sandbox [quickstart](https://northflank.com/docs/v1/application/sandboxes/quickstart) and [manage sandboxes](https://northflank.com/docs/v1/application/sandboxes/configure-and-manage-sandboxes) pages. `command` also accepts an argv list in `northflank` 2.1.0, which avoids shell-quoting the model's code.
- The SDK's exec `timeout` argument (60 s by default) only limits opening the connection, so coreutils `timeout` inside the sandbox enforces the run limit. Exit code `124` means the code was killed.
- **Why handles, not sessions.** MCP revision 2026-07-28 removed protocol-level sessions, the `Mcp-Session-Id` header and the `initialize` handshake ([changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)); every request stands alone. Cross-call state therefore goes through a server-minted handle that the model passes back as a tool argument. `mcp` 2.3.0 also still serves 2025-11-25 and earlier clients (it answers `initialize`), and the same tools work for them.
- **Handle ownership.** `run_code` and `delete_sandbox` accept only IDs in the in-memory `sandboxes` map, so a client can't point them at another service, and the API token is scoped to the sandbox project anyway. Every client that holds `MCP_AUTH_TOKEN` is trusted equally: anyone with the token and a valid `sandbox_id` can use that sandbox. Give separate users separate servers or tokens if that matters. `MAX_SANDBOXES` caps how many can exist at once.
- Anticipated failures raise `ToolError`, which `mcp` 2.3.0 returns as a tool result with `isError` and your message. Any other exception also becomes an `isError` result, but with only a generic message.
- `MCPServer`, `@mcp.tool()`, `ToolError`, `custom_route` and `streamable_http_app` come from `mcp` 2.3.0 (`pip install "mcp[cli]"` per the [SDK README](https://github.com/modelcontextprotocol/python-sdk)). Passing `host="0.0.0.0"` matters: with the default `127.0.0.1`, the SDK turns on DNS-rebinding protection that only accepts `localhost` Host headers, which would reject requests to your public hostname.
- `/healthz` stays outside the token check so health checks work.

`requirements.txt`:

```text
mcp==2.3.0
northflank==2.1.0
```

`Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY server.py .
CMD ["python", "server.py"]
```

Check that the server at least compiles, then commit and push. Northflank builds what's on the remote branch, not your local working tree:

```bash
python3 -m py_compile server.py && echo COMPILE_OK
git add server.py requirements.txt Dockerfile
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Store both tokens in a secret group

**This step needs a human:** in Team Settings, create an RBAC role [restricted to the `code-sandboxes` project](https://northflank.com/docs/v1/application/secure/use-role-based-access-control) with Services Create, Read, Update and Delete, plus the permission that allows command execution (its exact name isn't confirmed here; check the role editor), then generate an API token from that role with an expiry. The token is shown once. Export it as `NF_SANDBOX_TOKEN` in the shell that runs the commands below, and don't paste it into chat.

```bash
umask 077
: "${NF_SANDBOX_TOKEN:?ask the human to export NF_SANDBOX_TOKEN}"
MCP_AUTH_TOKEN=$(openssl rand -hex 32)
jq -n --arg t "$MCP_AUTH_TOKEN" --arg nf "$NF_SANDBOX_TOKEN" '{
  name: "code-mcp-secrets", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ {id: "code-mcp", type: "service"} ] },
  secrets: { variables: { MCP_AUTH_TOKEN: $t, NF_API_TOKEN: $nf } }
}' > /tmp/code-mcp-secrets.json
northflank put secret --projectId "$PROJECT_ID" --file /tmp/code-mcp-secrets.json -o json | jq '(.data // .).id'
rm -f /tmp/code-mcp-secrets.json
```

The group is restricted to the `code-mcp` service, so nothing else in the project inherits the sandbox token (see [secret groups](https://northflank.com/docs/v1/application/secure/manage-secret-groups)).

## Step 7: Deploy the server over Streamable HTTP

Save the service definition as `code-mcp.json`:

```json
{
  "name": "code-mcp",
  "description": "Code execution MCP server (Streamable HTTP)",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "mcp", "internalPort": 3000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "3000", "SANDBOX_PROJECT_ID": "code-sandboxes", "IDLE_SECONDS": "600", "RUN_TIMEOUT": "60" },
  "healthChecks": [
    { "type": "readinessProbe", "protocol": "HTTP", "path": "/healthz", "port": 3000,
      "initialDelaySeconds": 3, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

Use the same repository URL, provider and branch as in step 4. `SANDBOX_PROJECT_ID` is the empty project from step 4.

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file code-mcp.json | jq '(.data // .).id'
```

The first build starts automatically and takes a few minutes. This loop checks every 10 seconds and prints the server's URL when it's live:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId code-mcp --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId code-mcp --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId code-mcp --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId code-mcp`. Otherwise save the URL for the next steps:

```bash
URL="https://$HOST"
echo "$URL/mcp"
```

Keep `instances` at 1: the handle map lives in memory, so a second instance would reject handles the first one created and couldn't reap its sandboxes.

## Step 8: Verify with real tool calls

These requests use the MCP 2026-07-28 shape: no handshake, a `_meta` envelope in every request, and `MCP-Protocol-Version` and `Mcp-Method` headers that match the body. `tools/call` also needs an `Mcp-Name` header that matches the tool name, or `mcp` 2.3.0 rejects the request as a header mismatch.

```bash
META='{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientInfo":{"name":"curl-check","version":"0.0.1"},"io.modelcontextprotocol/clientCapabilities":{}}'
M=(-H "Authorization: Bearer $MCP_AUTH_TOKEN" -H "Content-Type: application/json" \
   -H "Accept: application/json, text/event-stream" -H "MCP-Protocol-Version: 2026-07-28")

curl -sS -X POST "$URL/mcp" "${M[@]}" -H "Mcp-Method: tools/list" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"tools/list\",\"params\":{\"_meta\":$META}}" \
  | sed 's/^data: //' | grep -o '"name":"[^"]*"'

call() {  # call <id> <tool> <arguments-json>
  jq -nc --argjson id "$1" --arg name "$2" --argjson args "$3" --argjson meta "$META" \
    '{jsonrpc: "2.0", id: $id, method: "tools/call", params: {name: $name, arguments: $args, _meta: $meta}}' \
  | curl -sS -X POST "$URL/mcp" "${M[@]}" -H "Mcp-Method: tools/call" -H "Mcp-Name: $2" --data-binary @- \
  | sed 's/^data: //'
}

SBX=$(call 2 create_sandbox '{}' | grep -o 'sandbox-[0-9a-f]\{8\}' | head -1); echo "sandbox_id=$SBX"
call 3 run_code "$(jq -nc --arg s "$SBX" '{sandbox_id: $s, code: "import platform; print(platform.python_version()); open(\"/tmp/x\",\"w\").write(\"hi\")"}')"
call 4 run_code "$(jq -nc --arg s "$SBX" '{sandbox_id: $s, code: "cat /tmp/x", language: "bash"}')"
call 5 run_code '{"sandbox_id":"sandbox-00000000","code":"print(1)"}'
```

`tools/list` should print `create_sandbox`, `run_code`, `delete_sandbox` and `run_code_once`. `create_sandbox` takes as long as the sandbox does to boot. Expect `"exit_code": 0` with a Python version in `stdout`, then `hi` from the second `run_code`, both with the same `sandbox_id`. The last call should come back with `"isError": true` and `unknown sandbox_id`, because the server didn't create that handle. Confirm the sandbox exists, delete it, and check that unauthenticated requests get `401`:

```bash
northflank list services --projectId code-sandboxes -o json | jq -r '(.data // .).services[].name'
call 6 delete_sandbox "$(jq -nc --arg s "$SBX" '{sandbox_id: $s}')"
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$URL/mcp" -H "Content-Type: application/json" -d '{}'
```

To test with an older client, use the legacy flow: POST `{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"curl-check","version":"0.0.1"}}}` with the same `Authorization`, `Content-Type` and `Accept` headers, keep the `Mcp-Session-Id` response header if there is one, then POST `{"jsonrpc":"2.0","method":"notifications/initialized"}` (expect `202`). After that the tool calls are the same, with `MCP-Protocol-Version` set to the `protocolVersion` the server returned, the `Mcp-Session-Id` header if you got one, and without `_meta` and the `Mcp-Method`/`Mcp-Name` headers.

## Step 9: Connect Claude Code and other clients

```bash
claude mcp add --transport http sandbox "$URL/mcp" --header "Authorization: Bearer $MCP_AUTH_TOKEN"
claude mcp get sandbox
```

`claude mcp add` only writes config; `claude mcp get` or `/mcp` inside Claude Code shows whether it connects ([Claude Code MCP docs](https://code.claude.com/docs/en/mcp)). Add `--scope user` to use it in every project. Cursor, Codex and custom agents need the same URL and `Authorization` header in their MCP configuration. Watch the server while a client works:

```bash
northflank get service logs --projectId "$PROJECT_ID" --serviceId code-mcp --types runtime --duration 600 --lineLimit 200 -o json
```

## How to know it worked

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId code-mcp --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. `GET $URL/healthz` returns `200`.
3. `tools/list` includes `create_sandbox`, `run_code`, `delete_sandbox` and `run_code_once`; a `run_code` call returns `exit_code`, `stdout` and `stderr` (step 8).
4. Two `run_code` calls with the same `sandbox_id` see each other's files, and a handle the server didn't create is rejected.
5. After `IDLE_SECONDS` without a call on its handle, the sandbox disappears from `northflank list services --projectId code-sandboxes`.
6. Requests to `/mcp` without the token return `401`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `421 Invalid Host header` on every MCP request | DNS-rebinding protection | Keep `host="0.0.0.0"` in `streamable_http_app`, or set `transport_security` with your hostname |
| Tool error mentioning `403` | Token role lacks a permission or project | **Human:** fix the role; tokens pick up role changes immediately |
| `create_sandbox` or `run_code_once` slow or timing out in the client | Sandbox creation plus `wait_for_service_ready` | Reuse one `sandbox_id` per conversation; raise the client's tool timeout |
| `exit_code` 124 | Code ran past `RUN_TIMEOUT` | Raise `RUN_TIMEOUT` or break the work up |
| `unknown sandbox_id` | Handle expired after `IDLE_SECONDS`, was deleted, the server restarted, or the model invented an ID | Call `create_sandbox` again and pass the new `sandbox_id` |
| `400` with error `-32020` (header mismatch) | `MCP-Protocol-Version`, `Mcp-Method` or `Mcp-Name` doesn't match the body | Send all three, matching `_meta` and the tool name |
| `sandbox limit reached` | `MAX_SANDBOXES` handles exist | Call `delete_sandbox`, or raise `MAX_SANDBOXES` |
| Sandboxes pile up | Server restarted while handles were live, or more than 1 instance | The startup sweep removes `sandbox-*`; keep 1 instance |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Run the step 2 check on `code-sandboxes` before you create anything there: it must hold no services but sandboxes.
- Never print `NF_SANDBOX_TOKEN`, `NF_API_TOKEN` or `MCP_AUTH_TOKEN`. Only a human creates the API role and token.
- Point `SANDBOX_PROJECT_ID` only at a project created for sandboxes: the startup sweep deletes services named `sandbox-*` there.
- Run the unauthenticated check. If `/mcp` answers without a token, stop: anyone could run code on the user's account.

## FAQ

### Where should I run a code execution MCP server?
Run the MCP server as an always-on HTTPS service, and run the code itself somewhere else: in a sandbox with its own kernel, one per conversation or per call. Running code inside the MCP server's container, or on the machine of whoever starts a local stdio server, puts that machine's secrets and network within reach of the model.

### Why not pause sandboxes instead of deleting them?
You can: `client.pause.service(...)` stops compute charges and `client.resume.service(...)` brings the sandbox back, but ephemeral files are lost on pause. Attach a volume if a sandbox's files must survive that.

### How do I limit the sandbox's network access?
On Northflank cloud, network policies aren't documented for sandboxes. On your own cluster, use network policies; see [How to run self-hosted AI agent sandboxes in your own cloud](self-hosted-ai-agent-sandboxes.md).

## Conclusion

You now have a remote MCP server whose tools hand out sandbox handles, run code in the sandbox a handle names, enforce a time limit, accept only handles the server created, keeps both tokens out of the sandbox, and deletes idle sandboxes. Clients need only the URL and a bearer token.

**Next:** give each sandbox a [persistent volume](https://northflank.com/docs/v1/application/sandboxes/examples) or run the sandboxes [in your own cloud](self-hosted-ai-agent-sandboxes.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy an MCP server](deploy-mcp-server.md)
- [How to create a code execution sandbox for AI agents](create-code-execution-sandbox-for-ai-agents.md)
- [How to run Claude Code in a sandbox](run-claude-code-in-a-sandbox.md)
- [How to run self-hosted AI agent sandboxes in your own cloud](self-hosted-ai-agent-sandboxes.md)
- [How to deploy AI agents in production](deploy-ai-agents-in-production.md)
