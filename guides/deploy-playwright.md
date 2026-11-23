<!--
agent-guide-meta:
  id: NF-STK-PLAYWRIGHT
  description: "How to deploy Playwright as a remote browser server: the official Docker image running run-server, a private WebSocket endpoint, version-matched clients."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/stacks/deploy-playwright
    - https://playwright.dev/docs/docker
    - https://playwright.dev/docs/api/class-browsertype#browser-type-connect
    - https://github.com/microsoft/playwright/blob/main/packages/playwright-core/src/cli/program.ts
    - https://github.com/microsoft/playwright/blob/main/packages/playwright-core/src/remote/playwrightServer.ts
    - https://mcr.microsoft.com/v2/playwright/tags/list
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-JOB-004, NF-SBX-CODEEXEC, NF-AI-003, NF-DEP-002]
-->

# How to deploy Playwright

To deploy Playwright as a shared remote browser, run the official `mcr.microsoft.com/playwright` Docker image with `npx playwright run-server --port 3000 --host 0.0.0.0`, give the container extra shared memory, keep the WebSocket endpoint private or behind auth, and connect with `browserType.connect()` from a client on exactly the same Playwright version. A version mismatch is the most common failure: the server refuses a client on a different major.minor version with a `Playwright version mismatch` error.

This guide does it on Northflank with the CLI: one service from the official image, a private WebSocket port that agents and test runners in the same project reach at `ws://playwright:3000/`, TCP health checks and 2 GB of `/dev/shm`. You end with a browser server that loads a page for a connected client, checked from your machine through a port-forward. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to deploy Playwright

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Pick a Playwright version with a published `mcr.microsoft.com/playwright:v<VERSION>-noble` image, confirm the team and project, and create the project if it's new.
3. Deploy it with `northflank put service deployment`: the command `npx -y playwright@<VERSION> run-server --port 3000 --host 0.0.0.0`, a private port 3000, `shmSize` 2048 and TCP probes, then wait for `COMPLETED`.
4. Connect clients in the project with `chromium.connect('ws://playwright:3000/')`, or set `PW_TEST_CONNECT_WS_ENDPOINT` for `npx playwright test`.
5. Verify that the port isn't public, then connect through `northflank forward service` with a script that prints a page title.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why a separate browser server?

Browsers are the heaviest dependency an agent or test suite can have: hundreds of megabytes of binaries, system libraries, fonts, and memory spikes per page. Bundling them into every app image makes builds slow and containers fat. Playwright's answer is the remote connection mode in its Docker docs: one container runs `run-server`, and clients anywhere call `browserType.connect()` over a WebSocket. The client code stays the same; only the launch line changes.

Running that server in production comes down to three settings from the Playwright Docker docs:

- **Same version on both sides.** The docs say to keep the Playwright version in your tests identical to the one in the container. `run-server` itself rejects a client whose major.minor version differs from the server's with a `Playwright version mismatch` error, and a `playwright` package that doesn't match the image can't locate the browser executables. Pin the image tag, the server package and the client package together.
- **Shared memory.** The docs recommend `--ipc=host` because "Chromium can run out of memory and crash" without it. The equivalent here is a larger `/dev/shm`.
- **No open endpoint.** `run-server` has no authentication. Anyone who reaches the WebSocket can drive a browser from your network. Keep it private, or put auth in front.

So where should a Playwright browser server run? Close to the agents or test runners that use it, on a private network, as an always-on container with a few GB of memory. Serverless functions are a poor fit: browsers start slowly and need more memory and `/dev/shm` than most functions allow. A VM works if you run Docker and a firewall yourself. A container platform with private networking runs the official image as is.

Northflank also has a [Playwright stack template](https://northflank.com/stacks/deploy-playwright). It deploys an example app that takes screenshots on HTTP request, which is useful if you want an HTTP API rather than a shared browser.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

Names used below: project `playwright`, service `playwright`. Private ports are only reachable from the same project, so if your agents or tests already run in a Northflank project, use that project's name instead of `playwright` in steps 2 and 3.

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

Before changing anything, choose the Playwright version (the one your clients already pin, or the latest release), check that its image is published, and print the deployment target:

```bash
PW_VERSION=$(curl -s https://registry.npmjs.org/playwright/latest | jq -r .version); echo "version: $PW_VERSION"
curl -s https://mcr.microsoft.com/v2/playwright/tags/list | jq -r '.tags[]' | grep -x "v$PW_VERSION-noble" \
  && echo "image:   mcr.microsoft.com/playwright:v$PW_VERSION-noble is published" \
  || echo "image:   v$PW_VERSION-noble is NOT published yet: pick the previous version"
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId playwright -o json >/dev/null 2>&1 \
  && echo "project: playwright exists and will be reused" \
  || echo "project: playwright will be created"
```

To use the version your clients pin, set `PW_VERSION` to it and re-run the `curl … | grep` line.

**If you're an agent:** show the user the team, the Playwright version, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services, and the clients that use the browser server must run in the same one. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "playwright", "region": "europe-west" }'
export PROJECT_ID=playwright
```

## Step 4: Deploy the browser server

Write `playwright.json`, replacing both `<PW_VERSION>` values with the same version:

```json
{
  "name": "playwright",
  "billing": { "deploymentPlan": "nf-compute-200-4" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "mcr.microsoft.com/playwright:v<PW_VERSION>-noble" },
    "docker": { "configType": "customCommand",
                "customCommand": "npx -y playwright@<PW_VERSION> run-server --port 3000 --host 0.0.0.0 --max-clients 4" },
    "storage": { "shmSize": 2048 }
  },
  "ports": [ { "name": "ws", "internalPort": 3000, "public": false, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "TCP", "type": "readinessProbe", "port": 3000,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 6, "successThreshold": 1 },
    { "protocol": "TCP", "type": "livenessProbe", "port": 3000,
      "initialDelaySeconds": 60, "periodSeconds": 20, "timeoutSeconds": 2, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file playwright.json
```

What the settings do:

- **The command** is the one from Playwright's Docker docs. `npx -y` fetches the pinned `playwright` package at start, which needs outbound internet, and uses the browsers already baked into the image. `--max-clients 4` caps concurrent connections, so one busy test run can't exhaust memory. Raise it together with the plan.
- **`shmSize: 2048`** gives `/dev/shm` 2 GB (the value is in MiB). This replaces `--ipc=host`.
- **`public: false`** keeps the endpoint on the project's private network. `run-server` has no auth of its own.
- **TCP probes** check that the port accepts connections without opening a browser session.
- **`nf-compute-200-4`** (2 vCPU, 4 GB) is a starting point for a few parallel Chromium contexts. Each extra concurrent page needs more memory.

## Step 5: Wait for the rollout

```bash
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId playwright --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
northflank get service logs --projectId "$PROJECT_ID" --serviceId playwright --lineLimit 20
```

`COMPLETED` means rolled out and serving. The log shows the server listening on port 3000. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read more of the log: `northflank get service logs --projectId "$PROJECT_ID" --serviceId playwright`.

## Step 6: Point your clients at it

Inside the project, the endpoint is `ws://playwright:3000/`. Pin the client to `<PW_VERSION>` (`playwright` or `playwright-core` in Node, `playwright==<PW_VERSION>` in Python).

Node.js:

```js
const { chromium } = require('playwright-core');
const browser = await chromium.connect(process.env.PW_WS_ENDPOINT ?? 'ws://playwright:3000/');
const page = await browser.newPage();
```

Python:

```python
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    browser = p.chromium.connect("ws://playwright:3000/")
    page = browser.new_page()
```

Playwright Test, per the Docker docs, needs no code change:

```bash
PW_TEST_CONNECT_WS_ENDPOINT=ws://playwright:3000/ npx playwright test
```

Firefox and WebKit work the same way (`firefox.connect`, `webkit.connect`). The image ships all three browsers. Close each browser when you're done, or the connection counts against `--max-clients` until it drops.

## How to know it worked

You're done when all of these hold:

1. The port isn't public:
   ```bash
   northflank get service --projectId "$PROJECT_ID" --serviceId playwright --noDefaults -o json \
     | jq '[(.data // .).ports[]? | select(.public == true)] | length'   # 0
   ```
2. A client can connect and load a page. Forward the port, read the local address from the log instead of assuming one, then run a script pinned to the same version:
   ```bash
   northflank forward service --projectId "$PROJECT_ID" --serviceId playwright --skipHostnames > /tmp/nf-pw.log 2>&1 &
   FWD_PID=$!; sleep 10; cat /tmp/nf-pw.log

   mkdir -p /tmp/pw-check && cd /tmp/pw-check && npm init -y >/dev/null && npm i "playwright-core@$PW_VERSION" >/dev/null
   PW_WS_ENDPOINT="ws://<LOCAL_ADDR>:<LOCAL_PORT>/" node -e '
     const { chromium } = require("playwright-core");
     (async () => {
       const b = await chromium.connect(process.env.PW_WS_ENDPOINT);
       const p = await b.newPage(); await p.goto("https://example.com");
       console.log(await p.title()); await b.close();
     })().catch(e => { console.error(e.message); process.exit(1); });'
   kill $FWD_PID
   ```
   This prints `Example Domain`.

## Optional: expose it publicly with basic auth

For clients outside the project (CI on another platform, a local agent), make the port public with Northflank basic auth. The patch replaces the `ports` list:

```bash
PW_BASIC_PW=$(openssl rand -hex 24)
northflank patch service deployment --projectId "$PROJECT_ID" --serviceId playwright --noDefaults -o json --input "$(jq -n --arg pw "$PW_BASIC_PW" '{
  ports: [ { name: "ws", internalPort: 3000, public: true, protocol: "HTTP",
             security: { credentials: [ { username: "browser", password: $pw, type: "basic-auth" } ] } } ]
}')"
HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId playwright --noDefaults -o json \
  | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
# Older API responses only list the host on the deprecated ports endpoint
[ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId playwright --noDefaults -o json \
  | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
echo "https://$HOST"
```

Clients connect to `wss://$HOST/` and send the credentials with `connect`'s `headers` option, for example `{ headers: { Authorization: 'Basic ' + Buffer.from('browser:' + pw).toString('base64') } }`. Before relying on it, check that a connect **without** the header fails. If it succeeds, remove the public port again. Restricting the port to known IPs with an IP policy is a further option ([port security policies](https://northflank.com/docs/v1/application/network/add-security-policies-for-ports)).

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Client error `Playwright version mismatch`, or server logs about missing browser executables | Client, server package and image tag on different Playwright versions | Pin all three to `<PW_VERSION>`; redeploy with the matching image tag |
| Rollout fails, logs show `npm` network errors | No outbound access to the npm registry at start | Allow egress, or build an image with `playwright` installed and run it without `npx -y` |
| Chromium tabs crash on heavy pages | `/dev/shm` or memory too small | Raise `shmSize` and the plan |
| `ws://playwright:3000/` doesn't resolve | Client runs in another project | Run the client in the same project, or use the optional public setup |
| New connections hang under load | `--max-clients` reached or browsers not closed | Close browsers in a `finally` block; raise the limit with the plan |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, including which project the clients run in.
- If the step 5 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Don't make the port public without auth, and never as a shortcut to fix a connection problem. An open `run-server` lets anyone browse from the user's network.
- The image runs browsers as `root`, which disables Chromium's sandbox, and the Playwright docs advise against visiting untrusted websites with it. For untrusted pages or code, use an isolated sandbox instead: see [how to create a code execution sandbox for AI agents](create-code-execution-sandbox-for-ai-agents.md).
- Never print `PW_BASIC_PW`.
- Bump the image tag and every client's Playwright version in the same change.

## FAQ

### Where should I deploy a Playwright browser server?
In the same private network as the agents or test runners that use it, as an always-on container with a few GB of memory and extra `/dev/shm`. A container platform with private service networking is the simplest fit. Serverless functions usually lack the memory, shared memory and startup time a browser needs.

### Can I run the browser as a non-root user?
The Playwright docs run as `pwuser` with a custom seccomp profile so Chromium's sandbox works. A service spec here can't pass a seccomp profile, so this guide runs the image's default user and keeps the browser away from untrusted sites.

### Should I use `connect` or `connectOverCDP`?
Use `connect` with `run-server`. It speaks Playwright's own protocol and works for Chromium, Firefox and WebKit. `connectOverCDP` attaches to a Chromium started with remote debugging, which is a different setup with lower fidelity.

### Can several agents share one server?
Yes. Each `connect` gets its own browser, up to `--max-clients`. For more throughput, raise the plan or run more instances behind the same private port; each connection stays on one instance.

## Conclusion

You now have a Playwright browser server on the official image, reachable privately at `ws://playwright:3000/`, with enough shared memory for Chromium and a client check that loads a real page.

**Next:** point your agent's browser tool or your CI's `PW_TEST_CONNECT_WS_ENDPOINT` at it, and pin the version in one place. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy AI agents in production](deploy-ai-agents-in-production.md)
- [How to create a code execution sandbox for AI agents](create-code-execution-sandbox-for-ai-agents.md)
- [How to deploy an MCP server](deploy-mcp-server.md)
- [How to deploy a Docker image](deploy-docker-image.md)
