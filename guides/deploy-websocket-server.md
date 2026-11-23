<!--
agent-guide-meta:
  id: NF-STK-WEBSOCKET
  description: "How to deploy a WebSocket server in production: a Node.js ws app behind HTTPS (wss), heartbeats for idle connections and Redis pub/sub to scale out instances."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://github.com/websockets/ws
    - https://socket.io/docs/v4/using-multiple-nodes/
    - https://socket.io/docs/v4/redis-adapter/
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/networking-on-northflank
    - https://northflank.com/docs/v1/application/network/load-balancing
    - https://northflank.com/docs/v1/application/scale/scale-instances
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-redis-on-northflank
  related: [NF-DEP-004, NF-DB-003, NF-DEP-001, NF-STK-NEST]
-->

# How to deploy a WebSocket server

To deploy a WebSocket server, run it as a long-lived process that serves HTTP on `0.0.0.0:$PORT` and accepts the `Upgrade` handshake on that port, behind a TLS proxy so browsers connect with `wss://`. Ping clients every 30 seconds so idle connections aren't dropped, and close them cleanly on shutdown. For more than one instance, add Redis pub/sub so messages reach clients on every instance.

This guide does it on Northflank with the CLI: a Node.js `ws` server built from your repo's `Dockerfile`, a public HTTP port that carries the upgrade over HTTPS, and a managed Redis addon for fan-out. You end with a `wss://` URL that echoes messages, then two instances relaying messages to each other, verified with Node's built-in WebSocket client. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a WebSocket server

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Node.js WebSocket server and confirm the team, repository, branch and project before changing anything, then give Northflank access to the repository (once).
3. Write a `ws` server that answers `/healthz` over HTTP, upgrades WebSocket connections on the same port, pings clients every 30 seconds and publishes messages to Redis when `REDIS_URL` is set. Add a `Dockerfile` and push.
4. Create a project and a service built from your repo with `northflank put service combined --file ws.json`, on a public `HTTP` port with HTTP health checks.
5. Wait for the deploy, check that the live build is your pushed commit and that a `wss://` client gets its message echoed back.
6. To scale out, create Redis, link `REDIS_MASTER_URL` as `REDIS_URL`, set `instances` to 2, and check that a message from one client reaches another.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Where to deploy a WebSocket server

A WebSocket server holds a TCP connection open for minutes or hours. That rules out request-scoped serverless functions. Hosts that work keep a process running and pass `Upgrade` requests through their proxy: a VM with nginx (you configure upgrade headers and timeouts), Kubernetes with an ingress controller, or a container platform whose load balancer handles the upgrade.

On Northflank, the networking docs say ordinary service ports support HTTP and HTTP/2 on public ingress, "including WebSockets and gRPC over those protocols." Public HTTP ports are served on 443 with TLS, and port 80 redirects to HTTPS, so clients use `wss://<host>` while your server speaks plain `ws` on its internal port. Two things are your job:

- **Idle connections.** The docs don't publish an idle timeout for WebSockets, so don't depend on one. The server pings every 30 seconds, which keeps traffic flowing and detects dead clients.
- **Scaling.** Each instance only knows its own clients. Redis pub/sub relays messages between instances. Plain WebSocket connections don't need sticky sessions, because each connection stays on the instance that accepted it. Socket.IO's HTTP long-polling fallback does need them (see the Socket.IO section).

## Prerequisites

- **A Node.js WebSocket server that starts locally**, or a Node.js app you'll add one to in step 5:
  ```bash
  npm ci && node server.js
  ```
  Stop it with Ctrl+C once it prints `listening on 8080`.
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI and the built-in `WebSocket` client used in the checks) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

Names used below: project `ws-app`, service `ws`, addon `ws-redis`, secret group `ws-env`.

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
jq -r '"ws: \(.dependencies.ws // "missing")  socket.io: \(.dependencies["socket.io"] // "missing")  ioredis: \(.dependencies.ioredis // "missing")"' package.json
ls package-lock.json pnpm-lock.yaml yarn.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId ws-app -o json >/dev/null 2>&1 \
  && echo "project: ws-app exists and will be reused" \
  || echo "project: ws-app will be created"
```

- If there's no `package.json`, this isn't a Node.js app. Stop and use a guide for your runtime. If `ws` and `ioredis` say `missing`, step 5 installs them. If the app uses `socket.io`, also read the Socket.IO section after step 9.
- The lockfile tells you the package manager. Step 5 assumes `package-lock.json`; for pnpm or yarn, swap the install command.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "ws-app", "region": "europe-west" }'
export PROJECT_ID=ws-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Write the server

Install the dependencies with `npm install ws ioredis` and set `"type": "module"` in `package.json`. Then write `server.js`:

```js
import http from 'node:http';
import { WebSocketServer, WebSocket } from 'ws';
import Redis from 'ioredis';

const port = Number(process.env.PORT ?? 8080);
const server = http.createServer((req, res) => {
  if (req.url === '/healthz') return res.writeHead(200).end('ok');
  res.writeHead(404).end();
});
const wss = new WebSocketServer({ server });

// Fan-out: with REDIS_URL, every instance relays messages through one channel.
const pub = process.env.REDIS_URL ? new Redis(process.env.REDIS_URL) : null;
const sub = pub ? pub.duplicate() : null;
const sendLocal = (msg) => { for (const c of wss.clients) if (c.readyState === WebSocket.OPEN) c.send(msg); };
if (sub) { await sub.subscribe('messages'); sub.on('message', (_ch, msg) => sendLocal(msg)); }

wss.on('connection', (ws) => {
  ws.isAlive = true;
  ws.on('pong', () => { ws.isAlive = true; });
  ws.on('message', (data) => { const msg = data.toString(); pub ? pub.publish('messages', msg) : sendLocal(msg); });
});

// Heartbeat: ping every 30 s, drop clients that didn't answer the last ping.
const heartbeat = setInterval(() => {
  for (const ws of wss.clients) {
    if (!ws.isAlive) { ws.terminate(); continue; }
    ws.isAlive = false; ws.ping();
  }
}, 30_000);

process.on('SIGTERM', () => {
  clearInterval(heartbeat);
  for (const ws of wss.clients) ws.close(1001, 'server restarting');
  server.close(() => process.exit(0));
});
server.listen(port, '0.0.0.0', () => console.log(`listening on ${port}`));
```

The heartbeat follows the `ws` README. Messages go to every client, including the sender, which makes the echo check easy. Add your own routing and authentication before going live.

Add a `Dockerfile` if there's none (if one exists, compare it with this one):

```dockerfile
FROM node:22-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY . .
ENV NODE_ENV=production
EXPOSE 8080
CMD ["node", "server.js"]
```

`CMD` in exec form makes `node` PID 1, so it receives `SIGTERM` directly on deploys.

Northflank builds what's on the remote branch, not your local working tree, so commit and push the files first:

```bash
git add server.js package.json package-lock.json Dockerfile
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create the service

Write `ws.json`:

```json
{
  "name": "ws",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "8080" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/healthz", "port": 8080,
      "initialDelaySeconds": 3, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/healthz", "port": 8080,
      "initialDelaySeconds": 15, "periodSeconds": 20, "timeoutSeconds": 2, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file ws.json
```

Use the same repository URL, provider and branch as in step 4. The first build starts automatically. The port protocol is `HTTP`, not `TCP`. The WebSocket handshake is an HTTP request, and the docs say TCP service ports need a separate Layer 4 load balancer to be reachable from the internet.

## Step 7: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds and prints your URL when the app is live. Clients connect to the same host with `wss://`:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId ws --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId ws --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId ws --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId ws`.

## Step 8: Connect over `wss://`

Don't continue if step 7 printed `ROLLOUT_NOT_COMPLETE`. Use the `$HOST` that step 7 printed:

```bash
node -e '
const ws = new WebSocket(process.argv[1]);
ws.onopen = () => ws.send("hello");
ws.onmessage = (e) => { console.log("got:", e.data); ws.close(); };
ws.onerror = (e) => { console.error("error", e.message); process.exit(1); };
' "wss://$HOST/"
```

This prints `got: hello`. Browsers use `new WebSocket("wss://<host>/")`; HTTPS pages can't open `ws://` connections.

## Step 9: Scale out with Redis

Create Redis, link its URL as `REDIS_URL`, and run two instances:

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "ws-redis", "type": "redis", "version": "8-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 4096, "replicas": 1 },
  "typeSpecificSettings": { "redisMaxMemoryPolicy": "noeviction", "redisSentinelEnabled": false }
}'
OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId ws-redis --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }

northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "ws-env", "type": "secret", "secretType": "environment", "priority": 10,
  "restrictions": { "restricted": false },
  "addonDependencies": [ { "addonId": "ws-redis", "keys": [ { "keyName": "REDIS_MASTER_URL", "aliases": ["REDIS_URL"] } ] } ],
  "secrets": { "variables": {} }
}'
northflank get secret-link --projectId "$PROJECT_ID" --secretId ws-env --addonId ws-redis --noDefaults -o json \
  | jq '(.data // .) | {linked, linkedKeys, availableKeys}'

northflank patch service combined --projectId "$PROJECT_ID" --serviceId ws --noDefaults -o json --input '{"deployment": {"instances": 2}}'
northflank restart service --projectId "$PROJECT_ID" --serviceId ws --noDefaults -o json
```

If `linkedKeys` is empty, re-run `put secret` with the key name from `availableKeys`, then wait for `COMPLETED` (step 7's loop). With TLS off, `REDIS_URL` is a `redis://` URL that `ioredis` reads as-is. By default, new connections go to the instance with the fewest open connections, which suits long-lived sockets. To autoscale instead of using a fixed count, see [autoscale deployments](https://northflank.com/docs/v1/application/scale/autoscale-deployments).

## Optional: Socket.IO

Socket.IO starts with HTTP long-polling before upgrading, and its multi-node docs say all requests for a session must reach the same process unless clients use `transports: ["websocket"]`. Two changes cover it:

1. **Adapter.** Use `@socket.io/redis-adapter`. The Socket.IO docs recommend `ioredis`, because the `redis` package has had trouble restoring subscriptions after reconnects:
   ```js
   import { Server } from 'socket.io';
   import { createAdapter } from '@socket.io/redis-adapter';
   import Redis from 'ioredis';
   const pubClient = new Redis(process.env.REDIS_URL);
   const subClient = pubClient.duplicate();
   const io = new Server(httpServer, { adapter: createAdapter(pubClient, subClient) });
   ```
2. **Sticky sessions.** Northflank's load-balancing docs say consistent hashing on client IP creates sticky sessions:
   ```bash
   northflank patch service combined --projectId "$PROJECT_ID" --serviceId ws --noDefaults -o json \
     --input '{"loadBalancing": {"mode": "consistentHash", "consistentHash": {"mode": "ip"}}}'
   ```
   Many clients behind one NAT will land on the same instance. Alternatively, keep the default strategy and set `transports: ["websocket"]` on every client.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId ws --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The handshake upgrades (prints `101`; `--max-time` ends the open connection):
   ```bash
   curl --http1.1 -s -o /dev/null -w '%{http_code}\n' --max-time 5 \
     -H 'Connection: Upgrade' -H 'Upgrade: websocket' -H 'Sec-WebSocket-Version: 13' \
     -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' "https://$HOST/"
   ```
3. `https://$HOST/healthz` returns 200, and step 8's client prints `got: hello`.
4. With two instances, a message from one client reaches another. Repeat a few times, because connections spread across instances:
   ```bash
   node -e '
   const url = process.argv[1]; const a = new WebSocket(url), b = new WebSocket(url);
   let open = 0; const go = () => { if (++open === 2) a.send("cross-instance"); };
   a.onopen = go; b.onopen = go;
   b.onmessage = (e) => { console.log("b got:", e.data); process.exit(0); };
   setTimeout(() => { console.error("timeout"); process.exit(1); }, 5000);
   ' "wss://$HOST/"
   ```
5. Both instances are running: `northflank get service --projectId "$PROJECT_ID" --serviceId ws --noDefaults -o json | jq '(.data // .).deployment.instances'` prints `2`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Browser: `Mixed Content` or connection refused | Page uses `ws://` | Use `wss://<host>/` |
| Handshake returns 200 or 404, not 101 | Upgrade not handled on that path, or `WebSocketServer` not attached to the HTTP server | Pass `{ server }` to `WebSocketServer` |
| Connections drop after a quiet period | No traffic on idle sockets | Keep the 30-second ping; reconnect with backoff on the client |
| With 2 instances, some clients miss messages | `REDIS_URL` not injected | Check step 9's `linkedKeys`; restart |
| Socket.IO `400 Session ID unknown` | Polling requests hit different instances | Enable consistent hashing or websocket-only transport |
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
- Add authentication to the upgrade handler before you scale or share the URL. The example accepts anyone.
- Don't switch the port to `TCP` to "support WebSockets"; WebSockets need the `HTTP` port type for public access.
- Print `REDIS_URL` only truncated (`cut -c1-8`), never in full.
- Ask before deleting the addon or service. Prefer pausing.

## FAQ

### Where should I host a WebSocket server?
On a platform that keeps a process running and passes the `Upgrade` handshake through its load balancer: a container platform, Kubernetes with an ingress controller, or a VM with nginx configured for upgrades. Serverless functions don't fit long-lived connections.

### Do I need sticky sessions?
Not for plain WebSockets, because each connection stays on the instance that accepted it. You need them for Socket.IO with HTTP long-polling, or for any protocol that spreads one session over several HTTP requests.

### Can the WebSocket server share a port with my HTTP API?
Yes. The upgrade happens on the HTTP port, as in this guide, which serves `/healthz` and WebSockets on 8080. Express, Fastify and NestJS apps can attach `ws` or Socket.IO to their HTTP server the same way.

## Conclusion

You now have a WebSocket server on a `wss://` URL, with heartbeats for idle connections, clean shutdowns on deploys, and Redis pub/sub so messages reach clients on every instance.

**Next:** add authentication to the upgrade, autoscale on connection load, and put your API on the same server with [how to deploy an Express app](deploy-express-app.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy an Express app](deploy-express-app.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy a NestJS app](deploy-nestjs-app.md)
