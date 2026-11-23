<!--
agent-guide-meta:
  id: NF-STK-TELEGRAM
  description: "How to deploy a Telegram bot in webhook mode on a public HTTPS port with setWebhook and secret_token, or as a long-polling worker, with the token as a secret."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://core.telegram.org/bots/api
    - https://core.telegram.org/bots/webhooks
    - https://grammy.dev/guide/deployment-types
    - https://grammy.dev/ref/core/webhookcallback
    - https://docs.python-telegram-bot.org/en/stable/telegram.ext.application.html
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-STK-DISCORD, NF-STK-EXPRESS, NF-DEP-004, NF-DEP-001]
-->

# How to deploy a Telegram bot

To deploy a Telegram bot, either run it in webhook mode behind a public HTTPS URL, registering that URL with `setWebhook` and a `secret_token` that every request must echo back, or run it as a long-polling worker with no public port and exactly one instance. Either way, the bot token belongs in a secret environment variable. The two modes exclude each other: `getUpdates` stops working while a webhook is set.

This guide does it on Northflank with the CLI, using grammY on Node.js: the token and webhook secret in a secret group, a service built from your `Dockerfile` with a public HTTPS port, and the bot calling `setWebhook` on its own URL at startup. You end with a bot that replies in Telegram, checked with `getWebhookInfo` and a `401` for unsigned requests. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run. A human only has to log in once, share the repository with Northflank and create the bot token.

## TL;DR: how to deploy a Telegram bot

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a grammY bot and `BOT_TOKEN` is set, and confirm the team, repository, branch and project before changing anything. Give Northflank access to the repository (once).
3. Make the bot serve `POST /telegram` with grammY's `webhookCallback` (checking `WEBHOOK_SECRET`) and `GET /health`, and call `setWebhook` at startup when `PUBLIC_URL` is set. Push.
4. Store `BOT_TOKEN` and a generated `WEBHOOK_SECRET` with `northflank put secret`, then create the service with `northflank put service combined` on a public HTTP port 8080.
5. Once it's live, set `PUBLIC_URL` to the service's public URL with `northflank patch service combined` and restart.
6. Check that the live build is your pushed commit, `getWebhookInfo` shows your URL with no error, an unsigned `POST` gets `401`, and the bot replies.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Webhook or long polling?

Telegram delivers updates one of two ways:

| | Webhook | Long polling |
|---|---|---|
| How | Telegram POSTs each update to your HTTPS URL | The bot calls `getUpdates` in a loop |
| Needs | A public HTTPS URL on port 443, 80, 88 or 8443 | Only outbound internet |
| Instances | Any number; the web app is stateless | Exactly one; a second gets `409 Conflict` |
| Good for | Production, scaling, fast replies | Simple bots, development |

Webhooks need a valid certificate on a supported port. Northflank's public ports serve HTTPS on 443 with an automatic certificate, so webhook mode works without extra setup. That makes webhook mode this guide's main path. Long polling is covered in step 10.

So where should you deploy a Telegram bot? Somewhere that either keeps a process running (polling) or serves HTTPS at a stable URL (webhooks). Your PC works only while it's on. Serverless functions can handle webhooks, but they time out on slow handlers and can't run a polling loop. A VPS works if you set up TLS, a process supervisor and restarts. A container platform does both modes from one `Dockerfile`, with HTTPS and restarts built in.

## Prerequisites

- **A grammY bot that runs locally** in polling mode (it prints `Polling as @<bot>`; stop it with Ctrl+C):
  ```bash
  npm ci && BOT_MODE=polling node bot.mjs
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **`openssl`** (to generate the webhook secret).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The bot token (needs a human).** Create the bot with [@BotFather](https://t.me/BotFather) (`/newbot`) and export the token in your shell without pasting it into chat: `export BOT_TOKEN=…`.

Names used below: project `telegram-bot`, secret group `telegram-env`, service `telegram-bot`.

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
jq -r '"grammy: \(.dependencies.grammy // "missing")  express: \(.dependencies.express // "missing")"' package.json
grep -isE 'python-telegram-bot' requirements.txt pyproject.toml
ls package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null
[ -n "$BOT_TOKEN" ] && echo "BOT_TOKEN: set" || echo "BOT_TOKEN: missing"

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId telegram-bot -o json >/dev/null 2>&1 \
  && echo "project: telegram-bot exists and will be reused" \
  || echo "project: telegram-bot will be created"
```

- If `grammy` says `missing`, this isn't a grammY bot. If the `grep` found `python-telegram-bot`, use the `run_webhook` / `run_polling` note in step 5 with your own Python `Dockerfile`; the Northflank steps are the same. Otherwise, stop and use a guide for your framework. If `express` is missing, step 5 needs it: `npm install express`.
- If `BOT_TOKEN` says `missing`, export it first (see the prerequisites). The check never prints the value.
- The lockfile tells you the package manager. Step 5 assumes `package-lock.json`; for pnpm or yarn, swap the install command.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "telegram-bot", "region": "europe-west" }'
export PROJECT_ID=telegram-bot

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Prepare the bot code

`bot.mjs` handles both modes, chosen by `BOT_MODE`. In webhook mode it serves Express, rejects requests without the right `X-Telegram-Bot-Api-Secret-Token` header (grammY returns `401`), and registers the webhook once it's listening:

```js
// bot.mjs  (package.json: "type": "module"; deps: grammy, express)
import express from "express";
import { Bot, webhookCallback } from "grammy";

const bot = new Bot(process.env.BOT_TOKEN);
bot.command("start", (ctx) => ctx.reply("Hello! I'm running."));
bot.on("message:text", (ctx) => ctx.reply(`You said: ${ctx.message.text}`));

if (process.env.BOT_MODE === "polling") {
  process.once("SIGTERM", () => bot.stop());
  await bot.start({ onStart: (me) => console.log(`Polling as @${me.username}`) }); // deletes any webhook first
} else {
  const app = express();
  app.use(express.json());
  app.get("/health", (_req, res) => res.send("ok"));
  app.post("/telegram", webhookCallback(bot, "express", { secretToken: process.env.WEBHOOK_SECRET }));

  const server = app.listen(Number(process.env.PORT ?? 8080), "0.0.0.0", async (err) => {
    if (err) throw err;
    if (process.env.PUBLIC_URL) {
      await bot.api.setWebhook(`${process.env.PUBLIC_URL}/telegram`, { secret_token: process.env.WEBHOOK_SECRET });
      console.log(`Webhook set to ${process.env.PUBLIC_URL}/telegram`);
    }
  });
  process.once("SIGTERM", () => server.close());
}
```

Add `scripts/webhook-info.mjs` for the check at the end. It prints the webhook state, never the token:

```js
const r = await fetch(`https://api.telegram.org/bot${process.env.BOT_TOKEN}/getWebhookInfo`);
const { result } = await r.json();
console.log(JSON.stringify({ url: result.url, pending: result.pending_update_count, error: result.last_error_message ?? null }));
```

And a `Dockerfile`:

```dockerfile
FROM node:24-slim
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
USER node
CMD ["node", "bot.mjs"]
```

grammY gives each webhook update 10 seconds by default; if a handler takes longer, Telegram retries the update. Push slow work to a queue. With python-telegram-bot, the same split is `application.run_webhook(listen="0.0.0.0", port=8080, url_path="telegram", webhook_url=…, secret_token=…)` versus `application.run_polling()`.

Northflank builds what's on the remote branch, not your local working tree, so commit and push the files first:

```bash
git add bot.mjs scripts/webhook-info.mjs package.json package-lock.json Dockerfile
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Store the token and webhook secret

`secret_token` allows 1–256 characters from `A-Z`, `a-z`, `0-9`, `_` and `-`, so a hex string works. Build the JSON with `jq` so neither value appears in logs:

```bash
WEBHOOK_SECRET=$(openssl rand -hex 32)
jq -n --arg t "$BOT_TOKEN" --arg s "$WEBHOOK_SECRET" '{
  name: "telegram-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "telegram-bot", type: "service" } ] },
  secrets: { variables: { BOT_TOKEN: $t, WEBHOOK_SECRET: $s } }
}' > telegram-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file telegram-env.json >/dev/null && rm telegram-env.json
unset BOT_TOKEN WEBHOOK_SECRET
```

If the restriction is rejected because `telegram-bot` doesn't exist yet, re-run this after step 7 and restart the service.

## Step 7: Create the webhook service and wait for it

Write `telegram-bot.json`:

```json
{
  "name": "telegram-bot",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "8080" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/health", "port": 8080,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/health", "port": 8080,
      "initialDelaySeconds": 30, "periodSeconds": 20, "timeoutSeconds": 2, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file telegram-bot.json
```

Use the same repository URL, provider and branch as in step 4. The first build starts automatically. This loop checks every 10 seconds and prints the bot's public URL when it's live:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId telegram-bot --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId telegram-bot --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId telegram-bot --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId telegram-bot`. `PUBLIC_URL` isn't set yet, so the bot is up but no webhook is registered.

## Step 8: Point the webhook at the public URL

Use the `$HOST` that step 7 printed, then send the complete `runtimeEnvironment` map (a patch may replace it) and restart:

```bash
[ -n "$HOST" ] || { echo "NO_PUBLIC_HOST: stop here; the public port has no DNS name yet"; false; }

[ -n "$HOST" ] && northflank patch service combined --projectId "$PROJECT_ID" --serviceId telegram-bot --noDefaults -o json \
  --input "$(jq -n --arg h "$HOST" '{ runtimeEnvironment: { PORT: "8080", PUBLIC_URL: ("https://" + $h) } }')"
northflank restart service --projectId "$PROJECT_ID" --serviceId telegram-bot --noDefaults -o json
```

On start, the bot calls `setWebhook("https://<HOST>/telegram", { secret_token })`. Calling it on every start is harmless. If you add a custom domain later, change `PUBLIC_URL` to it.

## Step 9: Wait for the restart

Re-run the polling loop from step 7, then check the bot registered the webhook:

```bash
northflank get service logs --projectId "$PROJECT_ID" --serviceId telegram-bot --lineLimit 100 --textIncludes "Webhook set"
```

## Step 10 (alternative): Run as a long-polling worker

If you'd rather not expose a URL, run the same image in polling mode: no ports, no HTTP health checks, one instance. grammY's `bot.start()` deletes any existing webhook before polling.

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId telegram-bot --noDefaults -o json \
  --input '{ "ports": [], "healthChecks": [], "deployment": { "instances": 1 }, "runtimeEnvironment": { "BOT_MODE": "polling" } }'
```

Never scale a polling bot above one instance: Telegram allows one `getUpdates` caller per token, and the others fail with `409 Conflict: terminated by other getUpdates request`. Logs show `Polling as @<bot>` when it's running.

## How to know it worked

For webhook mode, you're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId telegram-bot --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. Telegram has your URL and no delivery error. `exec` needs `script` to keep output without a TTY (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId telegram-bot \
     --shell-cmd 'sh -c' --cmd '"node scripts/webhook-info.mjs"'
   ```
   Expect `"url":"https://<HOST>/telegram"`, `"error":null` and a `pending` count that stays near 0.
3. Requests without the secret are rejected:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' -X POST -H 'Content-Type: application/json' -d '{}' "https://$HOST/telegram"   # 401
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/health"                                                       # 200
   ```
4. **This step needs a human:** send `/start` to the bot in Telegram and get `Hello! I'm running.` back.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `getWebhookInfo` shows `url: ""` | `PUBLIC_URL` not set, or the service wasn't restarted | Redo step 8 |
| `last_error_message: Wrong response from the webhook: 401` | `WEBHOOK_SECRET` changed after `setWebhook` | Restart so the bot re-registers with the current secret |
| `last_error_message` mentions a timeout, `pending` grows | Handler slower than the webhook timeout | Reply fast and push slow work to a queue |
| `409 Conflict: terminated by other getUpdates request` | Two pollers on one token (a second instance, or a laptop) | Keep one instance; stop other copies |
| Polling bot gets nothing | A webhook is still set | `bot.start()` deletes it; with other libraries call `deleteWebhook` |
| `401 Unauthorized` from Telegram in logs | Wrong or revoked token | Ask the user for a new token from @BotFather; redo step 6 |
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
- Never print `BOT_TOKEN` or `WEBHOOK_SECRET`, and never log full Telegram API URLs (they contain the token). Use the `webhook-info` script, which prints only the webhook state.
- Never accept webhook requests without checking the secret header.
- Don't switch modes, revoke the token or delete the service without asking the user first.

## FAQ

### Where should I deploy my Telegram bot?
Somewhere that serves HTTPS at a stable URL (webhooks) or keeps a process running (polling), and restarts it if it fails. A container platform does both from one `Dockerfile`, with HTTPS and restarts included. A VPS works if you handle TLS and process supervision. Your own PC works only while it's on.

### Is webhook mode better than polling?
For production, usually. Updates arrive as soon as they happen, the web app can run more than one instance, and nothing polls while idle. Polling is simpler and needs no public URL, but it's limited to one instance.

### Do I have to use port 443?
Telegram accepts webhooks on ports 443, 80, 88 and 8443. Northflank's public ports serve HTTPS on 443 and route to your container's port, so your app can listen on 8080.

### Can I use python-telegram-bot instead of grammY?
Yes. Use `run_webhook` with `listen="0.0.0.0"`, `port=8080`, `url_path`, `webhook_url` and `secret_token`, or `run_polling()` for worker mode. The Northflank steps are the same.

## Conclusion

You now have a Telegram bot in webhook mode on a public HTTPS URL, with its token and webhook secret in a secret group, unsigned requests rejected, and a one-line switch to a long-polling worker.

**Next:** for a general Express backend, see [how to deploy an Express app](deploy-express-app.md); for a bot that also lives on Discord, see [how to host a Discord bot](host-discord-bot.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to host a Discord bot](host-discord-bot.md)
- [How to deploy an Express app](deploy-express-app.md)
- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
