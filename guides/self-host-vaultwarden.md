<!--
agent-guide-meta:
  id: NF-STK-VAULTWARDEN
  description: "How to self-host Vaultwarden with the official Docker image, a /data volume, DOMAIN set to its HTTPS URL, an Argon2-hashed ADMIN_TOKEN and sign-ups disabled."
  sources:
    - https://github.com/dani-garcia/vaultwarden/wiki
    - https://github.com/dani-garcia/vaultwarden/wiki/Using-Docker-Compose
    - https://github.com/dani-garcia/vaultwarden/wiki/Enabling-admin-page
    - https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS
    - https://github.com/dani-garcia/vaultwarden/wiki/Disable-registration-of-new-users
    - https://github.com/dani-garcia/vaultwarden/wiki/Disable-invitations
    - https://github.com/dani-garcia/vaultwarden/blob/main/docker/Dockerfile.debian
    - https://github.com/dani-garcia/vaultwarden/blob/main/docker/healthcheck.sh
    - https://github.com/dani-garcia/vaultwarden/blob/main/src/api/core/mod.rs
    - https://hub.docker.com/r/vaultwarden/server/tags
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes
    - https://northflank.com/docs/v1/application/secure/inject-secrets
    - https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port
  related: [NF-DEP-002, NF-STK-N8N, NF-STK-GITEA, NF-STK-KEYCLOAK]
-->

# How to self-host Vaultwarden

To self-host Vaultwarden, run the official `vaultwarden/server` Docker image with `/data` on a persistent volume, put it behind HTTPS and set `DOMAIN` to that `https://` URL, protect the admin page with an Argon2-hashed `ADMIN_TOKEN`, and set `SIGNUPS_ALLOWED=false` so strangers can't register. The web vault uses browser crypto APIs that only work over HTTPS, and sign-ups are open by default.

This guide does it on Northflank with the CLI: a secret group holding the hashed admin token, a `vaultwarden/server` service with sign-ups off, a volume for `/data` and the automatic HTTPS hostname as `DOMAIN`. You end with a password vault on a public HTTPS URL, verified through `/alive` and `/api/config`. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human keeps the admin password and creates the first account.

## TL;DR: how to self-host Vaultwarden

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team and project, then create the `vaultwarden` project.
3. Generate an admin password, hash it with `argon2` into a `$argon2id$…` string, and store it as `ADMIN_TOKEN` in a secret group.
4. Run `vaultwarden/server:<version>` on port 80 with `SIGNUPS_ALLOWED=false` and a volume at `/data`.
5. Wait for the rollout, read the public host and set `DOMAIN=https://<host>`.
6. Verify that `/alive` returns 200, `/api/config` shows your domain and `ADMIN_TOKEN` starts with `$argon2id$`, then invite the first user from `/admin`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Vaultwarden is a lightweight server compatible with the Bitwarden clients. Its image listens on port 80 (`ROCKET_PORT=80`), keeps everything (the SQLite database, attachments, keys and `config.json`) in `/data`, and runs a health check against `/alive`. The wiki pages define the hardening:

- **HTTPS.** "The Bitwarden web vault uses web crypto APIs that most browsers only make available in HTTPS contexts." The recommended setup is a reverse proxy that terminates TLS, which is what a platform load balancer does.
- **`DOMAIN`.** The compose example marks it "required when using a reverse proxy". It's the URL that links and clients use.
- **Hashed `ADMIN_TOKEN`.** Setting `ADMIN_TOKEN` enables `/admin`. Store an Argon2 PHC string, not the plain password; you log in with the original password.
- **`SIGNUPS_ALLOWED=false`.** "By default, anyone who can access your instance can register for a new account." With sign-ups off, the admin can still invite users from the admin page.
- **One instance, one volume.** With SQLite, a single instance and a single read/write volume are the right fit.

So where to host Vaultwarden? Anywhere that runs one small container permanently with a persistent disk and a valid HTTPS certificate. A home server works if you manage certificates and remote access; a VM works with a proxy such as Caddy. A container platform that provides TLS and volumes removes both chores. Serverless platforms don't fit, because the database is a file on disk.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The `argon2` CLI and `openssl`.** `argon2` comes from the `argon2` package (`brew install argon2`, `apt install argon2`). Check with `command -v argon2 openssl jq`.
- **A pinned Vaultwarden version:**
  ```bash
  VW_VERSION=$(curl -s "https://hub.docker.com/v2/repositories/vaultwarden/server/tags?page_size=100&ordering=last_updated" \
    | jq -r '.results[].name' | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -n1); echo "$VW_VERSION"
  ```

Names used below: project `vaultwarden`, secret group `vaultwarden-env`, service `vaultwarden`, volume `vaultwarden-data`.

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

Before changing anything, print the deployment target:

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "image:   vaultwarden/server:$VW_VERSION"
northflank get project --projectId vaultwarden -o json >/dev/null 2>&1 \
  && echo "project: vaultwarden exists and will be reused" \
  || echo "project: vaultwarden will be created"
```

If `image` shows no version after the colon, re-run the version command from the prerequisites.

**If you're an agent:** show the user the team, the image version, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "vaultwarden", "region": "europe-west" }'
export PROJECT_ID=vaultwarden
```

## Step 4: Generate and hash the admin password

This follows the wiki's `argon2` command with Bitwarden's default parameters. The plain password goes to a file only the user can read, never to the terminal:

```bash
umask 077
openssl rand -base64 32 | tr -d '\n' > vaultwarden-admin-password.txt
ADMIN_TOKEN_HASH=$(tr -d '\n' < vaultwarden-admin-password.txt | argon2 "$(openssl rand -base64 32)" -e -id -k 65540 -t 3 -p 4)
echo "$ADMIN_TOKEN_HASH" | cut -c1-10    # $argon2id$
```

**Tell the user to move the password in `vaultwarden-admin-password.txt` into their password manager and then delete the file.** It's what they type on `/admin`.

## Step 5: Store the hash in a secret group

```bash
jq -n --arg t "$ADMIN_TOKEN_HASH" '{
  name: "vaultwarden-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  secrets: { variables: { ADMIN_TOKEN: $t } }
}' > vaultwarden-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file vaultwarden-env.json >/dev/null && rm vaultwarden-env.json
unset ADMIN_TOKEN_HASH
```

Use single `$` signs, as the wiki says for the Docker CLI and env files. The `$$` escaping is only for docker-compose `environment:` blocks.

## Step 6: Deploy Vaultwarden and attach `/data`

```bash
jq -n --arg img "vaultwarden/server:$VW_VERSION" '{
  name: "vaultwarden",
  billing: { deploymentPlan: "nf-compute-20" },
  deployment: { instances: 1, external: { imagePath: $img }, docker: { configType: "default" } },
  ports: [ { name: "http", internalPort: 80, public: true, protocol: "HTTP" } ],
  runtimeEnvironment: { SIGNUPS_ALLOWED: "false", INVITATIONS_ALLOWED: "true" },
  healthChecks: [
    { protocol: "HTTP", type: "readinessProbe", path: "/alive", port: 80,
      initialDelaySeconds: 5, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 6, successThreshold: 1 }
  ]
}' > vaultwarden.json
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file vaultwarden.json

northflank get volume --projectId "$PROJECT_ID" --volumeId vaultwarden-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "vaultwarden-data",
  "mounts": [ { "containerMountPath": "/data" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 2048 },
  "attachedObjects": [ { "id": "vaultwarden", "type": "service" } ]
}'
```

The image runs as root, so the volume is writable without extra steps. `nf-compute-20` (0.2 vCPU, 512 MB) is enough for a household or small team. Attaching the volume redeploys the service.

## Step 7: Set `DOMAIN` to the HTTPS URL

This waits for the first rollout, reads the public host, sets `DOMAIN` to it and waits for the restart:

```bash
wait_rollout() {
  OK=0
  for i in $(seq 1 60); do
    D=$(northflank get service --projectId "$PROJECT_ID" --serviceId vaultwarden --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
    echo "deployment=$D"
    [ "$D" = "COMPLETED" ] && { OK=1; break; }
    [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
    sleep 10
  done
  [ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; return 1; }
}

wait_rollout
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId vaultwarden --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId vaultwarden --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi

[ -n "$HOST" ] && northflank patch service deployment --projectId "$PROJECT_ID" --serviceId vaultwarden --noDefaults -o json --input "$(jq -n --arg h "$HOST" '{
  runtimeEnvironment: { SIGNUPS_ALLOWED: "false", INVITATIONS_ALLOWED: "true", DOMAIN: ("https://" + $h) }
}')" \
  && northflank restart service --projectId "$PROJECT_ID" --serviceId vaultwarden --noDefaults -o json \
  && wait_rollout
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId vaultwarden`. The patch sends the complete map. When you add a custom domain, [link it to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) and repeat this step with it, because clients store the server URL.

## Step 8: Create the first account

**This step needs a human:** open `https://$HOST/admin`, sign in with the admin password from step 4, go to **Users** and invite your own email address. Without SMTP configured, the wiki says the invited user registers at `https://$HOST/#/signup` with that exact address. Then sign in to the web vault and point the Bitwarden apps' **self-hosted** server URL at `https://$HOST`.

## How to know it worked

You're done when all of these hold:

1. The server is up:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/alive"     # 200
   ```
2. Vaultwarden knows its public URL:
   ```bash
   curl -s "https://$HOST/api/config" | jq -r '.environment.vault'     # https://<HOST>
   ```
3. The admin token arrived as a hash, and sign-ups are off. Only prefixes are printed (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId vaultwarden \
     --shell-cmd 'sh -c' --cmd '"printenv SIGNUPS_ALLOWED DOMAIN; printenv ADMIN_TOKEN | cut -c1-10"'
   ```
   Expect `false`, `https://<HOST>` and `$argon2id$`. The startup logs must not show `[NOTICE] You are using a plain text ADMIN_TOKEN which is insecure`; a mangled hash fails startup with `The configured Argon2 PHC in ADMIN_TOKEN is invalid`.
4. Data is on the volume: after the first account exists, restart the service, wait for `COMPLETED`, and sign in again. Also:
   ```bash
   northflank get volume --projectId "$PROJECT_ID" --volumeId vaultwarden-data --noDefaults -o json | jq '(.data // .) | {status, attachedObjects}'
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Web vault shows a crypto or `subtle` error | Opened over `http://` | Use the `https://` URL |
| Links in emails or the admin page show the wrong host | `DOMAIN` unset or old | Re-run step 7 |
| Plain-text token notice in the logs, or an invalid Argon2 PHC error at start | Token stored unhashed, with `$$`, or with quotes | Redo steps 4–5; check the prefix with check 3 |
| Env changes ignored | `/data/config.json` overrides env once you save settings in the admin page | Change the setting in the admin page, or remove the key from `config.json` |
| `/admin` returns 404 | `ADMIN_TOKEN` not injected | Check the secret group; restart |
| All accounts gone after a deploy | Volume not mounted at `/data` | Mount at `/data` exactly |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Never print the admin password, the full `ADMIN_TOKEN` or the contents of `/data`. Never read `vaultwarden-admin-password.txt` back into the conversation.
- Never set `SIGNUPS_ALLOWED=true` unless the user asks, and switch it back afterwards.
- This vault will hold every password the user has. Get explicit confirmation before deleting or detaching the volume, and suggest [volume backups](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes) right after setup.
- Don't scale past one instance; the SQLite database and the single read/write volume don't support it.

## FAQ

### Where should I host Vaultwarden?
On a host that runs one always-on container with a persistent disk and a valid HTTPS certificate. A container platform with volumes and automatic TLS is the least work; a home server or VM works if you run a reverse proxy like Caddy and keep certificates current.

### Is self-hosting Vaultwarden safe?
The server only stores encrypted vault data; decryption happens in clients. The risks are an exposed admin page, open sign-ups and lost backups, which this guide addresses with a hashed token, `SIGNUPS_ALLOWED=false` and a volume you can back up.

### Do I need PostgreSQL?
No. Vaultwarden defaults to SQLite in `/data`, which suits a single instance. It also supports PostgreSQL through `DATABASE_URL` if you outgrow it.

### Do I need SMTP?
Not for a small setup: invited users can register at `/#/signup`. Configure SMTP if you want email verification, invitations by email or 2FA by email.

## Conclusion

You now have Vaultwarden on the official image, served over HTTPS with `DOMAIN` set, data on a persistent volume, the admin page behind an Argon2-hashed token and registration closed to strangers.

**Next:** add your own domain, schedule volume backups, and configure SMTP for invitations. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Docker image](deploy-docker-image.md)
- [How to self-host n8n](self-host-n8n.md)
- [How to self-host Gitea](self-host-gitea.md)
- [How to deploy Keycloak](deploy-keycloak.md)
