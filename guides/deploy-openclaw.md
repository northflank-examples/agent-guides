<!--
agent-guide-meta:
  id: NF-STK-OPENCLAW
  description: "How to deploy OpenClaw, the self-hosted AI assistant gateway: the official image with token auth, a /data volume, a fixed public origin and a model key."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/stacks/deploy-openclaw
    - https://github.com/openclaw/openclaw
    - https://github.com/openclaw/openclaw/blob/main/Dockerfile
    - https://docs.openclaw.ai/install/docker
    - https://docs.openclaw.ai/install/fly
    - https://docs.openclaw.ai/install/northflank
    - https://docs.openclaw.ai/gateway/config-gateway
    - https://docs.openclaw.ai/help/environment
    - https://docs.openclaw.ai/providers/anthropic
    - https://docs.openclaw.ai/cli/config
    - https://docs.openclaw.ai/cli/devices
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-JOB-004, NF-STK-N8N, NF-DEP-002, NF-SBX-CLAUDECODE]
-->

# How to deploy OpenClaw

To deploy OpenClaw, the open-source personal AI assistant that connects models to chat apps, run its official `ghcr.io/openclaw/openclaw` image with the gateway bound to `0.0.0.0` on port 18789, a strong `OPENCLAW_GATEWAY_TOKEN`, a persistent volume for its state directory, and `gateway.publicOrigin` set to your HTTPS address. Without the token the gateway refuses non-loopback binds, and without the origin the Control UI rejects the browser.

This guide does it on Northflank with the CLI: a secret group for the token and a model key, the official image with health probes, a volume at `/data`, and the public origin written into OpenClaw's config. You end with the Control UI on a public HTTPS URL and a passing gateway health check. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human signs in once.

## TL;DR: how to deploy OpenClaw

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project, image version and model key, then create the `openclaw` project.
3. Store a generated `OPENCLAW_GATEWAY_TOKEN` and your model key in a secret group, and create a volume for `/data`.
4. Run `ghcr.io/openclaw/openclaw:<version>` with the command `node dist/index.js gateway --allow-unconfigured --port 18789 --bind lan`, `OPENCLAW_STATE_DIR=/data/.openclaw` and probes on `/startupz` and `/healthz`.
5. Read the public host and set `gateway.mode`, `gateway.publicOrigin` and the default model with `openclaw config set` through `northflank exec service`, then restart.
6. Verify that `/healthz` returns 200 and `openclaw gateway health` passes, then sign in to the Control UI with the token.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## What OpenClaw is, and what it needs on a server

OpenClaw (MIT-licensed, at `github.com/openclaw/openclaw`) is a self-hosted assistant that runs all the time. Its **Gateway** is a single Node.js process that multiplexes HTTP and WebSocket traffic on one port (18789 by default). It serves the browser **Control UI**, talks to model providers, and connects channels such as Telegram and Discord. Everything it knows lives in its state directory: `openclaw.json`, credential stores, sessions and the agent workspace.

On a laptop it binds to loopback and needs no auth. On a server, four things change:

- **Bind to all interfaces** (`--bind lan`) so the platform's load balancer can reach it. The config docs say non-loopback binds require gateway auth, which `OPENCLAW_GATEWAY_TOKEN` provides.
- **Persist the state directory.** OpenClaw's Fly.io guide sets `OPENCLAW_STATE_DIR` on a mounted volume. Without it, a redeploy wipes the config, credentials and history.
- **Allow the browser origin.** Non-loopback Control UI access needs the exact HTTPS origin in `gateway.publicOrigin` (or `gateway.controlUi.allowedOrigins`).
- **Give it memory.** The Fly guide says 512 MB is too small and recommends 2 GB.

Where to host OpenClaw follows from that: an always-on container (channels and schedules fire at any hour) with a volume, a stable HTTPS URL and WebSocket support. Serverless platforms stop idle processes, so they don't fit. A VPS works, and OpenClaw documents Docker setups for one, but you handle TLS, the firewall and restarts. A container platform with volumes runs the official image without that work.

If you'd rather click through it, Northflank has an [OpenClaw stack template](https://northflank.com/stacks/deploy-openclaw), which OpenClaw's own Northflank page links to. It adds a password-protected setup wizard in front of the gateway. This guide runs the official image directly so every setting is visible and scriptable.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`**, to generate the gateway token.
- **A model provider key**, such as `ANTHROPIC_API_KEY`, exported in your shell (never written to a file). OpenClaw also reads `OPENAI_API_KEY`, `GEMINI_API_KEY`, `OPENROUTER_API_KEY` and many others; swap the variable name below if you use another provider.

Names used below: project `openclaw`, secret group `openclaw-env`, volume `openclaw-data`, service `openclaw`.

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

Before changing anything, pin an OpenClaw version (image tags match GitHub release tags without the leading `v`) and print the deployment target:

```bash
OC_VERSION=$(curl -s https://api.github.com/repos/openclaw/openclaw/releases/latest | jq -r '.tag_name | ltrimstr("v")')
echo "image:   ghcr.io/openclaw/openclaw:$OC_VERSION"
test -n "$ANTHROPIC_API_KEY" && echo "key:     ANTHROPIC_API_KEY is set" || echo "key:     ANTHROPIC_API_KEY is MISSING"
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId openclaw -o json >/dev/null 2>&1 \
  && echo "project: openclaw exists and will be reused" \
  || echo "project: openclaw will be created"
```

If the image shows no version or the key is missing, fix that first: the version comes from GitHub's public API (rate-limited), and the key must be exported in this shell.

**If you're an agent:** show the user the team, the image version, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Never print the key itself. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "openclaw", "region": "europe-west" }'
export PROJECT_ID=openclaw
```

## Step 4: Create the secret group

```bash
OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)
jq -n --arg tok "$OPENCLAW_GATEWAY_TOKEN" --arg key "$ANTHROPIC_API_KEY" '{
  name: "openclaw-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  secrets: { variables: { OPENCLAW_GATEWAY_TOKEN: $tok, ANTHROPIC_API_KEY: $key } }
}' > openclaw-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file openclaw-env.json >/dev/null && rm openclaw-env.json
```

The token is the Control UI password. Keep it in this shell for the checks, and **tell the user to copy it from the secret group into their password manager.** Channel tokens such as `TELEGRAM_BOT_TOKEN` or `DISCORD_BOT_TOKEN` go in this group too. The OpenClaw docs prefer environment variables over the config file for secrets.

## Step 5: Create the state volume

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId openclaw-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "openclaw-data",
  "mounts": [ { "containerMountPath": "/data" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 6144 }
}'
```

The volume is created unattached, so the service mounts it from its first start. OpenClaw's docs say to keep one gateway per state directory, and a `ReadWriteOnce` volume enforces that.

## Step 6: Deploy the gateway

Write `openclaw-service.json` (a Northflank service definition, not OpenClaw's own `openclaw.json`). Replace `<OC_VERSION>` with the version step 2 printed:

```json
{
  "name": "openclaw",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "ghcr.io/openclaw/openclaw:<OC_VERSION>" },
    "docker": { "configType": "customCommand",
                "customCommand": "node dist/index.js gateway --allow-unconfigured --port 18789 --bind lan" }
  },
  "ports": [ { "name": "http", "internalPort": 18789, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "OPENCLAW_STATE_DIR": "/data/.openclaw", "OPENCLAW_WORKSPACE_DIR": "/data/workspace", "TZ": "UTC" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "startupProbe", "path": "/startupz", "port": 18789,
      "initialDelaySeconds": 15, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 30, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/healthz", "port": 18789,
      "initialDelaySeconds": 10, "periodSeconds": 30, "timeoutSeconds": 5, "failureThreshold": 3 }
  ],
  "createOptions": { "volumesToAttach": ["openclaw-data"] }
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file openclaw-service.json
```

What the settings do:

- **The command** replaces the image's `CMD` and keeps its `tini` entrypoint, the same as OpenClaw's Fly config. `--allow-unconfigured` lets the gateway start before `openclaw.json` exists. `--bind lan` listens on `0.0.0.0`.
- **`/startupz`** admits traffic once startup finishes, without depending on channel health. **`/healthz`** is the liveness check the image's own `HEALTHCHECK` uses. OpenClaw documents both as unauthenticated.
- **`nf-compute-100-2`** gives 1 vCPU and 2 GB, matching the Fly guide's memory recommendation.
- **One public HTTP port.** Northflank's HTTP ports carry WebSockets, which the Control UI uses.

If the volume doesn't show as attached after the first rollout, attach it explicitly (this redeploys the service):

```bash
northflank attach volume --projectId "$PROJECT_ID" --volumeId openclaw-data --noDefaults -o json \
  --input '{"nfObject": {"id": "openclaw", "type": "service"}}'
```

## Step 7: Wait for the rollout and read the host

```bash
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId openclaw --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }

if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId openclaw --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId openclaw --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId openclaw`. The logs print `gateway ready` once the listener is up.

## Step 8: Set the public origin and default model

`openclaw` is on the image's `PATH`. These writes go to `/data/.openclaw/openclaw.json` on the volume. Without a TTY, `exec` needs `script` to keep output (Linux: `script -qefc '<cmd>' /dev/null`):

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId openclaw --shell-cmd 'sh -c' \
  --cmd "\"openclaw config set gateway.mode local && openclaw config set gateway.publicOrigin https://$HOST && openclaw config set agents.defaults.model.primary anthropic/claude-opus-5\""
northflank restart service --projectId "$PROJECT_ID" --serviceId openclaw --noDefaults -o json
```

Then wait for `COMPLETED` again with the loop from step 7. `gateway.publicOrigin` also becomes the Control UI's allowed origin. Use a model ID your key can access. `openclaw models list --provider anthropic` lists them.

**This step needs a human** if they want guided setup (channels, extra providers): open an interactive shell with `northflank exec service --projectId "$PROJECT_ID" --serviceId openclaw --shell-cmd 'sh'` and run `openclaw onboard`.

## How to know it worked

You're done when all of these hold:

1. The liveness and startup endpoints return 200 over HTTPS:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/healthz"    # 200
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/startupz"   # 200
   ```
2. The authenticated deep health check passes, and the config file is on the volume:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId openclaw --shell-cmd 'sh -c' \
     --cmd '"openclaw gateway health && ls /data/.openclaw/openclaw.json && openclaw config get gateway.publicOrigin"'
   ```
   Expect a healthy report, the file path, and `https://<HOST>`. The CLI inside the container authenticates with the `OPENCLAW_GATEWAY_TOKEN` already in its environment.
3. **This step needs a human:** open `https://$HOST/`, paste the gateway token into Settings, and connect. If the UI says *pairing required*, approve the browser:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId openclaw --cmd 'openclaw devices list'
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId openclaw --cmd 'openclaw devices approve <REQUEST_ID>'
   ```
   Take `<REQUEST_ID>` from the pending entry in the list, and approve only a request the user confirms is their own browser.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Gateway exits, logs say auth is required for a non-loopback bind | `OPENCLAW_GATEWAY_TOKEN` not injected | Check the `openclaw-env` group; restart |
| Control UI shows an origin error | `gateway.publicOrigin` missing or not the exact `https://` host | Redo step 8; add custom domains to `gateway.controlUi.allowedOrigins` |
| `EACCES` writing `/data/.openclaw` | Volume not writable by the image's `node` user | Restart once; if it persists, see [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) |
| Config lost after a redeploy | Volume not attached, or `OPENCLAW_STATE_DIR` unset | Check `northflank get volume … --volumeId openclaw-data` shows the service; check the env var |
| Agent replies fail with a provider auth error | Model key missing or model ID wrong | Check the key name in the group; run `openclaw models status` |
| Restarts during heavy use | Out of memory | Move to `nf-compute-200-4` |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Never print `OPENCLAW_GATEWAY_TOKEN`, model keys or channel tokens. Don't paste them into `openclaw.json`; keep them in the secret group.
- Never set `gateway.auth.mode` to `none` or enable `dangerouslyAllowHostHeaderOriginFallback` to get past an error.
- OpenClaw agents can run tools with broad access inside the container. Ask the user before enabling exec tools, browsers or new channels.
- Don't scale past one instance, and get explicit confirmation before deleting the volume.

## FAQ

### Where should I host OpenClaw?
On an always-on container or VM with a persistent disk, an HTTPS URL that passes WebSockets, and around 2 GB of memory. OpenClaw documents Docker, Kubernetes and several cloud hosts. A container platform with volumes, like the setup above, gives you TLS, restarts and secrets without managing a server.

### What's the difference between this and the Northflank stack template?
The stack puts a wrapper with a password-protected `/setup` wizard in front of the gateway and serves the Control UI under `/openclaw`. This guide runs the official image on its own and configures it through the CLI. Both keep state on a volume.

### How do I connect Telegram or Discord?
Add the bot token to `openclaw-env` (for example `TELEGRAM_BOT_TOKEN`), restart, then run `openclaw channels add --channel telegram --use-env` through `northflank exec service`. With `--use-env`, OpenClaw reads the token from the environment instead of copying it into the config.

### How do I upgrade?
Change the image tag with `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId openclaw --input '{"deployment":{"instances":1,"external":{"imagePath":"ghcr.io/openclaw/openclaw:<NEW_VERSION>"}}}'`. The image entrypoint runs `openclaw doctor --fix --non-interactive` against the mounted state before the gateway starts. Back up the volume first.

## Conclusion

You now have OpenClaw's gateway on the official image, with token auth, its state on a persistent volume, the public origin configured, and a default model reading its key from a secret group.

**Next:** connect a channel, then add your own domain by [linking a domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) and adding it to the allowed origins. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy AI agents in production](deploy-ai-agents-in-production.md)
- [How to self-host n8n](self-host-n8n.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to run Claude Code in a sandbox](run-claude-code-in-a-sandbox.md)
