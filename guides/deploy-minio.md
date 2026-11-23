<!--
agent-guide-meta:
  id: NF-STK-MINIO
  description: "How to deploy MinIO as S3-compatible object storage, link its keys into your app as AWS variables, use path-style addressing and verify with an upload."
  sources:
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-minio-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/configure-addons-for-high-availability
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/databases-and-persistence/access-a-database
    - https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data
    - https://northflank.com/docs/v1/application/databases-and-persistence/migrate-data-to-northflank/migrate-your-minio-deployment-to-northflank
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/create-addon
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/create-project-secret
    - https://northflank.com/docs/v1/api/project/secrets/get-project-secret-addon-link-details
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/forwarding
    - https://api.northflank.com/v1/addon-types
    - https://min.io/docs/minio/linux/reference/minio-mc.html
    - https://docs.aws.amazon.com/botocore/latest/reference/config.html
    - https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/Package/-aws-sdk-client-s3/Interface/S3ClientConfig/
  related: [NF-DB-001, NF-DEP-004, NF-DEP-006, NF-DEP-001]
-->

# How to deploy MinIO (S3-compatible object storage)

To deploy MinIO as S3-compatible object storage, run it on persistent disks sized for your data, pass its endpoint and access keys to your app as `S3_ENDPOINT`, `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`, create the bucket, and configure the AWS SDK with that custom endpoint, path-style addressing and region `us-east-1`. Without path style, the SDK puts the bucket in the hostname, which doesn't resolve for a private MinIO host.

This guide does it on Northflank with the CLI: a managed MinIO addon, its keys linked into your service through a restricted secret group, and a bucket created over a tunnel. You end with storage your app reaches through the standard AWS variables, verified by an upload and a listing. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run; agree replicas (1 or 4) with the user first, since they're fixed.

## TL;DR: how to deploy MinIO (S3-compatible object storage)

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, project and storage name, and check sizes with the public `https://api.northflank.com/v1/addon-types`, before changing anything.
3. Create the project, then storage with `northflank put addon` (`type: "minio"`). Decide 1 or 4 replicas now; it can't change. Poll `northflank get addon` until `status` is `running`.
4. Link `MINIO_ENDPOINT`, `ACCESS_KEY`, `SECRET_KEY` into a restricted secret group as `S3_ENDPOINT`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, restart the app, and configure the SDK with `endpoint`, `forcePathStyle: true` (or `addressing_style: path`) and region `us-east-1`.
5. Create the bucket and upload a test object to verify it works.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Where to deploy MinIO, and why a managed addon

MinIO needs a long-running server process, persistent disks sized for your data, access keys, and a network path from your app (plus a public endpoint if browsers download files directly). The usual places to run it are a **managed MinIO addon** on a platform that offers one, a **self-run container** such as the official `minio` Docker image on a container platform or Kubernetes with persistent volumes, or a **VM** with attached disks where you install and operate it yourself. Self-running gives full control over disks and erasure coding but leaves volumes, upgrades and backups to you; static and serverless hosts can't keep files on disk, so they need object storage like this anyway.

Container filesystems are ephemeral and a volume only serves one service, so files belong in object storage. On Northflank, MinIO is an **addon**: S3-compatible storage inside your project, reachable by default only from services and jobs in that project, with generated access keys, optional TLS and public access, disk snapshot backups, and a 4-replica erasure-coded mode that keeps reads and writes working with half the replicas down. Your code keeps using the AWS SDK; only the endpoint and addressing style change.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The AWS CLI** (`aws --version`), for the check in step 9.

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

Before changing anything, print the deployment target. The storage goes in the project `my-app`; if your app already has a project, use its name instead throughout, so the app reaches MinIO over the private network.

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId my-app -o json >/dev/null 2>&1 \
  && echo "project: my-app exists and will be reused" \
  || echo "project: my-app will be created"
northflank get addon --projectId my-app --addonId app-storage -o json >/dev/null 2>&1 \
  && echo "addon:   app-storage already exists: pick another name" \
  || echo "addon:   app-storage is free"
curl -s https://api.northflank.com/v1/addon-types \
  | jq '.data.addonTypes[] | select(.type=="minio") | {versions, storage: .resources.storage.options, replicas: .resources.replicas.options}'
```

- If the addon name is taken, pick another and use it everywhere `app-storage` appears. `put addon` in step 5 would otherwise change the existing one.
- The addon-types endpoint is public. At the time of writing it listed version `2025.10.15`, storage from 4096 MB up to 5,120,000 MB, and replicas `1` or `4`. List plan IDs with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[].id'`.
- Agree on size and replicas with whoever owns the bill. Storage can only be increased, and MinIO can't be scaled horizontally after creation.
- Find how the app configures S3: grep for `S3Client`, `boto3.client("s3"`, `endpoint`, `AWS_ACCESS_KEY_ID`, `S3_ENDPOINT`, `forcePathStyle`. Frameworks often have their own names (Django `django-storages` uses `AWS_S3_ENDPOINT_URL`; Laravel uses `AWS_ENDPOINT` and `AWS_USE_PATH_STYLE_ENDPOINT`).

**If you're an agent:** show the user the team, the project (new or reused), the region (`europe-west` below, unless they choose another), the storage name, size and replicas. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "my-app", "region": "europe-west" }'
export PROJECT_ID=my-app
```

REST: `POST https://api.northflank.com/v1/projects` with the same body. For REST calls, also export an API token as `NF_API_TOKEN`.

## Step 4: Write the MinIO definition

Save this as `minio.json`:

```json
{
  "name": "app-storage",
  "description": "S3-compatible object storage for uploads",
  "type": "minio",
  "version": "latest",
  "tlsEnabled": true,
  "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 12288, "replicas": 1 }
}
```

What each choice means:

- **`replicas` is 1 or 4 and is fixed at creation.** The HA docs say MinIO can't be scaled horizontally afterwards. Four replicas use erasure coding, so storage keeps working with up to half of them unavailable. Use 4 for anything you can't regenerate.
- `storage` is in MB, must be one of the addon-types storage options (4096, 6144, 8192, 12288, …), and can only grow. Size it for your objects with headroom.
- `tlsEnabled` can be changed later; it switches the endpoint from `http://` to `https://`. Public access requires it.
- `externalAccessEnabled: true` gives an external `https://` endpoint on port 443. You only need it if clients outside the project (browsers following presigned URLs, other clouds) must reach the storage.
- MinIO has no custom database name, so leave `customCredentials` out. `name` is 3–39 characters starting with a letter; the ID is its slug (`app-storage`).

## Step 5: Create MinIO and wait until it's running

```bash
northflank put addon --projectId "$PROJECT_ID" --file minio.json -o json | jq '(.data // .) | {id, status}'
```

REST:

```bash
curl -sS -X PUT "https://api.northflank.com/v1/projects/$PROJECT_ID/addons" \
  -H "Authorization: Bearer $NF_API_TOKEN" -H "Content-Type: application/json" --data @minio.json
```

Poll every 10 seconds, stopping on a failure state:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get addon --projectId "$PROJECT_ID" --addonId app-storage -o json | jq -r '(.data // .).status')
  echo "status=$S"
  case "$S" in
    running) OK=1; break ;;
    failed|error|errorAllocating|deleting|deleted) echo "provisioning failed"; break ;;
  esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

## Step 6: Inspect the connection details without printing them

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-storage -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

The MinIO docs name `MINIO_ENDPOINT` (format `[http|https]://[host][:port]`, internal port 9000), `ACCESS_KEY`, `SECRET_KEY`, `HOST`, `PORT`, and `MINIO_CONNECT_COMMAND` for the `mc` client. Treat the printed key list as the source of truth for exact names.

## Step 7: Link the storage into your app

Map the MinIO keys to the names the AWS SDKs already read. Save this as `storage-secrets.json`:

```json
{
  "name": "app-storage-connection",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": true, "nfObjects": [ { "id": "<SERVICE_ID>", "type": "service" } ] },
  "addonDependencies": [
    {
      "addonId": "app-storage",
      "keys": [
        { "keyName": "MINIO_ENDPOINT", "aliases": ["S3_ENDPOINT", "AWS_ENDPOINT_URL_S3"] },
        { "keyName": "ACCESS_KEY", "aliases": ["AWS_ACCESS_KEY_ID"] },
        { "keyName": "SECRET_KEY", "aliases": ["AWS_SECRET_ACCESS_KEY"] }
      ]
    }
  ],
  "secrets": { "variables": { "AWS_REGION": "us-east-1", "S3_BUCKET": "uploads" } }
}
```

```bash
northflank put secret --projectId "$PROJECT_ID" --file storage-secrets.json -o json
northflank get secret-link --projectId "$PROJECT_ID" --secretId app-storage-connection --addonId app-storage -o json \
  | jq '(.data // .) | {linked, linkedKeys, availableKeys}'
```

If a key is missing from `linkedKeys`, use the exact name from `availableKeys` and re-link with `northflank update secret-link --projectId "$PROJECT_ID" --secretId app-storage-connection --addonId app-storage --input '{"keys":[…]}'`. Restricting to a service that doesn't exist yet hasn't been confirmed; deploy it first or start unrestricted and tighten later.

Restart the app and wait for `COMPLETED` (rolled out and serving):

```bash
northflank restart service --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> -o json
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> -o json | jq -r '(.data // .).status.deployment.status')
  echo "deployment=$D"; [ "$D" = "COMPLETED" ] && { OK=1; break; }; [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }; sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

## Step 8: Point the AWS SDK at MinIO

Two settings matter: the **custom endpoint** and **path-style addressing**. The default virtual-hosted style puts the bucket in the hostname (`uploads.host`), which doesn't resolve for an internal MinIO host, so force path style (`host/uploads`). MinIO answers to region `us-east-1` unless configured otherwise.

JavaScript (AWS SDK v3), using the `forcePathStyle` client option:

```js
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
const s3 = new S3Client({
  endpoint: process.env.S3_ENDPOINT,
  region: process.env.AWS_REGION ?? 'us-east-1',
  forcePathStyle: true,
  // credentials come from AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY
});
await s3.send(new PutObjectCommand({ Bucket: process.env.S3_BUCKET, Key: 'hello.txt', Body: 'hi' }));
```

Python (boto3), using botocore's `addressing_style`:

```python
import os, boto3
from botocore.config import Config
s3 = boto3.client("s3", endpoint_url=os.environ["S3_ENDPOINT"],
                  region_name=os.environ.get("AWS_REGION", "us-east-1"),
                  config=Config(s3={"addressing_style": "path"}))
s3.put_object(Bucket=os.environ["S3_BUCKET"], Key="hello.txt", Body=b"hi")
```

With TLS on, the image needs CA certificates (`apt-get install -y ca-certificates` or `apk add --no-cache ca-certificates`).

**Presigned URLs for browsers.** A URL signed against the internal endpoint only works inside the project. If browsers must download or upload directly, enable public access (TLS on, `externalAccessEnabled: true`) and sign with a second client whose `endpoint` is the external endpoint shown in the connection details. The external key's name isn't confirmed, so read it from `availableKeys` after enabling access.

## Step 9: Create the bucket and prove it works

A new MinIO has no buckets. Create one through a tunnel without sudo, and read the local address from the log rather than assuming port 9000:

```bash
northflank forward addon --projectId "$PROJECT_ID" --addonId app-storage \
  --skipHostnames --portConflict random > /tmp/nf-minio-fwd.log 2>&1 &
FWD_PID=$!
sleep 10; cat /tmp/nf-minio-fwd.log
```

Load the keys into environment variables without printing them, force path style through a throwaway AWS config file, and run the checks:

```bash
CREDS=$(northflank get addon credentials --projectId "$PROJECT_ID" --addonId app-storage -o json)
pick() { echo "$CREDS" | jq -r --arg k "$1" '(.data // .) | (.secrets + .envs) | to_entries[] | select(.key|ascii_downcase|gsub("_";"")==$k) | .value' | head -1; }
export AWS_ACCESS_KEY_ID=$(pick accesskey) AWS_SECRET_ACCESS_KEY=$(pick secretkey) AWS_REGION=us-east-1
printf '[default]\ns3 =\n  addressing_style = path\n' > /tmp/nf-minio-aws.cfg; export AWS_CONFIG_FILE=/tmp/nf-minio-aws.cfg
EP=https://<LOCAL_IP>:<LOCAL_PORT>
aws --endpoint-url "$EP" --no-verify-ssl s3 mb s3://uploads
echo ok | aws --endpoint-url "$EP" --no-verify-ssl s3 cp - s3://uploads/healthcheck.txt
aws --endpoint-url "$EP" --no-verify-ssl s3 ls s3://uploads/
unset CREDS AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_CONFIG_FILE; kill $FWD_PID
```

`--no-verify-ssl` is for this loopback check only; use `http://` if TLS is off. `pick` ignores case and underscores, so it matches both `ACCESS_KEY` and `accessKey`. If it still returns empty, match the key name you saw in step 6. With the [MinIO client](https://min.io/docs/minio/linux/reference/minio-mc.html) instead, `mc mb` and `mc ls` do the same, with `--insecure` for the loopback address.

Alternatively, have the app create its bucket at startup with `CreateBucket` and ignore `BucketAlreadyOwnedByYou`.

## How to know it worked

1. `northflank get addon --projectId "$PROJECT_ID" --addonId app-storage -o json | jq -r '(.data // .) | .status, .spec.type'` prints `running` and `minio`.
2. `aws s3 ls s3://uploads/` lists `healthcheck.txt`.
3. The service has the endpoint, checked by scheme only (expect `https://` with TLS):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> \
     --shell-cmd 'sh -c' --cmd '"printenv S3_ENDPOINT | cut -c1-8; env | cut -d= -f1 | grep ^AWS_"'
   ```
   `exec` drops output without a TTY, hence `script`; on Linux use `script -qefc '…' /dev/null`.
4. An upload through the app succeeds, and no S3 errors appear in its logs:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId <SERVICE_ID> --duration 300 --textIncludes "NoSuchBucket"
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `getaddrinfo ENOTFOUND uploads.<host>` | Virtual-hosted addressing | Set `forcePathStyle: true` / `addressing_style: "path"` |
| `NoSuchBucket` | Bucket never created | Create it (step 9) or at app startup |
| `SignatureDoesNotMatch` | Wrong region or endpoint altered after signing | Use `us-east-1`; sign with the same endpoint the client calls |
| Presigned URL fails in the browser | Signed with the internal endpoint | Enable public access; sign with the external endpoint |
| Checksum or `x-amz-checksum` errors on upload | Newer SDK checksum defaults | Set `request_checksum_calculation="when_required"` (botocore) or the SDK's equivalent |
| TLS error in the app | Image lacks CA certificates | Install `ca-certificates` |
| `S3_ENDPOINT` empty in the container | Service not in `restrictions`, or no restart | Fix `nfObjects`; restart |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 5 loop prints `ADDON_NOT_READY`, read the addon logs (`northflank get addon logs --projectId "$PROJECT_ID" --addonId app-storage`) before changing anything.
- Confirm replicas (1 or 4) and storage with the user before step 5. Replicas can't change and storage can't shrink.
- Never print `AWS_SECRET_ACCESS_KEY`, `SECRET_KEY` or `MINIO_CONNECT_COMMAND`. Print key names only.
- Keep `externalAccessEnabled: false` unless the user needs browser or off-platform access.
- Never delete buckets, run `aws s3 rm --recursive`, or run `northflank delete addon` without explicit confirmation for that specific storage. Object deletion is permanent.

## FAQ

### Where should I deploy my MinIO storage?
Close to the app that reads and writes the files: same region, ideally the same private network, so uploads and downloads don't cross the public internet. A managed MinIO addon is the least work; run the `minio` Docker image on Kubernetes or a container platform, or install it on a VM with attached disks, when you need control over disk layout and can own volumes, upgrades and backups.

### Is this a drop-in replacement for Amazon S3?
For the common API (put, get, list, delete, multipart uploads, presigned URLs) yes, through any S3 SDK with a custom endpoint and path-style addressing. AWS-specific features such as IAM policies and S3 event notifications to AWS services don't carry over.

### Can I browse the files?
Yes. Forward the addon and open the `MINIO_ENDPOINT` address in a browser for the console, logging in with the access and secret keys.

### How many requests can it handle?
MinIO sizes its request limit from available memory; a bigger plan raises it. You can set `requests_max` with `mc admin config set`.

### How do I move existing buckets in?
Follow [migrate your MinIO deployment to Northflank](https://northflank.com/docs/v1/application/databases-and-persistence/migrate-data-to-northflank/migrate-your-minio-deployment-to-northflank). Backups are disk snapshots; see [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data).

## Conclusion

You now have S3-compatible object storage next to your app, wired in with the standard AWS variable names through a restricted secret group, a bucket created, and an upload and listing that proved it works without printing a secret.

**Next:** deploy the app that writes to it, such as Directus or [Strapi](deploy-strapi.md), and add [PostgreSQL](deploy-postgresql-database.md) for the metadata. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Strapi](deploy-strapi.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
