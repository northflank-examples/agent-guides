<!--
agent-guide-meta:
  id: NF-STK-WORDPRESS
  description: "How to deploy a WordPress website in production with the official Docker image, managed MySQL 8, fixed salts and a persistent wp-content volume over HTTPS."
  sources:
    - https://northflank.com/stacks/deploy-wordpress
    - https://hub.docker.com/_/wordpress
    - https://github.com/docker-library/docs/blob/master/wordpress/content.md
    - https://github.com/docker-library/wordpress/blob/master/docker-entrypoint.sh
    - https://wordpress.org/support/article/administration-over-ssl/#using-a-reverse-proxy
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-mysql-on-northflank
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DB-004, NF-DEP-002, NF-DEP-009]
-->

# How to deploy a WordPress website

To deploy a WordPress website in production, run the official `wordpress` Docker image against a MySQL 8 database that already exists, pass the database credentials as `WORDPRESS_DB_*` variables, set the eight `WORDPRESS_*_KEY` and `_SALT` values to fixed secrets, and mount persistent storage at `wp-content`. Without fixed salts every restart logs users out, and without the volume uploads, themes and plugins disappear on redeploy.

This guide does it on Northflank with the CLI: a managed MySQL 8 addon, a secret group holding the credentials and generated salts, a `wordpress:7-apache` service and a volume at `/var/www/html/wp-content`. You end with a public HTTPS URL serving an installed WordPress site, verified through the REST API and a restart test. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is, except the install wizard, which a human completes once.

## TL;DR: how to deploy a WordPress website

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team and the project before changing anything.
3. Create a project, a MySQL 8 addon, and a secret group with the database credentials and eight generated `WORDPRESS_*_KEY`/`_SALT` values.
4. Run `wordpress:7-apache` with `northflank put service deployment` on port 80 and attach a volume at `/var/www/html/wp-content`.
5. Wait for the rollout, then complete the install wizard at `https://<HOST>/wp-admin/install.php`.
6. Verify that `/wp-json/` answers with an `https://` URL, `wp-login.php` returns 200 and a file in `wp-content` survives a restart.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

The official image does a lot on its own. On first start it copies WordPress into `/var/www/html` and writes `wp-config.php` from `WORDPRESS_*` environment variables. It also adds the `X-Forwarded-Proto` handling WordPress needs behind a TLS-terminating proxy. Three details still matter in a container platform:

- **The database must already exist.** The image docs say `WORDPRESS_DB_NAME` "will not be created". The addon's `dbName` takes care of that.
- **Salts must be fixed.** If you don't set the `WORDPRESS_*_KEY` and `_SALT` variables, the image generates random ones into `wp-config.php` in the container's own filesystem. Every restart would then log everyone out.
- **`wp-content` must be persistent.** It holds uploads, themes and plugins. WordPress core is copied from the image on each start, so its version follows the image tag.

Where to deploy WordPress depends on that list: a PHP web server that's always running, a MySQL database and persistent storage for `wp-content`. Static and serverless hosts don't run a stateful PHP app with a database, so they only fit if you export WordPress to static files. A raw VM gives full control, but you manage PHP, TLS, backups and patches yourself. A container platform with managed databases runs the official Docker image as-is and handles TLS and the database for you.

If you'd rather click through it, Northflank has a [WordPress stack template](https://northflank.com/stacks/deploy-wordpress) with the same pieces: a WordPress deployment, a volume, a MySQL addon and a secret group.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`** to generate the salts (`openssl version`).

Names used below: project `wordpress`, addon `wp-db`, secret group `wp-env`, service `wordpress`, volume `wp-content`.

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

Before changing anything, print the deployment target and check that MySQL 8 is offered:

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId wordpress -o json >/dev/null 2>&1 \
  && echo "project: wordpress exists and will be reused" \
  || echo "project: wordpress will be created"
curl -s https://api.northflank.com/v1/addon-types | jq -c '.data.addonTypes[] | select(.type=="mysql") | .majors'
```

The last line should include an `8` entry. This guide uses the `nf-compute-50` plan for WordPress and `nf-compute-20` for MySQL; list others with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[].id'`.

**If you're an agent:** show the user the team, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create a project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "wordpress", "region": "europe-west" }'
export PROJECT_ID=wordpress
```

## Step 4: Create MySQL 8 and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "wp-db", "type": "mysql", "version": "8-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "wordpress" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId wp-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon status and logs"; false; }
```

The database is only reachable inside the project, and the image doesn't configure MySQL TLS, so TLS is off here. According to the MySQL docs, `dbName`, TLS and high availability are fixed at creation. Storage can only be increased.

## Step 5: Create the secret group

Print the credential **key names** first (never the values):

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId wp-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

The MySQL docs list `HOST`, `PORT`, `USERNAME`, `PASSWORD` and `DATABASE`. If the output uses different casing, change the `keyName` values below. Then generate the salts and create the group without echoing them:

```bash
SALTS=$(for k in AUTH_KEY SECURE_AUTH_KEY LOGGED_IN_KEY NONCE_KEY AUTH_SALT SECURE_AUTH_SALT LOGGED_IN_SALT NONCE_SALT; do
  printf '%s\t%s\n' "WORDPRESS_$k" "$(openssl rand -hex 32)"; done)
echo "$SALTS" | jq -R -n '[inputs | split("\t") | {(.[0]): .[1]}] | add' > salts.json
jq -n --slurpfile s salts.json '{
  name: "wp-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "wp-db", keys: [
    { keyName: "HOST",     aliases: ["WORDPRESS_DB_HOST"] },
    { keyName: "DATABASE", aliases: ["WORDPRESS_DB_NAME"] },
    { keyName: "USERNAME", aliases: ["WORDPRESS_DB_USER"] },
    { keyName: "PASSWORD", aliases: ["WORDPRESS_DB_PASSWORD"] }
  ] } ],
  secrets: { variables: ($s[0] + { WORDPRESS_TABLE_PREFIX: "wp_" }) }
}' > wp-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file wp-env.json >/dev/null && rm wp-env.json salts.json
unset SALTS
```

`WORDPRESS_DB_HOST` takes `host` or `host:port`. MySQL's default port is 3306. If the credentials list a different `PORT`, set `WORDPRESS_DB_HOST` to `host:port` instead.

## Step 6: Deploy the WordPress service

Write `wordpress.json`:

```json
{
  "name": "wordpress",
  "billing": { "deploymentPlan": "nf-compute-50" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "wordpress:7-apache" },
    "docker": { "configType": "default" }
  },
  "ports": [ { "name": "http", "internalPort": 80, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "TCP", "type": "readinessProbe", "port": 80,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 6, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file wordpress.json
```

`7-apache` tracks the latest WordPress 7 release with Apache on port 80 (the `6-apache` tag no longer receives updates). Pin a full version tag such as `7.x.y-php8.4-apache` if you want controlled upgrades. A TCP probe is used because WordPress answers `/` with a redirect until it's installed.

## Step 7: Attach the `wp-content` volume

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId wp-content --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "wp-content",
  "mounts": [ { "containerMountPath": "/var/www/html/wp-content" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "wordpress", "type": "service" } ]
}'
```

Attaching redeploys the service. The image's entrypoint skips any plugin or theme folder that already exists in `wp-content`, so your installed versions are kept on restart. A single read/write volume limits the service to one instance, and the old container stops before the new one starts. Volume size can only be increased.

## Step 8: Wait for the rollout and get the URL

This loop checks every 10 seconds and prints your URL when the site is live:

```bash
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId wordpress --noDefaults -o json \
      | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId wordpress --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId wordpress --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId wordpress`. The troubleshooting table covers the common causes.

## Step 9: Check `wp-content` is writable, then install

Before installing, check who owns the volume (Linux: `script -qefc '<cmd>' /dev/null`):

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId wordpress \
  --shell-cmd 'sh -c' --cmd '"stat -c %U:%G /var/www/html/wp-content /var/www/html/wp-content/themes"'
```

Both lines should read `www-data:www-data`. If not, fix ownership once with the same command and `--cmd '"chown -R www-data:www-data /var/www/html/wp-content"'`.

**This step needs a human:** open `https://$HOST/wp-admin/install.php` right away and complete the wizard (site title, admin username, password, email). WordPress stores the site URL from this first visit, so use the final public address. Until someone installs it, anyone with the URL can.

## How to know it worked

You're done when all of these hold:

1. The REST API answers with JSON after install:
   ```bash
   curl -s "https://$HOST/wp-json/" | jq -r '.name, .url'
   ```
   `.url` must start with `https://`.
2. `curl -sL -o /dev/null -w '%{http_code}\n' "https://$HOST/wp-login.php"` returns 200.
3. Content survives a restart. Write a marker, restart, then read it back:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId wordpress --shell-cmd 'sh -c' --cmd '"date > /var/www/html/wp-content/.persist-check"'
   northflank restart service --projectId "$PROJECT_ID" --serviceId wordpress --noDefaults -o json
   # wait for COMPLETED (step 8 loop), then:
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId wordpress --shell-cmd 'sh -c' --cmd '"cat /var/www/html/wp-content/.persist-check"'
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Error establishing a database connection" | `WORDPRESS_DB_*` not injected, or wrong key names | Re-check step 5 key names; restart |
| Redirect loop or `http://` links on an HTTPS page | Proxy header not seen by WordPress | Add `WORDPRESS_CONFIG_EXTRA` = `$_SERVER['HTTPS'] = 'on';` to the group; restart |
| Everyone logged out after each restart | Salts not set | Make sure all eight `WORDPRESS_*_KEY`/`_SALT` values are in the group |
| Uploads fail: "Unable to create directory" | Volume not writable by `www-data` | Run the `chown` in step 9 |
| Uploads gone after redeploy | Volume not attached | `northflank get volume … --volumeId wp-content` and check `attachedObjects` |
| "Upload exceeds upload_max_filesize" | PHP default limits | Build your own image `FROM wordpress:7-apache` with a `php.ini` override |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Don't report success until the checks below pass, including the restart test in check 3.
- Never print database passwords, salts or the credentials response. Print key names only.
- Don't change salts on a live site without asking; it logs every user out.
- Hand the install wizard to the user immediately after the first rollout.
- Get explicit confirmation before deleting the addon, the volume or the service. Back up both the database and the volume before major upgrades (see [volume backups](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes)).

## FAQ

### How do I ship a custom theme or plugin?
Build an image `FROM wordpress:7-apache` and copy your code into `/usr/src/wordpress/wp-content/themes/` or `plugins/`. The image docs say that content is copied into `/var/www/html` on start, but only when the folder isn't already on the volume.

### Can I run more than one instance?
Not with a single read/write volume. To scale out, move uploads to object storage with a plugin, then drop the volume.

### Does WordPress update itself?
Plugins and themes on the volume can. WordPress core comes from the image on each start, so upgrade core by changing the image tag.

### Where should I deploy my WordPress website?
On a host that runs a long-lived PHP container (or server) next to a MySQL database and gives `wp-content` persistent storage. Static and serverless platforms don't fit a standard WordPress install. A VM works if you want to manage the stack yourself; a container platform with managed MySQL, as in this guide, runs the official Docker image and takes care of TLS and the database.

## Conclusion

You now have WordPress on the official image, backed by managed MySQL 8, with fixed salts and a persistent `wp-content` volume behind a public HTTPS URL.

**Next:** add your domain by [linking a domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port), then update the site address under Settings → General. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a MySQL database](deploy-mysql-database.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy a Laravel project](deploy-laravel-project.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
