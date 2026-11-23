<!--
agent-guide-meta:
  id: NF-STK-DISCORD
  description: "How to host a Discord bot 24/7 with discord.js or discord.py in Docker: the token as a secret, no public port, one instance and automatic restarts on failure."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://discord.js.org/docs
    - https://discordjs.guide/
    - https://discordpy.readthedocs.io/en/stable/quickstart.html
    - https://discordpy.readthedocs.io/en/stable/intents.html
    - https://docs.discord.com/developers/topics/gateway
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/application/observe/view-logs
  related: [NF-STK-TELEGRAM, NF-JOB-002, NF-DEP-004, NF-DEP-001, NF-STK-MINECRAFT]
-->

# How to host a Discord bot

To host a Discord bot, run it as an always-on worker process (discord.js or discord.py) in a container, pass the bot token as a secret environment variable, and keep exactly one instance running with automatic restarts. A Discord bot opens an outbound WebSocket to the Discord Gateway, so it needs no public port or URL, and a second copy makes every command run twice.

This guide does it on Northflank with the CLI: a secret group holding `DISCORD_TOKEN`, and a single-instance, port-less service built from your `Dockerfile` with a heartbeat check that restarts a hung bot. You end with a bot online 24/7, verified from its logs and a slash command. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run. A human only has to log in once, share the repository with Northflank and create the bot in the Discord Developer Portal.

## TL;DR: how to host a Discord bot

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a discord.js or discord.py bot and `DISCORD_TOKEN` is set, and confirm the team, repository, branch and project before changing anything. Give Northflank access to the repository (once).
3. Make the bot read `DISCORD_TOKEN` from the environment, write a heartbeat file while connected and exit cleanly on `SIGTERM`; add a `Dockerfile` and push.
4. Store the token with `northflank put secret`, then create the service with `northflank put service combined`: `"ports": []`, one instance and a CMD liveness check on the heartbeat file.
5. Wait for the deploy, then check that the live build is your pushed commit, the logs show `Logged in as`, and a slash command answers in Discord.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why a port-less, single-instance worker?

A Discord bot isn't a web server. It logs in with its token, opens a WebSocket to the Discord Gateway and keeps it open, receiving events as they happen. Three rules follow:

- **No public port.** Discord never calls your bot; the bot connects out. A health check on an HTTP port nothing listens on would restart the bot forever.
- **One instance.** Each running copy opens its own Gateway session and receives the same events, so two copies reply twice. Large bots split work with sharding, which is a code change, not extra replicas.
- **Always on, restarted on failure.** The process has to run 24/7, and something has to restart it if it crashes or hangs.

That's the answer to where to host a Discord bot: anywhere that keeps one process running all day. Your own PC works only while it's on. Serverless and static hosts stop processes between requests, so they can't hold a Gateway connection (HTTP-only interaction bots are the exception). A VPS works if you set up a process supervisor, restarts and updates yourself. A container platform runs your `Dockerfile` as a long-running service, restarts it, and rebuilds on every push.

## Prerequisites

- **A bot that runs locally** and prints `Logged in as …` (stop it with Ctrl+C). discord.js: `npm ci && node index.js`; discord.py:
  ```bash
  pip install -r requirements.txt && python bot.py
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The bot exists in Discord (needs a human).** In the [Discord Developer Portal](https://discord.com/developers/applications), create an application, open **Bot**, reset and copy the token, and turn on any privileged intents the code uses (Message Content, Server Members, Presence). Then use **OAuth2 → URL Generator** with the `bot` and `applications.commands` scopes to invite the bot to a server. Export the token in your shell without pasting it into chat: `export DISCORD_TOKEN=…`.

Names used below: project `discord-bot`, secret group `discord-env`, service `discord-bot`.

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
jq -r '"discord.js: \(.dependencies["discord.js"] // "missing")"' package.json 2>/dev/null
grep -isE '^discord\.py|"discord\.py' requirements.txt pyproject.toml
ls package-lock.json requirements.txt Dockerfile 2>/dev/null
[ -n "$DISCORD_TOKEN" ] && echo "DISCORD_TOKEN: set" || echo "DISCORD_TOKEN: missing"

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId discord-bot -o json >/dev/null 2>&1 \
  && echo "project: discord-bot exists and will be reused" \
  || echo "project: discord-bot will be created"
```

- If the first line says `missing` (or prints nothing, as in a Python repo) and the `grep` finds no `discord.py`, this isn't a discord.js or discord.py bot. Stop and use a guide for your framework.
- If `DISCORD_TOKEN` says `missing`, export it first (see the prerequisites). The check never prints the value.
- Step 5 assumes `package-lock.json` for discord.js and `requirements.txt` for discord.py; for pnpm, yarn or Poetry, swap the install command.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "discord-bot", "region": "europe-west" }'
export PROJECT_ID=discord-bot

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Prepare the bot code

The bot must read its token from the environment, and it should also do two small things for production:

- **Write a heartbeat file** every 30 seconds while connected, so a health check can tell a hung bot from a working one.
- **Handle `SIGTERM`** by closing the Gateway connection, so the bot shows offline straight away during redeploys instead of lingering.

**discord.js (Node.js, v14):**

```js
// index.js
const fs = require("node:fs");
const { Client, Events, GatewayIntentBits } = require("discord.js");

const client = new Client({ intents: [GatewayIntentBits.Guilds] });

client.once(Events.ClientReady, (c) => console.log(`Logged in as ${c.user.tag}`));

client.on(Events.InteractionCreate, async (interaction) => {
  if (interaction.isChatInputCommand() && interaction.commandName === "ping") {
    await interaction.reply("pong");
  }
});

setInterval(() => {
  if (client.isReady()) fs.writeFileSync("/tmp/heartbeat", String(Date.now()));
}, 30_000);

process.on("SIGTERM", async () => { await client.destroy(); process.exit(0); });

client.login(process.env.DISCORD_TOKEN);
```

Register the `/ping` command with the discord.js guide's `deploy-commands.js` script (it uses `REST` and `Routes.applicationCommands`), reading the token and application ID from the environment. You run it once in step 8.

```dockerfile
FROM node:24-slim
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
USER node
CMD ["node", "index.js"]
```

**discord.py (Python, 2.x):**

```python
# bot.py
import os, signal, time, asyncio
import discord
from discord.ext import tasks

client = discord.Client(intents=discord.Intents.default())

@tasks.loop(seconds=30)
async def heartbeat():
    if client.is_ready():
        with open("/tmp/heartbeat", "w") as f:
            f.write(str(time.time()))

@client.event
async def on_ready():
    print(f"Logged in as {client.user}", flush=True)
    if not heartbeat.is_running():
        heartbeat.start()

async def main():
    loop = asyncio.get_running_loop()
    loop.add_signal_handler(signal.SIGTERM, lambda: asyncio.create_task(client.close()))
    async with client:
        await client.start(os.environ["DISCORD_TOKEN"])

asyncio.run(main())
```

```dockerfile
FROM python:3.13-slim
WORKDIR /app
ENV PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "bot.py"]
```

For slash commands in discord.py, use `discord.app_commands.CommandTree` and call `await tree.sync()` once. If the bot reads message text, enable `intents.message_content = True` in code and the Message Content intent in the portal; without both, `message.content` is empty.

Leave `EXPOSE` out of the `Dockerfile`, because the bot has nothing to expose. Northflank builds what's on the remote branch, not your local working tree, so commit and push the files first:

```bash
git add Dockerfile index.js deploy-commands.js package.json package-lock.json   # discord.py: Dockerfile bot.py requirements.txt
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Store the token in a secret group

Build the JSON with `jq` so the token never appears on the command line or in logs:

```bash
jq -n --arg t "$DISCORD_TOKEN" '{
  name: "discord-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "discord-bot", type: "service" } ] },
  secrets: { variables: { DISCORD_TOKEN: $t } }
}' > discord-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file discord-env.json >/dev/null && rm discord-env.json
unset DISCORD_TOKEN
```

The group is restricted to the bot's service. If the call is rejected because `discord-bot` doesn't exist yet, re-run it after step 7 and restart the service. For `deploy-commands.js`, also add the application ID as `DISCORD_CLIENT_ID` (it isn't secret, but keeping it here is simplest).

## Step 7: Create the port-less service and wait for it

Write `discord-bot.json`:

```json
{
  "name": "discord-bot",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [],
  "healthChecks": [
    { "protocol": "CMD", "type": "livenessProbe", "cmd": "/bin/sh -c \"find /tmp/heartbeat -mmin -2 | grep -q .\"",
      "initialDelaySeconds": 90, "periodSeconds": 30, "timeoutSeconds": 5, "failureThreshold": 4 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file discord-bot.json
```

Use the same repository URL, provider and branch as in step 4. The first build starts automatically. How the bot stays up:

- **Crashes:** if the process exits, the container restarts.
- **Hangs:** if the heartbeat file is older than 2 minutes for 4 checks in a row (about 2 more minutes), the liveness probe fails and the container is replaced. discord.js and discord.py reconnect on their own after short network drops, so this only fires when the bot is really stuck.
- **New code:** every push to the branch rebuilds and redeploys.

`nf-compute-20` (0.2 vCPU, 512 MB) suits a small bot. Bots in many servers, or that cache a lot of members, need more memory.

The first build takes a few minutes. This loop checks every 10 seconds. The bot has no public port, so there's no URL to print:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId discord-bot --noDefaults -o json)
  B=$(echo "$S" | jq -r '(.data // .).status.build.status // "NONE"')
  D=$(echo "$S" | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "build=$B deployment=$D"
  case "$B" in FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo BUILD_FAILED; break;; esac
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  [ "$B" = "SUCCESS" ] && [ "$D" = "COMPLETED" ] && { OK=1; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
```

`COMPLETED` means the container is running. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId discord-bot`.

## Step 8: Register slash commands (once)

Commands are registered with Discord's API, not on every start. Run the registration script once inside the running container. `exec` needs `script` to keep output without a TTY (Linux: `script -qefc '<cmd>' /dev/null`):

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId discord-bot \
  --shell-cmd 'sh -c' --cmd '"node deploy-commands.js"'
```

With discord.py, call `tree.sync()` from a small script run the same way. Re-run registration only when command names or options change. Global commands can take a while to appear in clients; guild-scoped commands appear at once.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId discord-bot --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The bot logged in:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId discord-bot --lineLimit 200 --textIncludes "Logged in as"
   ```
3. Exactly one instance is running:
   ```bash
   northflank get service --projectId "$PROJECT_ID" --serviceId discord-bot --noDefaults -o json | jq '(.data // .).deployment.instances'   # 1
   ```
4. The heartbeat is fresh (prints `HEARTBEAT_OK`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId discord-bot \
     --shell-cmd 'sh -c' --cmd '"find /tmp/heartbeat -mmin -2 | grep -q . && echo HEARTBEAT_OK"'
   ```
5. **This step needs a human:** the bot shows as online in the server's member list, and `/ping` replies `pong` exactly once.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Logs show `TokenInvalid` or `LoginFailure` | Token missing, wrong or reset | Ask the user for a fresh token; redo step 6; restart |
| `Used disallowed intents` | Privileged intent in code but off in the portal | Turn it on in the Developer Portal, or remove it from the code |
| Every command answers twice | Two instances, or the bot also running on a laptop | Keep `instances: 1`; stop other copies using the same token |
| Container restarts every few minutes | Heartbeat never written (bot never ready), or the probe path is wrong | Check logs for login errors; check the heartbeat code writes `/tmp/heartbeat` |
| `/ping` doesn't appear | Commands not registered, or missing `applications.commands` scope | Run step 8; re-invite with both scopes |
| `message.content` is empty | Message Content intent off | Enable it in code and in the portal |
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
- Never print, log or commit `DISCORD_TOKEN`. Build JSON with `jq --arg` and delete temp files. If a token leaks, tell the user to reset it in the portal.
- Never scale the bot above one instance unless the code shards.
- Ask the user before enabling privileged intents; large bots need Discord's approval for them.
- Read errors with `northflank get service logs`, and don't delete the service or secret group without explicit confirmation.

## FAQ

### Where should I host my Discord bot 24/7?
Anywhere that keeps one process running all day and restarts it when it fails: a container platform, a VPS with a process supervisor, or a home server. Serverless hosts don't fit a Gateway bot because they stop the process between requests. A container platform that builds your repo and runs it as a service, as in this guide, gives 24/7 uptime without managing a machine.

### Does my Discord bot need a port or a domain?
No. A Gateway bot connects out to Discord. You only need a public HTTPS URL if you switch to HTTP interactions (an Interactions Endpoint URL) instead of the Gateway.

### Can I run two instances for redundancy?
Not without sharding. Two copies on the same token both receive every event and both reply. Rely on restarts and the health check instead.

### Does the same setup work for a Telegram bot?
Mostly. A Telegram bot in long-polling mode is the same port-less worker. In webhook mode it needs a public HTTPS port; see [how to deploy a Telegram bot](deploy-telegram-bot.md).

## Conclusion

You now have a Discord bot built from your repo, running as one instance with no public port, with its token in a secret group and a heartbeat check that restarts it if it hangs.

**Next:** if the bot needs to store data, add a managed database as in [how to deploy a Node.js application](deploy-nodejs-application.md), and use [view logs](https://northflank.com/docs/v1/application/observe/view-logs) to follow errors. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy a Telegram bot](deploy-telegram-bot.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to host a Minecraft server](host-minecraft-server.md)
