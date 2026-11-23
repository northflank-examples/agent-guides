<!--
agent-guide-meta:
  id: NF-STK-GHOST
  description: "How to deploy Ghost CMS in production with the official Docker image, managed MySQL 8, a persistent content volume, SMTP email and a public HTTPS URL."
  sources:
    - https://northflank.com/stacks/deploy-ghost
    - https://hub.docker.com/_/ghost
    - https://github.com/docker-library/docs/blob/master/ghost/content.md
    - https://github.com/docker-library/docs/blob/master/ghost/compose.yaml
    - https://docs.ghost.org/config/
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-mysql-on-northflank
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DB-004, NF-DEP-002]
-->

# How to deploy Ghost CMS

To deploy Ghost CMS in production, run the official `ghost` Docker image against a MySQL 8 database, configure it through environment variables (`database__*`, `mail__*`), mount a persistent volume at `/var/lib/ghost/content`, and set `url` to the public HTTPS address. Ghost won't run on SQLite in production, and without working SMTP you can lock yourself out of the admin.

This guide does it on Northflank with the CLI: managed MySQL 8 linked through a secret group, the `ghost:6-alpine` image on port 2368 with a content volume, and `url` set from the assigned hostname. You end with a public HTTPS blog, checked with `curl` and a test email. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is, except two human steps: SMTP credentials and the owner account.

## TL;DR: how to deploy Ghost CMS

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, the project and that your SMTP credentials are set before changing anything.
3. Create a project and a MySQL 8 addon, then a secret group that maps its credentials to `database__connection__*` and holds your `mail__*` SMTP settings.
4. Run `ghost:6-alpine` on port 2368 with `northflank put service deployment` and attach a volume at `/var/lib/ghost/content`.
5. Set `url=https://<HOST>` from the public host, restart, and create the owner account at `/ghost/`.
6. Check that the homepage's canonical link uses the HTTPS URL, that Ghost runs on MySQL, and that a staff invite or password-reset email arrives.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

The Ghost docs list three options as required in production: `url`, `database` and `mail`. Any config key can be set as an environment variable, with `__` separating nested keys (`database__connection__host`). The official image defaults to `NODE_ENV=production`. Its docs say production needs MySQL 8, HTTPS and a reverse proxy that sends `X-Forwarded-*` headers. The pieces map like this:

- **MySQL 8 addon** for `database`. SQLite only works in development mode.
- **A volume** at `/var/lib/ghost/content` for uploaded images, themes and logs.
- **`url`** set to the public HTTPS address, so links, redirects and emails are correct.
- **`mail__*`** pointing at an SMTP provider. Ghost emails a sign-in code when an admin logs in from a new device, so without mail you can lock yourself out.

Those requirements decide where to deploy Ghost: it's a long-running Node.js server, not a static site, so static and serverless hosts can't run it. A raw VM works with Ghost's own install tooling, but you look after the OS, reverse proxy, TLS and MySQL backups. A container platform with a managed MySQL database runs the official Docker image and handles HTTPS and the database for you.

If you'd rather click through it, Northflank has a [Ghost stack template](https://northflank.com/stacks/deploy-ghost) with the same pieces: a Ghost deployment, a volume, a MySQL addon and a secret group.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **SMTP credentials** (Mailgun, Amazon SES, Postmark or any SMTP provider) exported as `$SMTP_HOST`, `$SMTP_USER`, `$SMTP_PASS` and a sender address `$MAIL_FROM`. **This needs a human:** the user creates them in the provider's dashboard and passes them through environment variables, not chat or committed files.

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

Before changing anything, print the deployment target and check that the SMTP variables are set (this prints `set`, never the values):

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId ghost -o json >/dev/null 2>&1 \
  && echo "project: ghost exists and will be reused" \
  || echo "project: ghost will be created"
echo "smtp:    host=${SMTP_HOST:+set} user=${SMTP_USER:+set} pass=${SMTP_PASS:+set} from=${MAIL_FROM:+set}"
```

- If any SMTP field is blank, export it first, or read the mail caveat in step 5 before going on without mail.
- Ghost is a Node.js app. `nf-compute-100-1` (1 vCPU, 1 GB) is a reasonable start; list plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | "\(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'`.

Names used below: project `ghost`, addon `ghost-db`, secret group `ghost-env`, service `ghost`, volume `ghost-content`.

**If you're an agent:** show the user the team, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "ghost", "region": "europe-west" }'
export PROJECT_ID=ghost
```

## Step 4: Create MySQL 8 and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "ghost-db", "type": "mysql", "version": "8-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "ghost" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId ghost-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

Ghost requires MySQL 8, so pin the major as `8-latest`. The database is only reachable inside the project, so TLS is off to match Ghost's default connection settings. `dbName`, TLS and high availability are fixed at creation, and storage can only be increased.

## Step 5: Create the secret group

Print the credential **key names** first (never the values):

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId ghost-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

The MySQL docs list `HOST`, `PORT`, `USERNAME`, `PASSWORD` and `DATABASE`. If the output uses different casing, change the `keyName` values below.

```bash
jq -n --arg h "$SMTP_HOST" --arg u "$SMTP_USER" --arg p "$SMTP_PASS" --arg f "$MAIL_FROM" '{
  name: "ghost-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "ghost-db", keys: [
    { keyName: "HOST",     aliases: ["database__connection__host"] },
    { keyName: "PORT",     aliases: ["database__connection__port"] },
    { keyName: "DATABASE", aliases: ["database__connection__database"] },
    { keyName: "USERNAME", aliases: ["database__connection__user"] },
    { keyName: "PASSWORD", aliases: ["database__connection__password"] }
  ] } ],
  secrets: { variables: {
    database__client: "mysql",
    mail__transport: "SMTP", mail__from: $f,
    mail__options__host: $h, mail__options__port: "465", mail__options__secure: "true",
    mail__options__auth__user: $u, mail__options__auth__pass: $p
  } }
}' > ghost-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file ghost-env.json >/dev/null && rm ghost-env.json
```

Port 465 with `secure: true` matches the Mailgun and SES examples in the Ghost docs. For port 587 (STARTTLS), use `"587"` and `"false"`.

**Mail caveat.** This SMTP setup covers transactional email: staff invites, password resets, device sign-in codes and member magic links. Ghost sends **bulk newsletters only through Mailgun**, which you configure later in Ghost Admin under Settings. If you have no SMTP provider yet, add `security__staffDeviceVerification: "false"` so you aren't locked out. Leave out the `mail__*` keys until you have one, and remove that line as soon as mail works.

## Step 6: Deploy the Ghost service

Write `ghost.json`:

```json
{
  "name": "ghost",
  "billing": { "deploymentPlan": "nf-compute-100-1" },
  "deployment": {
    "instances": 1,
    "external": { "imagePath": "ghost:6-alpine" },
    "docker": { "configType": "default" }
  },
  "ports": [ { "name": "http", "internalPort": 2368, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "TCP", "type": "readinessProbe", "port": 2368,
      "initialDelaySeconds": 15, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 12, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file ghost.json
```

`ghost:6-alpine` is the tag used in the image's Compose example. Pin a full version for controlled upgrades. Don't use the `next` tags: they move content to `/home/ghost/content`. The first start runs database migrations, so the readiness probe allows about two minutes.

## Step 7: Attach the content volume

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId ghost-content --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "ghost-content",
  "mounts": [ { "containerMountPath": "/var/lib/ghost/content" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "ghost", "type": "service" } ]
}'
```

Attaching redeploys the service. A single read/write volume limits the service to one instance (Ghost runs as a single process anyway), and on restart the old container stops before the new one starts. Volume size can only be increased.

## Step 8: Set `url` to the public address

```bash
for i in $(seq 1 30); do
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId ghost --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId ghost --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  [ -n "$HOST" ] && break; sleep 10
done; [ -n "$HOST" ] && echo "$HOST" || { echo "NO_PUBLIC_HOST: stop here and do not run the patch below"; false; }

northflank patch service deployment --projectId "$PROJECT_ID" --serviceId ghost --noDefaults -o json \
  --input "$(jq -n --arg h "$HOST" '{ runtimeEnvironment: { url: ("https://" + $h) } }')"
northflank restart service --projectId "$PROJECT_ID" --serviceId ghost --noDefaults -o json

OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId ghost --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"; [ "$D" = "COMPLETED" ] && { OK=1; break; }; [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId ghost`. `runtimeEnvironment` is sent as a complete map. When you add a custom domain, repeat this step with that hostname.

**This step needs a human:** open `https://$HOST/ghost/` right away and create the owner account. Until someone does, anyone with the URL can claim the site.

## How to know it worked

You're done when all of these hold:

1. The homepage answers and its canonical link uses the public HTTPS URL. This prints `CANONICAL_OK`:
   ```bash
   curl -s "https://$HOST/" | grep -o '<link rel="canonical" href="[^"]*"' | grep -qF "href=\"https://$HOST/" && echo CANONICAL_OK
   ```
2. Ghost is on MySQL and has the right `url`. This prints no secrets (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId ghost \
     --shell-cmd 'sh -c' --cmd '"printenv url database__client; ls /var/lib/ghost/content"'
   ```
   Expect `https://<HOST>`, `mysql` and folders such as `images` and `themes`.
3. Mail works: from Ghost Admin, invite a staff user or use "Forgot password" and confirm the email arrives. Then check the logs:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId ghost --lineLimit 300 --textIncludes "mail"
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ER_ACCESS_DENIED_ERROR` or `ECONNREFUSED` in logs | `database__connection__*` not injected | Re-check key names from step 5; restart |
| Too many redirects on every page | Ghost doesn't see the request as HTTPS | Confirm `url` is `https://`; check the load balancer passes `X-Forwarded-Proto` |
| Links point to `localhost:2368` | `url` not set, or service not restarted | Re-run step 8 |
| "Failed to send email" | Wrong SMTP host, port or `secure` value | Fix the `mail__*` values; try 587 with `secure: false` |
| Locked out with a device code you never receive | No working mail | Set `security__staffDeviceVerification=false` temporarily; fix mail |
| Images gone after redeploy | Volume not attached at `/var/lib/ghost/content` | Check `attachedObjects` with `northflank get volume` |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- Don't report success until the checks above pass, including the canonical URL check.
- If a wait loop prints `ROLLOUT_NOT_COMPLETE` or `ADDON_NOT_READY`, read the logs before changing anything.
- Never print SMTP or database passwords, or the credentials response. Print key names only.
- Hand the owner-account step to the user immediately after the first rollout.
- The image docs say to update to the latest minor version before a major upgrade. Back up the database and the content volume first (see [volume backups](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes)).
- Get explicit confirmation before deleting the addon, volume or service.

## FAQ

### Can I use SQLite to keep it simple?
Not in production mode. The image docs say SQLite is only valid in development mode and production needs MySQL 8.

### Do I need Mailgun?
Only for sending newsletters. Transactional email works with any SMTP provider.

### Can I store images in S3 instead of a volume?
The Ghost config docs describe a built-in S3-compatible storage adapter. It needs extra configuration, and themes still live in the content folder, so this guide keeps the volume.

### Where should I deploy my Ghost blog?
On a host that keeps a Node.js container running next to a MySQL 8 database, with persistent storage for `content` and HTTPS in front. Static and serverless hosts don't fit. A VM works if you're comfortable managing the server; a container platform with managed MySQL, as in this guide, runs the official Docker image without that upkeep.

## Conclusion

You now have Ghost on the official image with managed MySQL 8, a persistent content volume, the correct public `url`, and SMTP for sign-in and invites.

**Next:** [link a custom domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) and repeat step 8 with it, then connect Mailgun in Ghost Admin to send newsletters. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a MySQL database](deploy-mysql-database.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
