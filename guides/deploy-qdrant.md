<!--
agent-guide-meta:
  id: NF-STK-QDRANT
  description: "How to deploy Qdrant in production with the official Docker image, an API key, a /qdrant/storage volume and private REST and gRPC ports, verified end to end."
  sources:
    - https://qdrant.tech/documentation/installation/
    - https://qdrant.tech/documentation/security/
    - https://qdrant.tech/documentation/ops-monitoring/monitoring/
    - https://hub.docker.com/r/qdrant/qdrant
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/networking-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
    - https://northflank.com/docs/v1/api/forwarding
  related: [NF-STK-CHROMA, NF-STK-MILVUS, NF-DEP-002, NF-STK-OLLAMA]
-->

# How to deploy Qdrant

To deploy Qdrant, run the official `qdrant/qdrant` Docker image with a pinned tag, set `QDRANT__SERVICE__API_KEY`, persist `/qdrant/storage` on block storage, and expose the REST API on port 6333 and gRPC on port 6334 only to the services that need them. Qdrant starts with no authentication and listens on all interfaces, so an exposed default container is open to anyone who finds it.

This guide does it on Northflank with the CLI: a secret group holding a generated API key, a `qdrant/qdrant` deployment service with private REST and gRPC ports, and a volume for the data. You end with Qdrant reachable from your other services at `http://qdrant:6333`, rejecting requests without the key and answering a test search, verified through a private tunnel. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to deploy Qdrant

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project and service name, and pin a Qdrant version, before changing anything.
3. Create the project, a secret group with a generated `QDRANT__SERVICE__API_KEY`, and a `qdrant/qdrant:<version>` deployment service with private ports 6333 (HTTP) and 6334 (HTTP/2) and probes on `/readyz` and `/livez`.
4. Attach a volume at `/qdrant/storage` with `northflank create volume` and wait for `COMPLETED`. Optionally make port 6333 public.
5. Verify through `northflank forward service`: `/readyz` returns 200, requests without the key get 401, and a test collection returns a search hit.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Qdrant's security docs say a self-hosted instance is not secure by default: no API key, no TLS, listening on every interface. Three choices fix that:

- **An API key** (`QDRANT__SERVICE__API_KEY`). Clients send it in the `api-key` header. An optional read-only key (`QDRANT__SERVICE__READ_ONLY_API_KEY`) suits services that only search.
- **Private ports by default.** On Northflank, a port with `public: false` is only reachable from the same project at `[service-name]:[port-number]`. Your app talks to `http://qdrant:6333` and nothing on the internet can.
- **A block-storage volume.** Qdrant's install docs say it needs block-level access with a POSIX file system and won't work on network file systems such as NFS. Northflank volumes are attached as single read/write block volumes.

Where to deploy Qdrant comes down to those needs: a long-running process with a few GB of memory, persistent block storage and private networking to your app. Serverless and static hosts can't run it. A VM or your own Kubernetes cluster works if you manage disks, TLS and upgrades yourself. Qdrant Cloud is the managed option. A container platform with volumes and private networking runs the same image with less to operate.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl` and `curl`**, to generate the API key and run the checks.

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

Before changing anything, print the deployment target. Names used below: project `qdrant`, secret group `qdrant-env`, service `qdrant`, volume `qdrant-data`. Deploy Qdrant in the same project as the apps that query it, because private ports are project-scoped; if they already have a project, use its name instead of `qdrant`.

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId qdrant -o json >/dev/null 2>&1 \
  && echo "project: qdrant exists and will be reused" \
  || echo "project: qdrant will be created"
northflank get service --projectId qdrant --serviceId qdrant -o json >/dev/null 2>&1 \
  && echo "service: qdrant already exists: step 5 would update it" \
  || echo "service: qdrant is free"
```

Then pin a Qdrant version. This prints the newest release tag on Docker Hub (for example `v1.20.2`):

```bash
QDRANT_VERSION=$(curl -s "https://hub.docker.com/v2/repositories/qdrant/qdrant/tags?page_size=100&ordering=last_updated" \
  | jq -r '.results[].name' | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -n1); echo "$QDRANT_VERSION"
```

- If the service already exists, `put service` in step 5 would change it. Pick another name, or confirm that updating it is intended.
- Qdrant keeps vectors and indexes in memory unless you configure on-disk storage, so size the plan to your data. `nf-compute-100-2` (1 vCPU, 2 GB) is a reasonable start for testing. List plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | "\(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'`.

**If you're an agent:** show the user the team, the project (new or reused), the region (`europe-west` below, unless they choose another), the plan and the Qdrant version. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "qdrant", "region": "europe-west", "description": "Qdrant vector database" }'
export PROJECT_ID=qdrant
```

## Step 4: Create the API keys

Generate an admin key and a read-only key, and store them in one secret group without echoing them. The group also gives client services `QDRANT_URL` and `QDRANT_API_KEY`:

```bash
QDRANT_API_KEY=$(openssl rand -hex 32); QDRANT_RO_KEY=$(openssl rand -hex 32)
jq -n --arg k "$QDRANT_API_KEY" --arg ro "$QDRANT_RO_KEY" '{
  name: "qdrant-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  secrets: { variables: {
    QDRANT__SERVICE__API_KEY: $k, QDRANT__SERVICE__READ_ONLY_API_KEY: $ro,
    QDRANT_API_KEY: $k, QDRANT_URL: "http://qdrant:6333"
  } }
}' > qdrant-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file qdrant-env.json >/dev/null && rm qdrant-env.json
```

Keep `QDRANT_API_KEY` in your shell for the checks below. The group is unrestricted, so every service in the project can read the key; restrict it with `restrictions.nfObjects` once you know which services need it. Qdrant's docs note that changing `api_key` invalidates any JWTs issued with it.

## Step 5: Deploy the Qdrant service

Write `qdrant.json`. Replace `<QDRANT_VERSION>` with the tag from step 2:

```json
{
  "name": "qdrant",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "qdrant/qdrant:<QDRANT_VERSION>" },
    "docker": { "configType": "default" }
  },
  "ports": [
    { "name": "http", "internalPort": 6333, "public": false, "protocol": "HTTP" },
    { "name": "grpc", "internalPort": 6334, "public": false, "protocol": "HTTP/2" }
  ],
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/readyz", "port": 6333,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 6, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/livez", "port": 6333,
      "initialDelaySeconds": 30, "periodSeconds": 20, "timeoutSeconds": 3, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file qdrant.json
```

Qdrant's monitoring docs say `/healthz`, `/livez` and `/readyz` are always reachable, whether or not an API key is set, so the probes don't need the key. Port 6335 is only for distributed (cluster) mode and stays closed here. If gRPC between services fails on the `HTTP/2` port, change its protocol to `TCP`; the networking docs say internal ports support TCP.

## Step 6: Attach a volume for `/qdrant/storage`

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId qdrant-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "qdrant-data",
  "mounts": [ { "containerMountPath": "/qdrant/storage" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "qdrant", "type": "service" } ]
}'
```

Attaching redeploys the service. A single read/write volume limits the service to one instance, and on restart the old container stops before the new one starts, so expect a short gap. Volume storage can be increased later, never reduced.

Then wait for the rollout:

```bash
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId qdrant --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
```

`COMPLETED` means rolled out and serving.

## Step 7 (optional): Expose the REST API over HTTPS

Skip this if only services in the project use Qdrant. To let an external client (a laptop, another cloud) connect, make port 6333 public. Northflank terminates TLS on 443, which covers Qdrant's advice to use TLS whenever an API key is sent. Send the complete `ports` array:

```bash
northflank patch service deployment --projectId "$PROJECT_ID" --serviceId qdrant --noDefaults -o json --input '{
  "ports": [
    { "name": "http", "internalPort": 6333, "public": true,  "protocol": "HTTP" },
    { "name": "grpc", "internalPort": 6334, "public": false, "protocol": "HTTP/2" }
  ]
}'
HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId qdrant --noDefaults -o json \
  | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
# Older API responses only list the host on the deprecated ports endpoint
[ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId qdrant --noDefaults -o json \
  | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
echo "https://$HOST"
```

The health endpoints stay open without a key, so anyone can see that Qdrant is up, but not read data. To narrow access further, add an IP allow-list with [security policies for ports](https://northflank.com/docs/v1/application/network/add-security-policies-for-ports).

## Step 8: Connect from your app

Inside the project, link the `qdrant-env` group (or copy `QDRANT_URL` and `QDRANT_API_KEY`) and use the official client. Python:

```python
import os
from qdrant_client import QdrantClient

client = QdrantClient(url=os.environ["QDRANT_URL"], api_key=os.environ["QDRANT_API_KEY"])
print(client.get_collections())
```

Traffic inside the project is plain HTTP, so the client may warn that the key is sent over an insecure connection. From outside, use `url=f"https://{HOST}:443"`. For gRPC inside the project, pass `prefer_grpc=True` with `grpc_port=6334`.

## How to know it worked

You're done when all of these hold. Open a private tunnel first and read the log for the local address instead of assuming `127.0.0.1:6333`:

```bash
northflank forward service --projectId "$PROJECT_ID" --serviceId qdrant --skipHostnames > /tmp/nf-qdrant.log 2>&1 &
FWD_PID=$!; sleep 10; cat /tmp/nf-qdrant.log
Q=http://<LOCAL_ADDR>:<LOCAL_PORT>    # the address forwarded to port 6333
```

1. Readiness returns 200:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "$Q/readyz"
   ```
2. Requests without the key are rejected (401) and requests with it succeed (200):
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "$Q/collections"
   curl -s -o /dev/null -w '%{http_code}\n' -H "api-key: $QDRANT_API_KEY" "$Q/collections"
   ```
3. A test collection stores and returns a vector:
   ```bash
   curl -s -X PUT "$Q/collections/smoke" -H "api-key: $QDRANT_API_KEY" -H 'Content-Type: application/json' \
     -d '{"vectors": {"size": 4, "distance": "Cosine"}}' | jq -r .status
   curl -s -X PUT "$Q/collections/smoke/points?wait=true" -H "api-key: $QDRANT_API_KEY" -H 'Content-Type: application/json' \
     -d '{"points": [{"id": 1, "vector": [0.1, 0.2, 0.3, 0.4], "payload": {"ok": true}}]}' | jq -r .status
   curl -s -X POST "$Q/collections/smoke/points/query" -H "api-key: $QDRANT_API_KEY" -H 'Content-Type: application/json' \
     -d '{"query": [0.1, 0.2, 0.3, 0.4], "limit": 1, "with_payload": true}' | jq '.result.points[0].payload'
   ```
   Expect `ok`, `ok` and `{"ok": true}`.
4. Data survives a restart. Run `northflank restart service --projectId "$PROJECT_ID" --serviceId qdrant`, wait for `COMPLETED`, re-open the tunnel and check that `GET $Q/collections/smoke` still returns 200. Then clean up with `curl -X DELETE "$Q/collections/smoke" -H "api-key: $QDRANT_API_KEY"` and `kill $FWD_PID`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `/collections` returns 200 without a key | `qdrant-env` not injected | Check the group exists and isn't restricted away from `qdrant`; restart |
| Client gets 401 or 403 | Wrong key, or read-only key used for writes | Use `QDRANT_API_KEY` for writes |
| Container restarts with permission errors on `/qdrant/storage` | Volume ownership doesn't match the image user | See [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) (permissions); report to the user |
| Killed or restarted under load | Out of memory | Move to a bigger plan, or enable on-disk vectors in the collection config |
| App can't resolve `qdrant` | App is in another project | Deploy it in the same project, or make port 6333 public (step 7) |
| gRPC calls fail inside the project | HTTP/2 port handling | Set the 6334 port's protocol to `TCP` |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 6 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs (`northflank get service logs --projectId "$PROJECT_ID" --serviceId qdrant`) before changing anything.
- Never print `QDRANT__SERVICE__API_KEY` or the read-only key; build JSON with `jq --arg` and delete temp files.
- Don't make a port public unless the user asks for access from outside the project.
- Ask before deleting the volume, the service or any collection that isn't your own `smoke` test. Take a snapshot (`POST /collections/{name}/snapshots`) before upgrading Qdrant.
- Keep `instances` at 1 with this volume. Multi-node Qdrant needs cluster mode and port 6335, which this guide doesn't cover.

## FAQ

### Where should I deploy Qdrant?
Next to the app that queries it, on a host that keeps a long-running container up with block storage and private networking. A container platform with volumes fits that with the least work. A VM or Kubernetes works if you manage disks and TLS yourself, and Qdrant Cloud is the fully managed option.

### Do I need the gRPC port?
No. The REST API on 6333 covers everything. gRPC on 6334 is faster for bulk uploads, and the official clients can use it with `prefer_grpc=True`.

### How do I upgrade Qdrant?
Snapshot your collections, then change the tag with `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId qdrant --input '{"deployment":{"instances":1,"external":{"imagePath":"qdrant/qdrant:<NEW_VERSION>"}}}'` and wait for `COMPLETED`. Read Qdrant's release notes before skipping minor versions.

### Can I scale Qdrant to more instances?
Not with this setup. A single read/write volume allows one instance. Scale up the plan first; horizontal scaling needs Qdrant's distributed mode.

## Conclusion

You now have Qdrant on the official image, with an API key, data on a persistent volume, and REST and gRPC reachable only inside your project unless you chose to open REST over HTTPS.

**Next:** point your RAG or agent service at `QDRANT_URL`, and serve embeddings next to it with [how to host Ollama in the cloud](host-ollama-in-the-cloud.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy Chroma](deploy-chroma.md)
- [How to deploy Milvus](deploy-milvus.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to host Ollama in the cloud](host-ollama-in-the-cloud.md)
- [How to deploy a LangGraph agent](deploy-langgraph-agent.md)
