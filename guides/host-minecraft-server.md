<!--
agent-guide-meta:
  id: NF-STK-MINECRAFT
  description: "How to host a Minecraft server (Java Edition) with the itzg/minecraft-server image, EULA=TRUE, a /data world volume and a public TCP load balancer on 25565."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://docker-minecraft-server.readthedocs.io/en/latest/
    - https://docker-minecraft-server.readthedocs.io/en/latest/versions/java/
    - https://docker-minecraft-server.readthedocs.io/en/latest/configuration/jvm-options/
    - https://docker-minecraft-server.readthedocs.io/en/latest/configuration/server-properties/
    - https://docker-minecraft-server.readthedocs.io/en/latest/sending-commands/commands/
    - https://hub.docker.com/r/itzg/minecraft-server
    - https://github.com/itzg/docker-minecraft-bedrock-server
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/configure-load-balancers
    - https://northflank.com/docs/v1/application/network/networking-on-northflank
    - https://northflank.com/docs/v1/api/team/load-balancers/put-load-balancer
    - https://northflank.com/docs/v1/api/team/load-balancers/get-load-balancer
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/volumes/backup-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DEP-002, NF-STK-DISCORD, NF-STK-N8N, NF-DEP-001]
-->

# How to host a Minecraft server

To host a Minecraft server, run the `itzg/minecraft-server` Docker image with `EULA=TRUE`, give the JVM a heap (`MEMORY`) about 25% smaller than the container's memory limit, keep `/data` on a persistent volume so the world survives restarts, and expose TCP port 25565 on a stable public IP. Minecraft speaks raw TCP, not HTTP, so a normal web URL or HTTPS proxy won't let players join.

This guide does it on Northflank with the CLI: a service running a pinned `itzg/minecraft-server` tag, a volume for `/data`, and a Layer 4 TCP load balancer that gives it a public IP. You end with a whitelisted Java Edition server that friends join at `<IP>:25565`, checked with `nc`, `mc-health` and `rcon-cli`. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run; a human accepts the EULA and joins from the game.

## TL;DR: how to host a Minecraft server

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project, image tag and load balancer access, then create the `minecraft` project.
3. Create a service from `itzg/minecraft-server:<tag>` with `EULA=TRUE`, `MEMORY=3G` on a 4 GB plan, a private TCP port 25565 and zero instances, then attach a volume at `/data` and scale to one instance.
4. Wait for `COMPLETED` and for the world to finish generating.
5. Create a public TCP load balancer on port 25565 pointing at the service with `northflank put load-balancer`, and read its IP.
6. Verify with `nc -vz <IP> 25565` and `rcon-cli list`, then join from the game.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

A Minecraft server is one long-running Java process that keeps the world in memory and saves it to disk. That shapes the setup:

- **One instance.** Two servers can't share a world, so this is never scaled out.
- **A persistent volume.** The world, `server.properties`, whitelist and ops list live in `/data`. Without a volume, every restart makes a new world.
- **Memory headroom.** The itzg docs say `MEMORY` only sets the Java heap; the container needs about 25% more for non-heap memory.
- **Raw TCP on a public IP.** Players connect to `host:25565` over TCP. Northflank's public service ports carry only HTTP and HTTP/2; the [configure ports](https://northflank.com/docs/v1/application/network/configure-ports) docs say TCP and UDP service ports are private. Internet TCP traffic goes through a separate [Layer 4 load balancer](https://northflank.com/docs/v1/application/network/configure-load-balancers) with a stable public IP, which step 7 creates. A `*.code.run` HTTPS URL won't work for Minecraft.

So where should you host a Minecraft server? Your own PC works for a LAN, but friends outside need port forwarding and the world is up only while the PC is. A VM runs anything, but you handle Java, backups and restarts. A container platform runs the same Docker image you'd use locally, with a volume, health checks and a public TCP endpoint.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The players' Minecraft usernames** for the whitelist, and who should be an operator.
- **Acceptance of the [Minecraft EULA](https://www.minecraft.net/en-us/eula)** by the person running the server.

Names used below: project `minecraft`, service `minecraft`, volume `minecraft-data`, load balancer `minecraft-lb`.

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

Before changing anything, pin an image tag and print the deployment target. The itzg image publishes dated tags per Java version (`latest` runs Java 25); this picks the newest dated Java 25 tag. Load balancers are team-level resources, so the last line checks that your login can use them (an API token needs the `Account > Networking > LoadBalancers` create and read permissions):

```bash
MC_TAG=$(curl -s "https://hub.docker.com/v2/repositories/itzg/minecraft-server/tags?page_size=100&ordering=last_updated" \
  | jq -r '.results[].name' | grep -E '^[0-9]{4}\.[0-9]+\.[0-9]+-java25$' | sort -V | tail -n1)
echo "image:   itzg/minecraft-server:$MC_TAG"
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId minecraft -o json >/dev/null 2>&1 \
  && echo "project: minecraft exists and will be reused" \
  || echo "project: minecraft will be created"
northflank list load-balancers -o json >/dev/null && echo "load balancers: OK" || echo "load balancers: NOT ALLOWED for this login"
```

If the image shows no tag or load balancers aren't allowed, fix that first: without a load balancer, players can't reach the server.

**If you're an agent:** show the user the team, the image tag, the region (`europe-west` below, unless they choose another) and whether the project is new or reused, and confirm they accept the Minecraft EULA. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your players (`northflank list regions` lists them); it can't be changed later, and the load balancer in step 7 must be in the same region. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "minecraft", "region": "europe-west" }'
export PROJECT_ID=minecraft
export REGION=$(northflank get project --projectId "$PROJECT_ID" --noDefaults -o json | jq -r '(.data // .).region'); echo "$REGION"
```

The last line reads the project's actual region, so it's right for a reused project too.

## Step 4: Create the server with zero instances

Starting at zero instances means the first world is generated on the volume, not on a throwaway filesystem. Write `minecraft.json`, replacing `<MC_TAG>` with the tag step 2 printed, and the usernames:

```json
{
  "name": "minecraft",
  "billing": { "deploymentPlan": "nf-compute-200-4" },
  "deployment": {
    "instances": 0,
    "external": { "imagePath": "itzg/minecraft-server:<MC_TAG>" },
    "docker": { "configType": "default" }
  },
  "ports": [ { "name": "game", "internalPort": 25565, "public": false, "protocol": "TCP" } ],
  "runtimeEnvironment": {
    "EULA": "TRUE", "TYPE": "PAPER", "VERSION": "LATEST", "MEMORY": "3G",
    "ONLINE_MODE": "true", "ENABLE_WHITELIST": "true",
    "WHITELIST": "<player1>,<player2>", "OPS": "<player1>"
  },
  "healthChecks": [
    { "protocol": "TCP", "type": "startupProbe", "port": 25565,
      "initialDelaySeconds": 30, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 60, "successThreshold": 1 },
    { "protocol": "CMD", "type": "livenessProbe", "cmd": "mc-health",
      "initialDelaySeconds": 10, "periodSeconds": 30, "timeoutSeconds": 10, "failureThreshold": 5 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file minecraft.json
```

What each setting does:

- **`EULA=TRUE`** accepts the Minecraft EULA. The server refuses to start without it. **This needs a human:** confirm the user has read and accepts the [Minecraft EULA](https://www.minecraft.net/en-us/eula) before setting it.
- **`TYPE`** picks the server software (`VANILLA` is the default; `PAPER`, `FABRIC`, `FORGE` and others exist). **`VERSION=LATEST`** upgrades on every restart, so once the world matters, pin `VERSION` to the release it's running now (shown in the startup logs, such as `26.3`); never pin an older version than the world was last saved with.
- **`MEMORY=3G` on a 4 GB plan** leaves about 25% for non-heap memory. Keep that ratio: on `nf-compute-200-8` use `6G`, on `nf-compute-200-16` use `12G`.
- **The port is TCP and private**; the load balancer reaches it. **The startup probe** allows 10 minutes for the first world to generate before `mc-health` (shipped in the image) takes over as the liveness check.

## Step 5: Attach the world volume and start the server

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId minecraft-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "minecraft-data",
  "mounts": [ { "containerMountPath": "/data" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "minecraft", "type": "service" } ]
}'

northflank patch service deployment --projectId "$PROJECT_ID" --serviceId minecraft --noDefaults -o json \
  --input '{"deployment": {"instances": 1}}'
```

`create volume` returns 409 if the volume exists, hence the `get` guard. 10 GB suits a vanilla world; volumes can grow but never shrink. A `ReadWriteOnce` volume keeps the service at one instance, which Minecraft needs anyway.

## Step 6: Wait for the server to be ready

```bash
OK=0
for i in $(seq 1 90); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId minecraft --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
[ "$OK" = 1 ] && northflank get service logs --projectId "$PROJECT_ID" --serviceId minecraft --lineLimit 50 --textIncludes "Done"
```

`COMPLETED` means the instance is running and passed its startup probe. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId minecraft`. The log line `Done (…)! For help, type "help"` means the world is loaded.

## Step 7: Give the server a public IP on port 25565

Write `minecraft-lb.json`. The backend `id` is `<projectId>/<serviceId>`, and `<REGION>` is the project's region that step 3 printed:

```json
{
  "name": "minecraft-lb",
  "description": "Public TCP endpoint for the Minecraft server",
  "spec": {
    "type": "tcp",
    "visibility": "public",
    "target": { "type": "region", "targetId": "<REGION>" },
    "ports": [
      { "id": "port-25565", "port": "25565",
        "backends": [ { "id": "minecraft/minecraft", "type": "service", "port": 25565, "weight": 100 } ] }
    ]
  }
}
```

```bash
northflank put load-balancer --noDefaults -o json --file minecraft-lb.json

OK=0
for i in $(seq 1 60); do
  LB=$(northflank get load-balancer --loadBalancerId minecraft-lb --noDefaults -o json)
  S=$(echo "$LB" | jq -r '(.data // .).state.status // "NONE"')
  echo "load-balancer=$S"
  [ "$S" = "created" ] && { OK=1; break; }
  [ "$S" = "error" ] && echo "LB_ERROR (the platform retries automatically)"
  sleep 10
done
[ "$OK" = 1 ] || { echo "LOAD_BALANCER_NOT_CREATED: stop here and check the load balancer in the UI"; false; }
[ "$OK" = 1 ] && MC_IP=$(echo "$LB" | jq -r '(.data // .).state.endpoints[0]') && echo "$MC_IP"
```

Provisioning takes a few minutes. Ports 30000–32767 are reserved. Updating a load balancer re-provisions it, with a short interruption.

Players connect to `$MC_IP` (port 25565 is the default, so `:25565` is optional). To use a name instead, **a human creates a DNS A record** such as `mc.example.com → $MC_IP` at their DNS provider.

## How to know it worked

You're done when all of these hold:

1. The server answers its own status ping and RCON. `exec` needs `script` to keep output without a TTY (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId minecraft \
     --shell-cmd 'sh -c' --cmd '"mc-health && rcon-cli list"'
   ```
   `mc-health` exits 0 only if the server answers a status ping, so `rcon-cli` runs only then and prints `There are 0 of a max of 20 players online`.
2. Port 25565 is reachable from the internet:
   ```bash
   nc -vz -w 5 "$MC_IP" 25565    # "succeeded" or "open"
   ```
3. The world is on the volume:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId minecraft \
     --shell-cmd 'sh -c' --cmd '"ls /data"'
   ```
   Expect `world`, `server.properties` and `whitelist.json` in the listing.
4. **This step needs a human:** in Minecraft Java Edition, open Multiplayer → Add Server, enter `$MC_IP`, and join with a whitelisted account.

## Bedrock Edition

`itzg/minecraft-server` is Java Edition only. Bedrock uses `itzg/minecraft-bedrock-server`, and since Bedrock Dedicated Server 1.26.50 its default NetherNet transport needs TCP 19132 for signalling plus a UDP port range for gameplay. A Northflank load balancer carries either TCP or UDP, so that means two load balancers with different IPs; this hasn't been tested. The older `TRANSPORT=raknet` mode used only UDP 19132, but the itzg README notes that BDS 1.26.51 logs that NetherNet is the only supported transport, so don't plan around RakNet.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Logs say you need to agree to the EULA | `EULA` missing or not `TRUE` | Set `EULA=TRUE` (after the user accepts it) and restart |
| Container restarts, `OOMKilled` or exit code 137 | `MEMORY` too close to the plan's RAM | Keep `MEMORY` at about 75% of the plan, or pick a bigger plan |
| New world after every restart | Volume not attached at `/data` | Check `attachedObjects` with `northflank get volume`; redo step 5 |
| `Connection timed out` in the client | Load balancer not `created`, wrong region, or wrong backend `id` | Re-check step 7; the backend must be `minecraft/minecraft` on port 25565 |
| `You are not white-listed on this server` | Username missing from `WHITELIST` | Add it, or run `rcon-cli whitelist add <name>` via `exec` |
| Startup probe fails on first boot | World generation took longer than 10 minutes | Raise `failureThreshold` on the startup probe |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 6 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Don't set `EULA=TRUE` until the user has confirmed they accept the Minecraft EULA.
- Never turn off `ONLINE_MODE` or the whitelist on a public IP without the user asking; anyone who finds the IP could join.
- Before a version change, a `TYPE` change or deleting anything, back up the world: `northflank backup volume --projectId "$PROJECT_ID" --volumeId minecraft-data --input '{"name": "before-upgrade"}'`. Before a planned restart, save it: `rcon-cli save-all` via `exec`.
- Don't delete the volume, service or load balancer without explicit, per-resource confirmation. Deleting the volume deletes the world. Deleting the load balancer changes the IP players use.

## FAQ

### Where should I host a Minecraft server?
Somewhere that keeps one Java process running, keeps the world on persistent storage and gives it a public TCP port. Your own PC needs port forwarding and is up only while it's on. A container platform with volumes and a TCP load balancer, as in this guide, runs the same Docker image without you managing a machine.

### How much RAM does a Minecraft server need?
For vanilla or Paper with a few players, 4 GB with a 3 GB heap is a reasonable start; modpacks often need 6–12 GB of heap. Set `MEMORY` to about 75% of the plan's memory.

### Can I run mods or a CurseForge modpack?
Yes. Set `TYPE` to `FABRIC`, `FORGE` or `NEOFORGE` (with mods in `/data/mods`), or use one of the itzg modpack platforms such as `AUTO_CURSEFORGE`. Give modded servers a bigger plan and a longer startup probe.

## Conclusion

You now have a Java Edition server on the `itzg/minecraft-server` image, with the EULA accepted, a heap sized to the plan, the world on a persistent volume, a whitelist, and a public IP on TCP port 25565 from a Layer 4 load balancer.

**Next:** schedule volume backups (see [backup and clone volumes](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes)) and pin `VERSION` once the world matters. If your friends also use Discord, [host a Discord bot](host-discord-bot.md) next to the server. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy a Docker image](deploy-docker-image.md)
- [How to host a Discord bot](host-discord-bot.md)
- [How to self-host n8n](self-host-n8n.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
