<!--
agent-guide-meta:
  id: NF-STK-NEXTCLOUD
  description: "How to self-host Nextcloud with the official Docker image, managed PostgreSQL and Redis, a /var/www/html volume, trusted domains and HTTPS overwrite settings."
  sources:
    - https://northflank.com/stacks/deploy-nextcloud
    - https://hub.docker.com/_/nextcloud
    - https://github.com/nextcloud/docker/blob/master/README.md
    - https://github.com/nextcloud/docker/blob/master/docker-entrypoint.sh
    - https://github.com/nextcloud/docker/blob/master/.config/reverse-proxy.config.php
    - https://github.com/nextcloud/docker/blob/master/.config/redis.config.php
    - https://docs.nextcloud.com/server/latest/admin_manual/installation/system_requirements.html
    - https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/reverse_proxy_configuration.html
    - https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/background_jobs_configuration.html
    - https://docs.nextcloud.com/server/latest/admin_manual/occ_apps.html
    - https://docs.nextcloud.com/server/latest/admin_manual/configuration_files/primary_storage.html
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/secrets/get-project-secret-addon-link-details
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/project/jobs/run-job
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-redis-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DB-001, NF-DB-003, NF-STK-WORDPRESS, NF-STK-MINIO, NF-DEP-002]
-->

# How to self-host Nextcloud

To self-host Nextcloud, run the official `nextcloud` Apache image with a persistent volume at `/var/www/html`, give it PostgreSQL through `POSTGRES_*` variables and Redis through `REDIS_HOST*` for caching and file locking, add your public hostname to `NEXTCLOUD_TRUSTED_DOMAINS`, set `OVERWRITEPROTOCOL=https` behind the TLS proxy, and schedule `cron.php`. Without the trusted domain, Nextcloud refuses every request with an "untrusted domain" page.

This guide does it on Northflank with the CLI: managed PostgreSQL and Redis addons, a secret group holding their credentials and a generated admin password, the official Apache image with a volume, and a cron job that calls `cron.php`. You end with an installed Nextcloud on a public HTTPS URL, verified with `status.php` and `occ`. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to self-host Nextcloud

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Pin a Nextcloud version, and confirm the team, project, region and disk size before changing anything.
3. Create a project, PostgreSQL, Redis and a secret group that links their credentials plus a generated admin password.
4. Create the `nextcloud:<tag>-apache` service paused, attach a volume at `/var/www/html`, then start it with your public host in `NEXTCLOUD_TRUSTED_DOMAINS`.
5. Schedule `curl https://<host>/cron.php` every 5 minutes with a cron job.
6. Verify that `status.php` reports `"installed":true` and `occ` shows the trusted domain and Redis locking.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

The official image installs Nextcloud on first start. If the PostgreSQL variables and both `NEXTCLOUD_ADMIN_*` variables are set, it runs `occ maintenance:install` without the web wizard, and then applies `NEXTCLOUD_TRUSTED_DOMAINS`. That last part only runs at install time, which is why this guide reads the public hostname **before** the first boot. A few other choices follow from the image's docs:

- **`/var/www/html` on a volume.** The image keeps the Nextcloud code, `config/`, `custom_apps/` and `data/` there, and upgrades work by syncing new code into that folder. The volume must be attached before the first boot; otherwise the install lands on ephemeral disk while the database tables persist, and the next start can't reinstall.
- **Redis.** With `REDIS_HOST` set, the image configures Redis for distributed caching, transactional file locking and PHP sessions.
- **`OVERWRITEPROTOCOL=https`.** TLS ends at the load balancer, so Nextcloud sees plain HTTP. The image reads `OVERWRITEPROTOCOL`, `OVERWRITECLIURL` and `TRUSTED_PROXIES` from the environment on every request, so these can change later without touching `config.php`.
- **PostgreSQL**, not SQLite, because the image falls back to SQLite when the database variables are incomplete.

Where to host Nextcloud comes down to what it needs: an always-on PHP and Apache process, PostgreSQL or MySQL, Redis, a disk that survives restarts and grows with your files, and a public HTTPS URL. Serverless and static hosts can't keep files on disk. A VM works but leaves backups, TLS and upgrades to you. A container platform with managed databases and volumes runs the same official image and handles the rest.

If you'd rather click through it, Northflank has a [Nextcloud stack template](https://northflank.com/stacks/deploy-nextcloud) that deploys a similar setup from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** and **`openssl`**.
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **A disk size agreed with the user.** All files live on the volume, and storage can only grow.

Names used below: project `nextcloud`, addons `nextcloud-db` and `nextcloud-redis`, secret group `nextcloud-env`, service `nextcloud`, volume `nextcloud-data`, job `nextcloud-cron`.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Pin a version and confirm where it will deploy

Before changing anything, pin a Nextcloud version and print the deployment target. This lists the newest full `-apache` tag on Docker Hub (35.0.3 at the time of writing):

```bash
NC_VERSION=$(for p in 1 2; do curl -s "https://hub.docker.com/v2/repositories/library/nextcloud/tags?page_size=100&page=$p&ordering=last_updated" | jq -r '.results[].name'; done \
  | grep -E '^[0-9]+\.[0-9]+\.[0-9]+-apache$' | sort -V | tail -n1); echo "version: $NC_VERSION"

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId nextcloud -o json >/dev/null 2>&1 \
  && echo "project: nextcloud exists and will be reused" \
  || echo "project: nextcloud will be created"
```

- The Nextcloud docs recommend at least 512 MB per PHP process, so the service starts on `nf-compute-100-2` (1 vCPU, 2 GB). List plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | "\(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'`.
- The volume below is 20 GB (`storageSize: 20480`). Change it to the size agreed with the user.
- If the project is reused and already has a `nextcloud` service or volume, stop and ask the user: the steps below change them.

**If you're an agent:** show the user the team, the Nextcloud version, the region (`europe-west` below, unless they choose another), the disk size and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create a project

A project holds your services and addons. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "nextcloud", "region": "europe-west" }'
export PROJECT_ID=nextcloud
```

## Step 4: Create PostgreSQL and Redis

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "nextcloud-db", "type": "postgresql", "version": "17",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "nextcloud" }
}'
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "nextcloud-redis", "type": "redis", "version": "8.10.1",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 4096, "replicas": 1 }
}'
```

Check versions with `curl -s https://api.northflank.com/v1/addon-types` (PostgreSQL 14 to 18 are in Nextcloud's supported range). Both addons are private to the project; TLS is off to match the image's defaults (`REDIS_PROTOCOL=tcp`). Poll until both are `running`:

```bash
OK=1
for a in nextcloud-db nextcloud-redis; do
  A_OK=0
  for i in $(seq 1 90); do
    S=$(northflank get addon --projectId "$PROJECT_ID" --addonId "$a" --noDefaults -o json | jq -r '(.data // .).status')
    echo "$a=$S"
    case "$S" in running) A_OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
    sleep 10
  done
  [ "$A_OK" = 1 ] || { OK=0; break; }
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon status and logs"; false; }
```

## Step 5: Check the credential key names

Print **key names only**, never values:

```bash
for a in nextcloud-db nextcloud-redis; do
  northflank get addon credentials --projectId "$PROJECT_ID" --addonId "$a" --noDefaults -o json \
    | jq --arg a "$a" '(.data // .) | {addon: $a, secrets: (.secrets | keys), envs: (.envs | keys)}'
done
```

The PostgreSQL docs name `HOST`, `PORT`, `USERNAME`, `PASSWORD` and `DATABASE`; the Redis docs mention `host`, `port` and `password`. If the casing differs, change the `keyName` values in step 6. PostgreSQL listens on 5432, which is the default Nextcloud uses, so `POSTGRES_HOST` needs no port.

## Step 6: Create the secret group

```bash
NC_ADMIN_PASSWORD=$(openssl rand -base64 24)
jq -n --arg pw "$NC_ADMIN_PASSWORD" '{
  name: "nextcloud-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [
    { addonId: "nextcloud-db", keys: [
      { keyName: "HOST",     aliases: ["POSTGRES_HOST"] },
      { keyName: "DATABASE", aliases: ["POSTGRES_DB"] },
      { keyName: "USERNAME", aliases: ["POSTGRES_USER"] },
      { keyName: "PASSWORD", aliases: ["POSTGRES_PASSWORD"] } ] },
    { addonId: "nextcloud-redis", keys: [
      { keyName: "HOST",     aliases: ["REDIS_HOST"] },
      { keyName: "PORT",     aliases: ["REDIS_HOST_PORT"] },
      { keyName: "PASSWORD", aliases: ["REDIS_HOST_PASSWORD"] } ] } ],
  secrets: { variables: { NEXTCLOUD_ADMIN_USER: "admin", NEXTCLOUD_ADMIN_PASSWORD: $pw } }
}' > nextcloud-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file nextcloud-env.json >/dev/null && rm nextcloud-env.json
unset NC_ADMIN_PASSWORD

northflank get secret-link --projectId "$PROJECT_ID" --secretId nextcloud-env --addonId nextcloud-redis --noDefaults -o json \
  | jq '(.data // .) | {linkedKeys, availableKeys}'
```

If a Redis key is missing from `linkedKeys`, pick the right name from `availableKeys` and re-run `put secret` with it. **Tell the user to copy the admin password from the `nextcloud-env` secret group in the UI to their password manager.**

## Step 7: Create the service paused and attach the volume

Write `nextcloud.json`, replacing `<NC_VERSION>` with the tag from step 2. `instances: 0` keeps the container from starting until the volume and hostname are in place.

```json
{
  "name": "nextcloud",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 0,
    "external": { "imagePath": "nextcloud:<NC_VERSION>" },
    "docker": { "configType": "default" }
  },
  "ports": [ { "name": "http", "internalPort": 80, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": {
    "PHP_UPLOAD_LIMIT": "1G", "PHP_MEMORY_LIMIT": "512M", "OVERWRITEPROTOCOL": "https",
    "TRUSTED_PROXIES": "10.0.0.0/8 172.16.0.0/12 192.168.0.0/16"
  },
  "healthChecks": [
    { "protocol": "TCP", "type": "readinessProbe", "port": 80,
      "initialDelaySeconds": 20, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 60, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file nextcloud.json

northflank get volume --projectId "$PROJECT_ID" --volumeId nextcloud-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "nextcloud-data",
  "mounts": [ { "containerMountPath": "/var/www/html" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 20480 },
  "attachedObjects": [ { "id": "nextcloud", "type": "service" } ]
}'
```

Apache only listens on port 80 after the entrypoint finishes installing or upgrading, so a TCP readiness probe means "install done". The high `failureThreshold` gives the first install ten minutes. There's no liveness probe, so a slow upgrade isn't killed halfway. `PHP_UPLOAD_LIMIT` raises `upload_max_filesize` and `post_max_size` (default 512M). The image runs as root and chowns the code to `www-data` as it copies it into the volume.

## Step 8: Set the hostname and start Nextcloud

Read the public hostname, then send the **complete** `runtimeEnvironment` map (the patch replaces it) together with `instances: 1`:

```bash
for i in $(seq 1 18); do
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId nextcloud --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId nextcloud --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns // empty' | head -n1)
  [ -n "$HOST" ] && break; sleep 10
done; echo "HOST=$HOST"

northflank patch service deployment --projectId "$PROJECT_ID" --serviceId nextcloud --noDefaults -o json --input "$(jq -n --arg h "$HOST" '{
  deployment: { instances: 1 },
  runtimeEnvironment: {
    PHP_UPLOAD_LIMIT: "1G", PHP_MEMORY_LIMIT: "512M", OVERWRITEPROTOCOL: "https",
    TRUSTED_PROXIES: "10.0.0.0/8 172.16.0.0/12 192.168.0.0/16",
    NEXTCLOUD_TRUSTED_DOMAINS: $h, OVERWRITECLIURL: ("https://" + $h)
  }
}')"
```

If `HOST` is still empty while the service is paused, run the patch with `NEXTCLOUD_TRUSTED_DOMAINS` left out, read the host once the service is running, and add it as in the troubleshooting table. Then wait for the rollout:

```bash
OK=0
for i in $(seq 1 90); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId nextcloud --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  northflank get service logs --projectId "$PROJECT_ID" --serviceId nextcloud --lineLimit 100 | grep -E 'Installing with PostgreSQL|Setting trusted domains|Initializing finished|failed'
  echo "https://$HOST"
fi
```

Expect `Installing with PostgreSQL database`, `Setting trusted domains…` and `Initializing finished`. `COMPLETED` means rolled out and serving.

## Step 9: Schedule background jobs

The image's own cron example runs a second container on the same volume, which a `ReadWriteOnce` volume doesn't allow. So this guide uses Nextcloud's **webcron** mode: a scheduled job calls `https://<host>/cron.php` every 5 minutes. First switch the mode (Linux: `script -qefc '<cmd>' /dev/null`):

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId nextcloud --user www-data \
  --shell-cmd 'sh -c' --cmd '"php /var/www/html/occ background:webcron"'
```

Then create the cron job. `put job` with `settings.cron` makes a cron job (`put job cron` is deprecated):

```bash
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --input "$(jq -n --arg h "$HOST" '{
  name: "nextcloud-cron",
  billing: { deploymentPlan: "nf-compute-10" },
  settings: { backoffLimit: 0, activeDeadlineSeconds: 240, runOnSourceChange: "never",
    cron: { schedule: "*/5 * * * *", concurrencyPolicy: "forbid", timeZone: "UTC" } },
  deployment: { external: { imagePath: "curlimages/curl:8.23.0" },
    docker: { configType: "customCommand", customCommand: ("curl -fsS -o /dev/null https://" + $h + "/cron.php") } }
}')"

RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId nextcloud-cron --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 30); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId nextcloud-cron --runId "$RUN_ID" --noDefaults -o json)
  S=$(echo "$R" | jq -r '(.data // .).status'); echo "run=$S"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$S" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "JOB_NOT_SUCCEEDED: stop here and read the job run logs"; false; }
```

**The trade-off:** the Nextcloud docs say webcron runs one job per call, which suits small instances (roughly 1 to 5 users). For more users, cron mode (`php -f cron.php` as `www-data` in the same container) is the recommended setup; that needs a custom image that runs a scheduler next to Apache, which this guide doesn't cover.

## How to know it worked

You're done when all of these hold:

1. Nextcloud is installed and not in maintenance mode:
   ```bash
   curl -s "https://$HOST/status.php" | jq '{installed, maintenance, needsDbUpgrade, versionstring}'
   ```
   Expect `"installed": true`, `"maintenance": false`, `"needsDbUpgrade": false`.
2. The login page loads on the public host (an untrusted domain returns an error page instead):
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/login"   # 200
   ```
3. `occ` shows PostgreSQL, the trusted domain and Redis locking (no secrets printed):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId nextcloud --user www-data --shell-cmd 'sh -c' \
     --cmd '"php /var/www/html/occ config:system:get dbtype; php /var/www/html/occ config:system:get trusted_domains; php /var/www/html/occ config:system:get memcache.locking; php /var/www/html/occ config:system:get overwriteprotocol"'
   ```
   Expect `pgsql`, a list containing your host, `\OC\Memcache\Redis` and `https`.
4. The cron job run concluded with `status == "SUCCESS"`.
5. **This step needs a human:** log in as `admin`, open **Administration settings → Overview** and check the security and setup warnings, then confirm **Basic settings → Background jobs** shows Webcron with a recent last run.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Access through untrusted domain" | Host wasn't set before install, or you added a custom domain | `occ config:system:set trusted_domains 2 --value=<host>` via exec, as in check 3 |
| Logs show `Installing with SQLite` or the setup wizard appears | A `POSTGRES_*` variable is missing | Check key names (step 5) and fix the secret group; if no users or files exist yet, ask the user before recreating the volume and database, then restart |
| Install retries, then `Installing of nextcloud failed!` | Database unreachable or credentials wrong | Confirm the addon is `running` and the four `POSTGRES_*` keys are linked |
| Login redirects to `http://` or loops | `OVERWRITEPROTOCOL` not applied | Re-run step 8's patch; if still wrong, also set `OVERWRITEHOST` to the host |
| Errors connecting to Redis | Wrong key linked, or Redis created with TLS | Check `availableKeys`; with TLS on, set `REDIS_PROTOCOL=tls` |
| `exec … --user www-data` errors | User switching not supported by exec | Run as root with `su -p www-data -s /bin/sh -c "php /var/www/html/occ status"` |
| `Console has to be executed with the user that owns config.php` | `occ` ran as root | Run it as `www-data` |
| Can't scale to 2 instances | Single `ReadWriteOnce` volume | Scale up the plan instead |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write: team, version, region, disk size and whether the project is new or reused.
- Don't report success until checks 1 to 4 pass.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the deployment logs before changing anything.
- Never print `NEXTCLOUD_ADMIN_PASSWORD`, database or Redis passwords, or the credentials response. Print key names only, and don't use `occ config:list --private`.
- Attach the volume before the first boot (step 7 before step 8). If an install has already run on ephemeral disk, stop and tell the user rather than wiping the database.
- Get explicit confirmation before deleting the volume, either addon or the service. Back up the database and take a volume backup before changing the image's major version, and only move up one major version at a time (the entrypoint refuses bigger jumps).

## FAQ

### Can I use MySQL or MariaDB instead of PostgreSQL?
Yes. The image also accepts `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD` and `MYSQL_HOST`; set one database type only. See [how to deploy a MySQL database](deploy-mysql-database.md) for the addon. This guide uses PostgreSQL because Nextcloud's requirements list current PostgreSQL versions and Northflank's PostgreSQL addon links cleanly as separate keys.

### Can I store files in S3-compatible object storage instead of the volume?
The image supports primary object storage through `OBJECTSTORE_S3_*` variables, for example against a [MinIO addon](deploy-minio.md). Decide before the first install: Nextcloud's primary storage docs warn that configuring it on an existing instance makes all existing files inaccessible. The volume is still needed for code and config.

### How do I upgrade Nextcloud?
Back up the database and volume, then change the tag to the next major or a newer patch: `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId nextcloud --input '{"deployment":{"instances":1,"external":{"imagePath":"nextcloud:<NEW_TAG>"}}}'`. The entrypoint runs `occ upgrade` on start; wait for `COMPLETED` and re-check `status.php`.

### Where should I host Nextcloud?
Anywhere that runs the official image as an always-on container next to PostgreSQL, Redis and a persistent disk sized for your files, behind HTTPS. Serverless and static hosts don't fit. A VM or Kubernetes cluster works if you run the proxy, TLS, backups and upgrades yourself; a container platform with managed databases and volumes, like the setup in this guide, takes those off your hands.

## Conclusion

You now have Nextcloud on the official image, installed against managed PostgreSQL, using Redis for caching and file locking, with its code, config and files on a persistent volume, HTTPS-aware URLs and scheduled background jobs.

**Next:** add a custom domain by [linking a domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port), then add it as a trusted domain and update `OVERWRITECLIURL` and the cron job URL. Set up [volume backups](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
- [How to deploy a WordPress website](deploy-wordpress-website.md)
- [How to self-host n8n](self-host-n8n.md)
