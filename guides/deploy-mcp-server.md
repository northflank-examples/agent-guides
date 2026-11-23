<!--
agent-guide-meta:
  id: NF-AI-003
  description: "How to deploy an MCP server as a remote Streamable HTTP service in Docker, with HTTPS, a bearer token in a secret group and verified tools/list calls."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/api/project/services/create-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/update-service-health-checks
    - https://northflank.com/docs/v1/api/project/services/get-service-logs
    - https://northflank.com/docs/v1/api/project/secrets/create-project-secret
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
    - https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http
    - https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning
    - https://modelcontextprotocol.io/specification/2026-07-28/changelog
    - https://modelcontextprotocol.io/specification/2025-11-25/basic/transports
    - https://code.claude.com/docs/en/mcp
  related: [NF-JOB-004, NF-DEP-001, NF-DEP-002, NF-DB-001]
-->

# How to deploy an MCP server

To deploy an MCP server, switch it to the Streamable HTTP transport on one path such as `/mcp`, bind it to `0.0.0.0`, check an `Authorization: Bearer` token before handling any message, and run it as a long-running container behind HTTPS with an unauthenticated `/healthz` route. Servers on MCP 2025-11-25 or earlier keep sessions in memory, so they need one instance or stateless mode before scaling.

This guide does it on Northflank with the CLI: a combined service built from your `Dockerfile` with one public HTTP port and the token in a restricted secret group. You end with a remote MCP server on a public HTTPS URL, verified with `server/discover` (or `initialize`) and `tools/list` requests plus a rejected unauthenticated call. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy an MCP server

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is an MCP server and confirm the team, repository, branch and project before changing anything, then give Northflank access to the repository (once).
3. Make the server speak Streamable HTTP on one path (for example `/mcp`), listen on `0.0.0.0:<PORT>`, check `Authorization: Bearer $MCP_AUTH_TOKEN` and serve an unauthenticated `/healthz`. Push.
4. Generate a token, store it in a secret group restricted to the service, and create a combined service with a public `HTTP` port: `northflank put service combined --file mcp.json`.
5. Wait for the deploy, check that the live build is your pushed commit, then verify with `curl -X POST "$URL/mcp"`: on MCP 2026-07-28 send `server/discover` and `tools/list` with per-request `_meta` and `MCP-Protocol-Version`/`Mcp-Method` headers (plus `Mcp-Name` on `tools/call`); on 2025-11-25 and earlier send `initialize`, then `tools/list` with the returned `Mcp-Session-Id`. A request without the token must fail.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Where to host a remote MCP server

A remote MCP server needs a public HTTPS endpoint, somewhere to keep its auth token and upstream API keys, and a process that can hold Streamable HTTP responses and sessions open. Serverless functions can host stateless MCP servers but may cut off long streams or in-memory sessions between invocations; a container platform runs the server as a long-running process with TLS handled; Kubernetes or a VM work too, but you set up ingress, TLS and restarts yourself.

A remote MCP server is a small web service, but it isn't a typical request/response API. Streamable HTTP lets the server answer a single POST with a stream of events, and clients may hold sessions open across many calls. A container service suits that:

- **Persistent process**: in-memory sessions and open streams aren't cut off between invocations.
- **HTTPS by default**: public ports get a TLS-terminated `code.run` hostname, and you can add your own domain.
- **Secrets at runtime**: the auth token and any upstream API keys never go into the image.
- **One definition**: the same JSON redeploys the server for staging and production.

## Prerequisites

- **An MCP server that runs locally** with its usual start command, for example:
  ```bash
  npm ci && npm start
  ```
  (Python: `pip install -r requirements.txt` and your start command.) Step 5 lists what it needs for remote use.
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output), plus **`openssl`** to generate the token.
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

Names used below: project `mcp-server`, secret group `mcp-secrets`, service `mcp-server`.

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
jq -r '"sdk: \(.dependencies["@modelcontextprotocol/sdk"] // .dependencies["@modelcontextprotocol/server"] // "missing")"' package.json 2>/dev/null
grep -isE '^mcp([<>=~ \[]|$)|"mcp[<>=~ \["]' requirements.txt pyproject.toml
ls package-lock.json pnpm-lock.yaml yarn.lock requirements.txt pyproject.toml Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId mcp-server -o json >/dev/null 2>&1 \
  && echo "project: mcp-server exists and will be reused" \
  || echo "project: mcp-server will be created"
```

- If `sdk` says `missing` (or prints nothing, as in a Python repo) and the `grep` finds no `mcp` package, check that this is an MCP server built on an official SDK. If it isn't an MCP server, stop and use a guide for your framework.
- The lockfile tells you the package manager; use it in your `Dockerfile`.
- If a `Dockerfile` already exists, check it against step 5 before changing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "mcp-server", "region": "europe-west" }'
export PROJECT_ID=mcp-server

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Make the server ready for remote use and push it

Before deploying, check the server against this list. Change the code where it doesn't hold:

- **It uses the Streamable HTTP transport**, not stdio, on one endpoint path such as `/mcp`. Official MCP SDKs (TypeScript, Python and others) include it. It must bind to `0.0.0.0`, not `127.0.0.1`, and listen on `PORT` (3000 below).
- **It rejects requests without the right token.** Read the expected value from `MCP_AUTH_TOKEN` and compare it to the `Authorization: Bearer …` header before handling any MCP message. The MCP specification also says servers must validate the `Origin` header on incoming connections and answer `403` when it's present and invalid.
- **There's a cheap `GET /healthz`** that returns 200 without auth, separate from the MCP path.
- **There's a `Dockerfile`** that installs dependencies and starts the server.

Northflank builds what's on the remote branch, not your local working tree. Commit and push the server changes (transport, auth check, `/healthz`, `Dockerfile`), listing the files you changed:

```bash
git add Dockerfile <SERVER_FILES>
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Generate the token and store it as a secret

```bash
umask 077
MCP_AUTH_TOKEN=$(openssl rand -hex 32)
jq -n --arg t "$MCP_AUTH_TOKEN" '{
  name: "mcp-secrets",
  type: "secret",
  secretType: "environment",
  priority: 10,
  restrictions: { restricted: true, nfObjects: [ {id: "mcp-server", type: "service"} ] },
  secrets: { variables: { MCP_AUTH_TOKEN: $t } }
}' > /tmp/mcp-secrets.json
northflank put secret --projectId "$PROJECT_ID" --file /tmp/mcp-secrets.json -o json | jq '(.data // .).id'
rm -f /tmp/mcp-secrets.json
```

Keep `MCP_AUTH_TOKEN` in your current shell for the verification step and hand it to the user through their password manager, not chat. Add any upstream keys your tools need (`GITHUB_TOKEN`, database URLs) to the same group. See [manage secret groups](https://northflank.com/docs/v1/application/secure/manage-secret-groups).

## Step 7: Create the service

Save as `mcp.json`. Port names are at most 8 characters and start with a letter:

```json
{
  "name": "mcp-server",
  "description": "Remote MCP server (Streamable HTTP)",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "mcp", "internalPort": 3000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "3000", "HOST": "0.0.0.0" },
  "healthChecks": [
    { "type": "readinessProbe", "protocol": "HTTP", "path": "/healthz", "port": 3000,
      "initialDelaySeconds": 3, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 },
    { "type": "livenessProbe", "protocol": "HTTP", "path": "/healthz", "port": 3000,
      "initialDelaySeconds": 10, "periodSeconds": 30, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file mcp.json | jq '(.data // .).id'
```

Use the same repository URL, provider and branch as in step 4. The first build starts automatically.

For a prebuilt image, drop `vcsData` and `buildSettings`, add `"external": {"imagePath": "<IMAGE>:<TAG>"}` inside `deployment`, and use `northflank put service deployment` (see [how to deploy a Docker image](deploy-docker-image.md)). REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined`.

Port choices, from the ports docs:

- Only `HTTP` and `HTTP/2` ports can be public. Public HTTP ports are served on 443 with an automatic TLS certificate, and port 80 redirects to HTTPS. `HTTP` works for Streamable HTTP and SSE; `HTTP/2` is also available.
- The public hostname looks like `mcp--mcp-server--<random>.code.run`.
- Port changes don't require a restart.

**Instances and sessions.** MCP revision 2026-07-28 removed protocol-level sessions, so a server that only speaks it can scale freely. Servers on 2025-11-25 or earlier (for example `@modelcontextprotocol/sdk` 1.x, or the legacy path of the 2.x SDKs) often keep session state in memory: keep `instances` at 1, or switch the server to stateless mode before scaling. Requests from one client can otherwise land on an instance that doesn't know its session.

**Long streams.** Load balancer idle timeouts for long-lived streams aren't documented in the sources used here. Keep tool calls short, or send progress notifications or SSE keep-alive comments (`:` lines) so the stream isn't idle.

## Step 8: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds and prints your URL when the server is live:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId mcp-server --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId mcp-server --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId mcp-server --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId mcp-server` (and the runtime logs, see [view logs](https://northflank.com/docs/v1/application/observe/view-logs)). Only run the rest if it printed a URL:

```bash
URL="https://$HOST"
echo "$URL/mcp"
curl -sS -o /dev/null -w '%{http_code}\n' "$URL/healthz"
```

## Step 9: Verify with real MCP requests

Streamable HTTP clients POST each JSON-RPC message to the MCP endpoint and must accept both JSON and an event stream. What a request looks like depends on the protocol revision the server implements:

- **2026-07-28** (the current revision; TypeScript SDK v2 `@modelcontextprotocol/server`, Python `mcp` 2.x): no `initialize` handshake and no sessions. Every request carries `_meta` with `io.modelcontextprotocol/protocolVersion` and `io.modelcontextprotocol/clientCapabilities`, plus the `MCP-Protocol-Version` and `Mcp-Method` headers. `tools/call`, `resources/read` and `prompts/get` also need an `Mcp-Name` header equal to `params.name` (or `params.uri`); `server/discover` and `tools/list` don't send it.
- **2025-11-25 and earlier** (for example `@modelcontextprotocol/sdk` 1.x): an `initialize` handshake, then an optional `Mcp-Session-Id` header on later requests.

Dual-era servers (the Python 2.x SDK, for example) answer both. Try the current revision first:

```bash
META='"_meta":{"io.modelcontextprotocol/protocolVersion":"2026-07-28","io.modelcontextprotocol/clientInfo":{"name":"curl-check","version":"0.0.1"},"io.modelcontextprotocol/clientCapabilities":{}}'
M=(-H "Authorization: Bearer $MCP_AUTH_TOKEN" -H "Content-Type: application/json" \
   -H "Accept: application/json, text/event-stream" -H "MCP-Protocol-Version: 2026-07-28")

curl -sS -w '\nstatus=%{http_code}\n' -X POST "$URL/mcp" "${M[@]}" -H "Mcp-Method: server/discover" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"server/discover\",\"params\":{$META}}"
curl -sS -X POST "$URL/mcp" "${M[@]}" -H "Mcp-Method: tools/list" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":2,\"method\":\"tools/list\",\"params\":{$META}}" \
  | sed 's/^data: //' | grep -o '"name":"[^"]*"'

# Optional: call one listed tool. Mcp-Name must equal params.name.
TOOL='<TOOL_NAME>'; ARGS='<ARGUMENTS_JSON>'
curl -sS -X POST "$URL/mcp" "${M[@]}" -H "Mcp-Method: tools/call" -H "Mcp-Name: $TOOL" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":3,\"method\":\"tools/call\",\"params\":{\"name\":\"$TOOL\",\"arguments\":$ARGS,$META}}" \
  | sed 's/^data: //'
```

The body is either plain JSON or SSE lines starting with `data: `. `server/discover` should return a `"result"` with `supportedVersions` and `capabilities`, and `tools/list` should print your tool names. If the server answers `400` with error `-32022` (unsupported protocol version), retry with a version from `error.data.supported`. If it answers `400` or `404` with an empty body or a body without a recognised MCP error, it's a legacy server: use the `initialize` flow below.

**Legacy servers (2025-11-25 and earlier).** Send `initialize` and save the response headers:

```bash
curl -sS -D /tmp/mcp-h.txt -X POST "$URL/mcp" \
  -H "Authorization: Bearer $MCP_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"curl-check","version":"0.0.1"}}}' \
  | tee /tmp/mcp-init.txt
SESSION=$(grep -i '^mcp-session-id:' /tmp/mcp-h.txt | awk '{print $2}' | tr -d '\r')
echo "session=${SESSION:-none}"
```

The response should contain `"result"` with `serverInfo`, `capabilities` and the `protocolVersion` the server picked. A server may answer with a different protocol version than you asked for; use the one it returns.

Finish the handshake and list tools. Include the session header only if the server returned one:

```bash
PV=$(grep -o '"protocolVersion":"[^"]*"' /tmp/mcp-init.txt | head -1 | cut -d'"' -f4)
H=(-H "Authorization: Bearer $MCP_AUTH_TOKEN" -H "Content-Type: application/json" \
   -H "Accept: application/json, text/event-stream" -H "MCP-Protocol-Version: $PV")
[ -n "$SESSION" ] && H+=(-H "Mcp-Session-Id: $SESSION")

curl -sS -X POST "$URL/mcp" "${H[@]}" -d '{"jsonrpc":"2.0","method":"notifications/initialized"}' -o /dev/null -w 'initialized=%{http_code}\n'
curl -sS -X POST "$URL/mcp" "${H[@]}" -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
  | sed 's/^data: //' | grep -o '"name":"[^"]*"'
```

The notification should return `202`, and `tools/list` should print your tool names.

Finally, confirm auth is enforced. The bearer check runs before any MCP parsing, so this request works for either revision. It should be `401` (or `403`), never `200`:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$URL/mcp" \
  -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"x","version":"0"}}}'
```

## Step 10 (optional): Point a custom domain at the port

Clients are easier to configure with a stable name like `mcp.example.com`. **This step needs a human:** the domain must be added and verified in your Northflank team, which requires DNS records at your registrar. Then link the subdomain to the `mcp` port, either from the subdomain's settings or by adding it to the port's `domains` array (each entry must be a verified subdomain) with `northflank patch service combined --projectId "$PROJECT_ID" --serviceId mcp-server --input '{"ports": [ { "name": "mcp", "internalPort": 3000, "public": true, "protocol": "HTTP", "domains": ["mcp.example.com"] } ]}'`. Send the full `ports` array (every port, with its existing settings), since the array you send replaces the current one. (`update service ports` is deprecated in the API.) Follow [link a domain to a port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) for the full flow, then re-run step 9 against `https://mcp.example.com/mcp`.

For an extra network-level gate, ports also support IP allow-lists and header checks under `security`; see the port security docs. A header rule stores the expected value in the port configuration, so keep the bearer check in your server as the primary control.

## Step 11: Connect a client

Give the user the URL and tell them to read the token from their password manager. For example, in Claude Code:

```bash
claude mcp add --transport http my-tools "$URL/mcp" --header "Authorization: Bearer $MCP_AUTH_TOKEN"
```

Other clients (Cursor, Codex, custom agents) take the same two things in their MCP configuration: the URL and an `Authorization` header. Then watch the server's logs while the client connects:

```bash
northflank get service logs --projectId "$PROJECT_ID" --serviceId mcp-server --types runtime --tail
```

`--tail` streams until you stop it. In an agent shell without a terminal, use `--duration 600 --lineLimit 200 -o json` instead.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId mcp-server --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. `status.deployment.status` is `COMPLETED`, and `GET $URL/healthz` returns `200`.
3. `server/discover` returns a `"result"` with `supportedVersions` (or, on a legacy server, `initialize` returns `serverInfo` and a `protocolVersion`).
4. `tools/list` returns at least one tool name you expect.
5. An MCP request without the `Authorization` header returns `401` or `403`.
6. Runtime logs show the client's requests with no errors.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `502`/`503` from the URL | App bound to `127.0.0.1` or wrong port | Bind `0.0.0.0`; match `internalPort` and `PORT` |
| `406 Not Acceptable` | Missing `Accept` header | Send `Accept: application/json, text/event-stream` |
| `400` "session not found" on `tools/list` (legacy servers) | Missing `Mcp-Session-Id`, or request hit another instance | Send the header; run 1 instance or stateless mode |
| `400` with error `-32020` or `-32602` (2026-07-28 servers) | Missing or mismatched `MCP-Protocol-Version` / `Mcp-Method` headers (or `Mcp-Name` on `tools/call`), or missing `_meta` fields | Send the headers matching the body (`Mcp-Name` = tool name), and `_meta` with `protocolVersion` and `clientCapabilities` |
| `401` with the right token | Secret not inherited | Check the group's `restrictions` include `mcp-server`; `northflank restart service` |
| An MCP request returns `200` without a token | Auth middleware not on the MCP path | Fix the server before giving anyone the URL |
| Stream drops on long tool calls | Idle timeout on the load balancer | Send progress notifications; keep calls short |
| `/healthz` fails, container restarts | Health route behind auth | Serve `/healthz` without auth |
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
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print the token, write it to the repo, or paste it into chat or PR descriptions. Reference `$MCP_AUTH_TOKEN` only.
- Run the unauthenticated check in step 9. If it returns `200`, stop and tell the user: the tools are publicly callable.
- MCP tools can act on the user's systems. Ask what each tool may access before deploying it publicly.
- Don't scale above one instance unless the server is stateless or the user accepts session errors.
- Rotating the token means updating the secret group, restarting, and updating every client. Confirm with the user first.

## FAQ

### Where should I deploy my MCP server?
If it's a remote MCP server that clients reach over HTTP, deploy it where it runs as a long-running process behind HTTPS, such as a container platform, so streams and sessions aren't cut off. A serverless function can work for a fully stateless server. A local stdio MCP server doesn't need hosting at all: the client starts it on the user's machine.

### Does Northflank support the older HTTP+SSE MCP transport?
The platform just routes HTTP. Any transport your server implements over HTTP works on a public HTTP port. Streamable HTTP is the current MCP transport, and this guide verifies it.

### Can I use OAuth instead of a static token?
Yes, if your server implements the MCP authorization flow; it runs inside your app. A static bearer token is simpler for private use.

### Do I need HTTP/2?
No. A public `HTTP` port carries Streamable HTTP and SSE. `HTTP/2` is available if your server benefits from it.

### How do I see which tools clients are calling?
Log each tool call in your server and read runtime logs with `northflank get service logs --types runtime`.

## Conclusion

Your MCP server now runs as an always-on HTTPS service with a token kept in a restricted secret group, health checks on a route that doesn't need auth, and verified `server/discover` (or `initialize`) and `tools/list` calls from the public internet. Clients only need the URL and the header.

**Next:** give the tools persistent state with [a long-running agent backend](deploy-ai-agents-in-production.md), and add [a custom domain](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy AI agents in production](deploy-ai-agents-in-production.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
