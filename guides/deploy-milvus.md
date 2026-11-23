<!--
agent-guide-meta:
  id: NF-STK-MILVUS
  description: "How to deploy Milvus standalone with the official image, embedded etcd, local storage on a volume and authentication on, verified with a private test search."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://milvus.io/docs/install_standalone-docker.md
    - https://milvus.io/docs/prerequisite-docker.md
    - https://milvus.io/docs/authenticate.md
    - https://raw.githubusercontent.com/milvus-io/milvus/master/scripts/standalone_embed.sh
    - https://github.com/milvus-io/milvus/releases/download/v3.0.6/milvus-standalone-docker-compose.yaml
    - https://raw.githubusercontent.com/milvus-io/milvus/v3.0.6/configs/milvus.yaml
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/secure/upload-secret-files
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-minio-on-northflank
    - https://northflank.com/docs/v1/api/forwarding
  related: [NF-STK-QDRANT, NF-STK-CHROMA, NF-STK-MINIO, NF-DEP-002]
-->

# How to deploy Milvus

To deploy Milvus, run the `milvusdb/milvus` image in standalone mode (`milvus run standalone`) with its two dependencies: etcd for metadata and object storage for vector data. The official Docker script embeds etcd in the process (`ETCD_USE_EMBED=true`) and stores data on local disk (`COMMON_STORAGETYPE=local`), so one container with a volume at `/var/lib/milvus` is enough. Authentication stays off until you enable it.

This guide does it on Northflank with the CLI, following the official embedded-etcd script: one `milvusdb/milvus` service with a volume, authentication on with a generated root password, and private ports 19530 and 9091. You end with Milvus at `http://milvus:19530` inside your project, verified by a health check and an authenticated search through a private tunnel. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; Milvus needs an 8 GB plan.

## TL;DR: how to deploy Milvus

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, the project, the Milvus version and an 8 GB plan before changing anything.
3. Create a project and a secret group with a generated root password, then run `milvusdb/milvus` with `northflank put service deployment`: `milvus run standalone`, embedded etcd, private ports 19530 and 9091, and probes on `/healthz`.
4. Attach a volume at `/var/lib/milvus` and wait for the rollout.
5. Verify through `northflank forward service`: `/healthz` returns `OK`, an unauthenticated call is refused, and a test collection returns a search hit that survives a restart.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## What Milvus standalone actually needs

Milvus comes in three forms: Milvus Lite (a Python library), standalone (one server process) and distributed (many components on Kubernetes). Standalone is the right size for most single-team workloads. It always needs:

- **A metadata store (etcd).** Either embedded in the Milvus process (`ETCD_USE_EMBED=true`, data in `ETCD_DATA_DIR`) or a separate etcd server (`ETCD_ENDPOINTS`).
- **Object storage for segments and indexes.** Either local disk (`COMMON_STORAGETYPE=local`, under `/var/lib/milvus`) or MinIO/S3 (`COMMON_STORAGETYPE=remote` plus `MINIO_*` settings).
- **A local data directory** at `/var/lib/milvus` for the write-ahead log and caches, in both layouts.

The v3.0.6 `standalone_embed.sh` script picks embedded etcd and local storage, with one volume. The Compose file runs three containers (etcd, MinIO and Milvus). This guide follows the script and covers the Compose layout as an option at the end.

Milvus's hardware page asks for at least 8 GB RAM for standalone (16 GB recommended), SSD storage, and a CPU with SSE4.2, AVX, AVX2 or AVX-512.

Where to deploy Milvus depends on scale. Standalone runs anywhere that keeps one large container up with an SSD volume and private networking. Serverless hosts can't. Distributed Milvus needs Kubernetes, and Zilliz Cloud is the managed option. For standalone, a container platform with volumes and private ports runs the official image without a cluster.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`, `base64` and Python 3 with `pymilvus`** (`pip install pymilvus`) for the password and the checks.

Names used below: project `milvus`, secret group `milvus-env`, service `milvus`, volume `milvus-data`.

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

Before changing anything, print the deployment target, pin a Milvus version and check that 8 GB plans exist:

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId milvus -o json >/dev/null 2>&1 \
  && echo "project: milvus exists and will be reused" \
  || echo "project: milvus will be created"
MILVUS_VERSION=$(curl -s https://api.github.com/repos/milvus-io/milvus/releases/latest | jq -r .tag_name); echo "milvus:  $MILVUS_VERSION"
curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | select(.ramResource >= 8192) | "\(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'
```

- The release tag doubles as the image tag (for example `v3.0.6`). Keep `MILVUS_VERSION` in your shell for step 5.
- `nf-compute-200-8` (2 vCPU, 8 GB) meets Milvus's minimum and `nf-compute-400-16` matches its recommendation. If neither is listed, stop: Milvus won't run on a smaller plan.

**If you're an agent:** show the user the team, the region (`europe-west` below, unless they choose another), whether the project is new or reused, the Milvus version and the plan. Wait for them to confirm before you create anything.

## Step 3: Create a project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "milvus", "region": "europe-west" }'
export PROJECT_ID=milvus
```

## Step 4: Create the root password

```bash
MILVUS_ROOT_PASSWORD=$(openssl rand -hex 24)
jq -n --arg p "$MILVUS_ROOT_PASSWORD" '{
  name: "milvus-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  secrets: { variables: {
    COMMON_SECURITY_DEFAULTROOTPASSWORD: $p,
    MILVUS_URI: "http://milvus:19530", MILVUS_TOKEN: ("root:" + $p)
  } }
}' > milvus-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file milvus-env.json >/dev/null && rm milvus-env.json
```

`common.security.defaultRootPassword` sets the `root` password only when Milvus initialises, which replaces the documented default `Milvus`. Milvus reads any config key from an environment variable with the dots replaced by underscores; the official script sets `COMMON_STORAGETYPE` the same way. `MILVUS_URI` and `MILVUS_TOKEN` are for client services in the project. Keep `MILVUS_ROOT_PASSWORD` in your shell for the checks. **Tell the user to save it in their password manager.**

## Step 5: Deploy the Milvus service

The official script writes an `embedEtcd.yaml` for the embedded etcd. Encode the same file for a runtime file:

```bash
EMBED_ETCD_B64=$(printf '%s\n' \
  'listen-client-urls: http://0.0.0.0:2379' \
  'advertise-client-urls: http://0.0.0.0:2379' \
  'quota-backend-bytes: 4294967296' \
  'auto-compaction-mode: revision' \
  "auto-compaction-retention: '1000'" | base64 | tr -d '\n')
```

Write `milvus.json`, replacing `<MILVUS_VERSION>` with the version from step 2 and `<EMBED_ETCD_B64>` with the value above:

```json
{
  "name": "milvus",
  "billing": { "deploymentPlan": "nf-compute-200-8" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "milvusdb/milvus:<MILVUS_VERSION>" },
    "docker": { "configType": "customCommand", "customCommand": "milvus run standalone" }
  },
  "ports": [
    { "name": "milvus",  "internalPort": 19530, "public": false, "protocol": "TCP" },
    { "name": "metrics", "internalPort": 9091,  "public": false, "protocol": "HTTP" }
  ],
  "runtimeEnvironment": {
    "DEPLOY_MODE": "STANDALONE",
    "ETCD_USE_EMBED": "true",
    "ETCD_DATA_DIR": "/var/lib/milvus/etcd",
    "ETCD_CONFIG_PATH": "/milvus/etcd-config/embedEtcd.yaml",
    "COMMON_STORAGETYPE": "local",
    "COMMON_SECURITY_AUTHORIZATIONENABLED": "true"
  },
  "runtimeFiles": { "/milvus/etcd-config/embedEtcd.yaml": { "data": "<EMBED_ETCD_B64>", "encoding": "utf-8" } },
  "healthChecks": [
    { "protocol": "HTTP", "type": "startupProbe", "path": "/healthz", "port": 9091,
      "initialDelaySeconds": 30, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 30, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/healthz", "port": 9091,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 5, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/healthz", "port": 9091,
      "initialDelaySeconds": 30, "periodSeconds": 30, "timeoutSeconds": 20, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file milvus.json
```

The environment matches `standalone_embed.sh`, except the config file sits in its own folder (`/milvus/etcd-config`) so the mount can't hide `/milvus/configs/milvus.yaml`. Port 19530 serves both the gRPC SDKs and the RESTful API. Port 9091 serves `/healthz`, metrics and the WebUI, and both stay private. The startup probe allows about five minutes for the first start, in line with the script's 90-second start period plus retries.

## Step 6: Attach a volume and wait for the rollout

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId milvus-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "milvus-data",
  "mounts": [ { "containerMountPath": "/var/lib/milvus" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 20480 },
  "attachedObjects": [ { "id": "milvus", "type": "service" } ]
}'

OK=0
for i in $(seq 1 90); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId milvus --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

If it prints `ROLLOUT_NOT_COMPLETE`, read the logs with `northflank get service logs --projectId "$PROJECT_ID" --serviceId milvus`; the troubleshooting table covers the common causes. This one volume holds the embedded etcd data, the vector data and the WAL. Attaching it redeploys the service, and it limits the service to one instance. `COMPLETED` means rolled out and serving.

## Step 7: Connect from your app

Inside the project, link `milvus-env` and use the SDK:

```python
import os
from pymilvus import MilvusClient

client = MilvusClient(uri=os.environ["MILVUS_URI"], token=os.environ["MILVUS_TOKEN"])
print(client.list_collections())
```

For least privilege, create a dedicated user and role for the app instead of sharing `root`. See Milvus's [authentication docs](https://milvus.io/docs/authenticate.md).

## How to know it worked

You're done when all of these hold. Open a private tunnel and read the log for the local addresses of ports 19530 and 9091:

```bash
northflank forward service --projectId "$PROJECT_ID" --serviceId milvus --skipHostnames > /tmp/nf-milvus.log 2>&1 &
FWD_PID=$!; sleep 10; cat /tmp/nf-milvus.log
M=http://<LOCAL_ADDR>:<PORT_FOR_19530>; H=http://<LOCAL_ADDR>:<PORT_FOR_9091>
```

1. Health returns `OK`:
   ```bash
   curl -s "$H/healthz"
   ```
2. Authentication is on. Without a token the RESTful API refuses the call, and with the root token it lists collections:
   ```bash
   curl -s -X POST "$M/v2/vectordb/collections/list" -H 'Content-Type: application/json' -d '{}'
   curl -s -X POST "$M/v2/vectordb/collections/list" -H 'Content-Type: application/json' \
     -H "Authorization: Bearer root:$MILVUS_ROOT_PASSWORD" -d '{}'
   ```
   Expect an authentication error code first, then `{"code":0,"data":[...]}`.
3. A test collection returns a search hit:
   ```bash
   MILVUS_URI="$M" MILVUS_TOKEN="root:$MILVUS_ROOT_PASSWORD" python3 - <<'PY'
   import os
   from pymilvus import MilvusClient
   c = MilvusClient(uri=os.environ["MILVUS_URI"], token=os.environ["MILVUS_TOKEN"])
   c.create_collection("smoke", dimension=4)
   c.insert("smoke", [{"id": 1, "vector": [0.1, 0.2, 0.3, 0.4], "text": "hello"}])
   c.flush("smoke")
   print(c.search("smoke", data=[[0.1, 0.2, 0.3, 0.4]], limit=1, output_fields=["text"]))
   PY
   ```
   The result includes `id: 1` and `text: hello`.
4. Data survives a restart. Run `northflank restart service --projectId "$PROJECT_ID" --serviceId milvus`, wait for `COMPLETED`, re-open the tunnel and check that `smoke` is still listed. Then drop it with `c.drop_collection("smoke")` and `kill $FWD_PID`.

## Optional: separate etcd and MinIO (the Compose layout)

To match the official Compose file, or to keep vector data in object storage:

1. **etcd:** Northflank has no etcd addon, so deploy `quay.io/coreos/etcd:v3.5.25` (the Compose version) as a second deployment service named `etcd`. Use custom command `etcd -advertise-client-urls=http://etcd:2379 -listen-client-urls=http://0.0.0.0:2379 --data-dir=/etcd`, a private `TCP` port 2379, the Compose `ETCD_AUTO_COMPACTION_*`, `ETCD_QUOTA_BACKEND_BYTES` and `ETCD_SNAPSHOT_COUNT` variables, and a volume at `/etcd`. Milvus warns that etcd needs fast SSDs.
2. **MinIO:** create a managed MinIO addon and wait until its `status` is `running`. `replicas` is 1 or 4 and is fixed at creation (4 uses erasure coding); `storage` is in MB, must be one of the addon type's storage options (4096, 6144, 8192, 12288, …) and can only grow:
   ```bash
   northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
     "name": "milvus-minio", "type": "minio", "version": "latest",
     "tlsEnabled": false, "externalAccessEnabled": false,
     "billing": { "deploymentPlan": "nf-compute-20", "storage": 12288, "replicas": 1 }
   }'
   OK=0
   for i in $(seq 1 90); do
     S=$(northflank get addon --projectId "$PROJECT_ID" --addonId milvus-minio --noDefaults -o json | jq -r '(.data // .).status')
     echo "status=$S"
     case "$S" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
     sleep 10
   done
   [ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
   ```
   The MinIO docs name the addon's keys `HOST`, `PORT`, `ACCESS_KEY` and `SECRET_KEY` (internal port 9000). Link its host, port and keys into `milvus-env` as `MINIO_ADDRESS`, `MINIO_PORT`, `MINIO_ACCESS_KEY_ID` and `MINIO_SECRET_ACCESS_KEY`, checking real key names with `northflank get secret-link … | jq '(.data // .).availableKeys'`. `MINIO_ADDRESS` is a host without a scheme. Set `MINIO_USE_SSL` to match the addon's TLS setting and `MINIO_BUCKET_NAME` to a bucket name. Milvus creates the bucket if it's missing.
3. **Milvus:** remove `ETCD_USE_EMBED`, `ETCD_DATA_DIR`, `ETCD_CONFIG_PATH` and the runtime file, then set `ETCD_ENDPOINTS=etcd:2379` and `COMMON_STORAGETYPE=remote`. Keep the `/var/lib/milvus` volume, because the Compose file still mounts it for local data.

This layout hasn't been run end to end here. It costs more and gives no extra availability in standalone mode. Choose it when you want data in object storage.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Startup probe fails; logs show `permission denied` under `/var/lib/milvus` | Volume not writable by the `milvus` user | See [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) (permissions); report to the user |
| Logs mention `seccomp`, or crash at start | Official scripts run with `seccomp:unconfined` | Report to the user; this setting isn't exposed |
| `SIGILL` / illegal instruction | Node CPU lacks the SIMD extensions | Ask Northflank support about node CPU types |
| OOM-killed during index build | Plan too small | Move to `nf-compute-400-16` |
| Calls succeed without a token | Auth variables not applied | Check `COMMON_SECURITY_AUTHORIZATIONENABLED` in `runtimeEnvironment`; restart |
| `root:<password>` rejected | Password set after first init (default only applies once) | Use `root:Milvus`, then change it with `update_password` |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 6 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Don't report success until checks 2 to 4 pass: auth refused without a token, a search hit, and the data surviving a restart.
- Never print `MILVUS_ROOT_PASSWORD` or `MILVUS_TOKEN`. Build JSON with `jq --arg`.
- Changing `COMMON_SECURITY_DEFAULTROOTPASSWORD` after the first start does nothing. Rotate with the SDK's `update_password`, then update `MILVUS_TOKEN`.
- Keep ports private unless the user asks otherwise. Port 9091's admin endpoints have no auth by default.
- Ask before deleting the volume, the service or collections other than your own `smoke` test. Back up the volume before upgrades; see [backup and clone volumes](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes).

## FAQ

### What is Milvus standalone?
A single Milvus process that runs every component. It needs etcd and object storage, which can be embedded and local (as here) or separate etcd and MinIO (as in the Compose file). Distributed mode splits the components across Kubernetes pods.

### Where should I deploy Milvus?
For standalone, on a host with at least 8 GB RAM, an SSD volume and private networking next to your app. A container platform with volumes runs the official image without a Kubernetes cluster. Use Kubernetes with the Milvus Operator or Helm for distributed mode, or Zilliz Cloud for a managed service.

### Can I expose Milvus publicly?
The guide doesn't. Northflank's public ports carry HTTP and HTTP/2, and Milvus mixes gRPC and REST on 19530, which hasn't been tested through the public load balancer. Use `northflank forward service` from outside, or put a REST client service in the project in front of Milvus.

### How do I upgrade Milvus?
Back up the volume, read the release notes for your path, then change the tag with `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId milvus --input '{"deployment":{"instances":1,"external":{"imagePath":"milvusdb/milvus:<NEW_VERSION>"}}}'` and wait for `COMPLETED`.

## Conclusion

You now have Milvus standalone on the official image, with embedded etcd and local storage on one persistent volume, authentication on with your own root password, and private ports for the API and health.

**Next:** create an app user with its own role, and move to the external etcd and MinIO layout if you want vector data in object storage. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy Qdrant](deploy-qdrant.md)
- [How to deploy Chroma](deploy-chroma.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
- [How to deploy a Docker image](deploy-docker-image.md)
