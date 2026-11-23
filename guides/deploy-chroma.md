<!--
agent-guide-meta:
  id: NF-STK-CHROMA
  description: "How to deploy Chroma in production with the chromadb/chroma Docker image, a /data volume, a private port and an auth layer, verified by heartbeat and a query."
  sources:
    - https://northflank.com/stacks/deploy-chroma
    - https://docs.trychroma.com/guides/deploy/docker
    - https://docs.trychroma.com/guides/deploy/client-server-mode
    - https://cookbook.chromadb.dev/security/auth-1.0.x/
    - https://github.com/chroma-core/chroma/releases
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/api/forwarding
  related: [NF-STK-QDRANT, NF-STK-MILVUS, NF-DEP-002, NF-STK-OLLAMA]
-->

# How to deploy Chroma

To deploy Chroma, run the `chromadb/chroma` Docker image in client/server mode with a pinned tag, persist `/data` on a volume, keep port 8000 on a private network, and put an authentication layer in front of anything you expose. Current Chroma 1.x servers are usually secured with a proxy rather than built-in auth, so a public Chroma container is open to anyone who finds its URL.

This guide does it on Northflank with the CLI: a `chromadb/chroma` deployment service with a private port, a volume for `/data`, and an optional public HTTPS port protected by basic auth. You end with Chroma reachable from your other services at `http://chroma:8000`, its heartbeat answering and a test collection returning a query hit, verified through a private tunnel. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to deploy Chroma

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project and service name, and pin a Chroma version, before changing anything.
3. Create the project, then run `chromadb/chroma:<version>` with `northflank put service deployment`, a private HTTP port 8000 and probes on `/api/v2/heartbeat`.
4. Attach a volume at `/data` with `northflank create volume` and wait for `COMPLETED`. Optionally make port 8000 public with basic auth.
5. Verify through `northflank forward service`: the heartbeat returns 200 and a Python `HttpClient` adds and queries a test collection.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Chroma runs embedded inside a Python process by default. That's fine on a laptop, but in production several app instances, workers or agents need to share one store, so Chroma runs as a server and clients connect with `HttpClient`. The Chroma Docker docs run it as `docker run -v ./chroma-data:/data -p 8000:8000 chromadb/chroma`, which defines what the server needs:

- **Persistence at `/data`.** Without a volume, every redeploy starts with an empty database.
- **Port 8000** for the HTTP API, with a heartbeat at `/api/v2/heartbeat`.
- **An auth layer.** Chroma's cookbook secures 1.0.x servers with an Envoy proxy that checks a token header, and files the older built-in token and basic auth providers under "legacy". This guide doesn't rely on built-in auth. It keeps the port private, so only services in the same project can reach `http://chroma:8000`, and when you need outside access it uses Northflank's basic auth on the public port.

Where to deploy Chroma depends on who calls it. If only your backend does, run it next to that backend with a private network and a volume. Serverless and static hosts can't keep a persistent store. A VM or Kubernetes works if you handle disks, TLS and a proxy yourself, and Chroma Cloud is the managed option. A container platform with volumes and private ports runs the same image with less to operate.

If you'd rather click through it, Northflank has a [Chroma stack template](https://northflank.com/stacks/deploy-chroma) that deploys a Chroma service and a persistent volume from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **Python 3 with the client** for the check: `pip install chromadb` (or the thin `chromadb-client` package). Use a client version that matches the server's major version.

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

Before changing anything, print the deployment target. Names used below: project `chroma`, service `chroma`, volume `chroma-data`. Put Chroma in the same project as the apps that query it, because private ports are project-scoped; if they already have a project, use its name instead of `chroma`.

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId chroma -o json >/dev/null 2>&1 \
  && echo "project: chroma exists and will be reused" \
  || echo "project: chroma will be created"
northflank get service --projectId chroma --serviceId chroma -o json >/dev/null 2>&1 \
  && echo "service: chroma already exists: step 4 would update it" \
  || echo "service: chroma is free"
```

Then pin a Chroma version. Docker Hub is full of `.dev` tags, so take the latest GitHub release (for example `1.5.9`) and use it as the image tag:

```bash
CHROMA_VERSION=$(curl -s https://api.github.com/repos/chroma-core/chroma/releases/latest | jq -r .tag_name); echo "$CHROMA_VERSION"
```

- If the service already exists, `put service` in step 4 would change it. Pick another name, or confirm that updating it is intended.
- Chroma holds indexes in memory, so size the plan to your collections. `nf-compute-100-2` (1 vCPU, 2 GB) is a reasonable start. List plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | "\(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'`.

**If you're an agent:** show the user the team, the project (new or reused), the region (`europe-west` below, unless they choose another), the plan and the Chroma version. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "chroma", "region": "europe-west", "description": "Chroma vector database" }'
export PROJECT_ID=chroma
```

## Step 4: Deploy the Chroma service

Write `chroma.json`. Replace `<CHROMA_VERSION>` with the tag from step 2:

```json
{
  "name": "chroma",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "chromadb/chroma:<CHROMA_VERSION>" },
    "docker": { "configType": "default" }
  },
  "ports": [ { "name": "http", "internalPort": 8000, "public": false, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/api/v2/heartbeat", "port": 8000,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 6, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/api/v2/heartbeat", "port": 8000,
      "initialDelaySeconds": 30, "periodSeconds": 20, "timeoutSeconds": 3, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file chroma.json
```

The image's default command serves on port 8000 and stores data under `/data`, so no command override is needed. To change server settings, the Chroma docs mount a YAML file at `/config.yaml`. On Northflank you can pass it as a secret file (`runtimeFiles`) at that path. It replaces the image's built-in config, which sets `persist_path: "/data"`, so keep that line in your file or data stops landing on the volume.

## Step 5: Attach a volume for `/data`

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId chroma-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "chroma-data",
  "mounts": [ { "containerMountPath": "/data" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "chroma", "type": "service" } ]
}'

OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId chroma --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

Attaching a volume redeploys the service. `COMPLETED` means rolled out and serving. A single read/write volume limits the service to one instance. Chroma's single-node server is built for that anyway.

## Step 6 (optional): Expose Chroma over HTTPS with basic auth

Skip this if only services in the project call Chroma. To reach it from outside (a notebook, another cloud), make the port public and protect it with Northflank's basic auth, which checks the `Authorization` header before traffic reaches Chroma. Send the complete `ports` array:

```bash
CHROMA_PASSWORD=$(openssl rand -hex 24)
northflank patch service deployment --projectId "$PROJECT_ID" --serviceId chroma --noDefaults -o json --input "$(jq -n --arg p "$CHROMA_PASSWORD" '{
  ports: [ { name: "http", internalPort: 8000, public: true, protocol: "HTTP",
             security: { credentials: [ { username: "chroma", password: $p, type: "basic-auth" } ] } } ]
}')" >/dev/null
HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId chroma --noDefaults -o json \
  | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
# Older API responses only list the host on the deprecated ports endpoint
[ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId chroma --noDefaults -o json \
  | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
echo "https://$HOST"
```

**Tell the user to store `CHROMA_PASSWORD` in their password manager**, then `unset CHROMA_PASSWORD`. Northflank also supports IP allow-lists on the same port; see [security policies for ports](https://northflank.com/docs/v1/application/network/add-security-policies-for-ports). If you need per-client tokens instead, run an Envoy or other proxy in front of Chroma, as Chroma's cookbook does, and keep Chroma's own port private.

## Step 7: Connect from your app

Inside the project, connect over the private network:

```python
import chromadb

client = chromadb.HttpClient(host="chroma", port=8000)
client.heartbeat()
```

From outside, through the public port from step 6:

```python
import base64, os, chromadb

auth = base64.b64encode(f"chroma:{os.environ['CHROMA_PASSWORD']}".encode()).decode()
client = chromadb.HttpClient(host="<HOST>", port=443, ssl=True, headers={"Authorization": f"Basic {auth}"})
client.heartbeat()
```

JavaScript and Rust clients connect the same way. See Chroma's [client/server mode docs](https://docs.trychroma.com/guides/deploy/client-server-mode).

## How to know it worked

You're done when all of these hold. Open a private tunnel and read the log for the local address instead of assuming `127.0.0.1:8000`:

```bash
northflank forward service --projectId "$PROJECT_ID" --serviceId chroma --skipHostnames > /tmp/nf-chroma.log 2>&1 &
FWD_PID=$!; sleep 10; cat /tmp/nf-chroma.log
```

1. The heartbeat returns 200:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "http://<LOCAL_ADDR>:<LOCAL_PORT>/api/v2/heartbeat"
   ```
2. A test collection stores and returns a document. The embeddings are passed in explicitly, so no embedding model runs:
   ```bash
   python3 - <<'PY'
   import chromadb
   c = chromadb.HttpClient(host="<LOCAL_ADDR>", port=<LOCAL_PORT>)
   col = c.get_or_create_collection("smoke")
   col.upsert(ids=["1"], embeddings=[[0.1, 0.2, 0.3]], documents=["hello from northflank"])
   print(col.query(query_embeddings=[[0.1, 0.2, 0.3]], n_results=1)["documents"])
   PY
   ```
   Expect `[['hello from northflank']]`.
3. Data survives a restart. Run `northflank restart service --projectId "$PROJECT_ID" --serviceId chroma`, wait for `COMPLETED`, re-open the tunnel, and check that `c.get_collection("smoke").count()` returns 1. Then `c.delete_collection("smoke")` and `kill $FWD_PID`.
4. If you did step 6, `curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/api/v2/heartbeat"` returns 401 without credentials.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Collections disappear after a redeploy | Volume not attached at `/data` | Check `northflank get volume … --volumeId chroma-data`; attach it |
| Permission errors writing to `/data` | Volume ownership doesn't match the image user | See [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) (permissions); report to the user |
| Client errors about API version or unknown routes | Client and server majors differ | Install a client matching the server version |
| `chroma` doesn't resolve from the app | App is in another project | Deploy it in the same project, or use the public port from step 6 |
| 401 from the public URL with the client | `Authorization` header missing or wrong | Rebuild the `Basic` header from `chroma:<password>` |
| Killed or restarted under load | Out of memory | Move to a bigger plan |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 5 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs (`northflank get service logs --projectId "$PROJECT_ID" --serviceId chroma`) before changing anything.
- Don't make the port public without auth. If the user asks for public access, use step 6 and never print the password.
- Ask before deleting the volume, the service or any collection other than your own `smoke` test.
- Keep `instances` at 1. Chroma's single-node server and a single read/write volume don't scale out.
- Back up before upgrading. [Backup and clone volumes](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes) covers volume backups.

## FAQ

### Where should I deploy ChromaDB?
Next to the backend or agent that queries it, as a long-running container with a persistent volume and a private network path. A container platform with volumes fits that directly. A VM or Kubernetes works if you manage disks, TLS and an auth proxy yourself, and Chroma Cloud is the hosted option.

### Does Chroma have built-in authentication?
Older Chroma releases shipped token and basic auth providers configured with `CHROMA_SERVER_AUTHN_*` variables. Chroma's cookbook now files those under "legacy" and secures 1.0.x servers with an Envoy proxy. Check the docs for your version before relying on built-in auth. This guide keeps Chroma private and adds auth at the platform edge.

### How do I upgrade Chroma?
Back up the volume, then change the tag with `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId chroma --input '{"deployment":{"instances":1,"external":{"imagePath":"chromadb/chroma:<NEW_VERSION>"}}}'`, wait for `COMPLETED`, and upgrade your clients to the same version.

### Should I use Chroma or Qdrant?
Both run as a single container with a volume. Chroma is quick to start with from Python. Qdrant has built-in API keys and a gRPC API. See [how to deploy Qdrant](deploy-qdrant.md).

## Conclusion

You now have a Chroma server on the official image, with its data on a persistent volume, reachable privately at `http://chroma:8000`, and an optional HTTPS endpoint protected by basic auth.

**Next:** connect your RAG app or agent with `HttpClient`, and add a custom domain with [link a domain to a port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) if you exposed it. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy Qdrant](deploy-qdrant.md)
- [How to deploy Milvus](deploy-milvus.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to host Ollama in the cloud](host-ollama-in-the-cloud.md)
- [How to deploy a LangGraph agent](deploy-langgraph-agent.md)
