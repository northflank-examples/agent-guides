<!--
agent-guide-meta:
  id: NF-STK-GITEA
  description: "How to self-host Gitea with the official Docker image, managed PostgreSQL, a /data volume, ROOT_URL set to its HTTPS address and SSH via a TCP load balancer."
  sources:
    - https://docs.gitea.com/installation/install-with-docker
    - https://docs.gitea.com/installation/install-with-docker-rootless
    - https://docs.gitea.com/administration/config-cheat-sheet
    - https://docs.gitea.com/administration/command-line
    - https://docs.gitea.com/1.19/installation/install-on-kubernetes
    - https://github.com/go-gitea/gitea/blob/main/models/db/conn.go
    - https://github.com/go-gitea/gitea/blob/main/cmd/admin_user_create.go
    - https://hub.docker.com/r/gitea/gitea/tags
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/configure-load-balancers
    - https://northflank.com/docs/v1/api/team/load-balancers/put-load-balancer
    - https://northflank.com/docs/v1/api/team/load-balancers/get-load-balancer
  related: [NF-DB-001, NF-DEP-002, NF-STK-KEYCLOAK, NF-STK-N8N]
-->

# How to self-host Gitea

To self-host Gitea, run the official `gitea/gitea` Docker image against PostgreSQL through `GITEA__database__*` variables, persist `/data` (repositories, LFS, `app.ini`) on a volume, set `ROOT_URL` and `DOMAIN` to the public HTTPS address, and lock the installer with a fixed `SECRET_KEY`. Git over SSH needs a raw TCP port in addition to HTTPS, which many hosts can't expose by default, so plan clone URLs before you invite users.

This guide does it on Northflank with the CLI: a managed PostgreSQL addon, a secret group with the database credentials and secret key, a `gitea/gitea` service and a volume for `/data`. You end with Gitea on a public HTTPS URL with HTTPS cloning, verified through `/api/healthz` and its OpenID issuer, plus an optional SSH path. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human creates the first admin.

## TL;DR: how to self-host Gitea

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Pick a Gitea version and confirm the team and project before changing anything.
3. Create a project, a PostgreSQL addon and a secret group that links the database as `GITEA__database__*` with `INSTALL_LOCK=true` and a generated `SECRET_KEY`.
4. Run `gitea/gitea:<version>` on port 3000 with a volume at `/data`, then set `ROOT_URL` and `DOMAIN` to its public HTTPS address.
5. Create the admin with `gitea admin user create` through `northflank exec`.
6. Verify that `/api/healthz` returns `pass` and the OpenID issuer is your `https://` URL.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

The Gitea Docker docs configure everything through `GITEA__<section>__<KEY>` environment variables, which the image writes into `app.ini` under `/data` on start. The pieces:

- **PostgreSQL.** `GITEA__database__DB_TYPE=postgres` with `HOST`, `NAME`, `USER` and `PASSWD`. Gitea's source falls back to port 5432 when `HOST` has no port.
- **`/data` on a volume.** Repositories, attachments, LFS objects, avatars and the generated `app.ini` live there. Without a volume, every deploy loses your code.
- **`ROOT_URL` and `DOMAIN`.** `ROOT_URL` defaults to `http://localhost:3000/`, which ends up in clone URLs, emails and OAuth redirects. Behind a proxy you set it to the public URL.
- **`INSTALL_LOCK` and `SECRET_KEY`.** The lock skips the web installer, so nobody who finds the URL first can configure your instance. The cheat sheet warns that data encrypted with `SECRET_KEY` (like 2FA secrets) can't be decrypted if the key is lost.
- **SSH caveat.** Gitea's SSH server listens on TCP port 22 inside the container. On Northflank, only HTTP and HTTP/2 ports can be public on the service itself; TCP needs a separate Layer 4 load balancer, which the CLI creates with `northflank put load-balancer`. So this guide starts with HTTPS cloning and `DISABLE_SSH=true`, and adds SSH as an option.

Where to host Gitea comes down to a long-running container, a database, a persistent disk and (for SSH) a TCP port. Serverless hosts can't keep Git repositories on disk. A VM handles everything, including port 22, but you run backups, TLS and upgrades. A container platform with managed PostgreSQL and volumes covers HTTPS hosting directly, and SSH with a TCP load balancer.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`**, to generate Gitea's secret key.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Pick a version and confirm where it will deploy

Pin a Gitea version. This lists the newest plain version tag on Docker Hub, then prints the target:

```bash
GITEA_VERSION=$(curl -s "https://hub.docker.com/v2/repositories/gitea/gitea/tags?page_size=100&ordering=last_updated" \
  | jq -r '.results[].name' | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -n1)
echo "gitea:   ${GITEA_VERSION:-NOT FOUND}"
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId gitea -o json >/dev/null 2>&1 \
  && echo "project: gitea exists and will be reused" \
  || echo "project: gitea will be created"
```

- If the version prints `NOT FOUND`, pick a tag from [Docker Hub](https://hub.docker.com/r/gitea/gitea/tags) and set `GITEA_VERSION` by hand.
- Names used below: addon `gitea-db`, secret group `gitea-env`, service `gitea`, volume `gitea-data`.

**If you're an agent:** show the user the team, the Gitea version, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create a project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "gitea", "region": "europe-west" }'
export PROJECT_ID=gitea
```

## Step 4: Create PostgreSQL and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "gitea-db", "type": "postgresql", "version": "16-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "gitea" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId gitea-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon status and logs"; false; }
```

TLS is off to match Gitea's default `SSL_MODE=disable`; the database only accepts traffic from inside the project. TLS and `dbName` are fixed at creation.

## Step 5: Create the secret group

Print the credential key **names** first:

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId gitea-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

Adjust the `keyName` values below if the casing differs from `HOST`, `DATABASE`, `USERNAME`, `PASSWORD`. Then:

```bash
jq -n --arg sk "$(openssl rand -hex 32)" '{
  name: "gitea-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "gitea-db", keys: [
    { keyName: "HOST",     aliases: ["GITEA__database__HOST"] },
    { keyName: "DATABASE", aliases: ["GITEA__database__NAME"] },
    { keyName: "USERNAME", aliases: ["GITEA__database__USER"] },
    { keyName: "PASSWORD", aliases: ["GITEA__database__PASSWD"] }
  ] } ],
  secrets: { variables: {
    GITEA__database__DB_TYPE: "postgres", GITEA__database__SSL_MODE: "disable",
    GITEA__security__INSTALL_LOCK: "true", GITEA__security__SECRET_KEY: $sk
  } }
}' > gitea-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file gitea-env.json >/dev/null && rm gitea-env.json
```

`INTERNAL_TOKEN` isn't set here: with `INSTALL_LOCK=true` and no token, Gitea generates one on first start and saves it to `app.ini` on the `/data` volume, so it survives redeploys.

**Tell the user to copy `GITEA__security__SECRET_KEY` from the secret group into their password manager.**

## Step 6: Deploy Gitea and attach `/data`

```bash
jq -n --arg img "gitea/gitea:$GITEA_VERSION" '{
  name: "gitea",
  billing: { deploymentPlan: "nf-compute-100-2" },
  deployment: { instances: 1, external: { imagePath: $img }, docker: { configType: "default" } },
  ports: [ { name: "http", internalPort: 3000, public: true, protocol: "HTTP" } ],
  runtimeEnvironment: { GITEA__server__HTTP_PORT: "3000", GITEA__server__DISABLE_SSH: "true", GITEA__service__DISABLE_REGISTRATION: "true" },
  healthChecks: [
    { protocol: "HTTP", type: "readinessProbe", path: "/api/healthz", port: 3000,
      initialDelaySeconds: 10, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 6, successThreshold: 1 }
  ]
}' > gitea.json
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file gitea.json

northflank get volume --projectId "$PROJECT_ID" --volumeId gitea-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "gitea-data",
  "mounts": [ { "containerMountPath": "/data" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "gitea", "type": "service" } ]
}'
```

Notes:

- The standard (rootful) image starts as root and sets up `/data` for the `git` user (UID 1000), which avoids the volume permission problems the rootless docs describe.
- `/api/healthz` is the health endpoint Gitea's Kubernetes docs probe; it returns `pass` once the database and cache respond.
- `DISABLE_REGISTRATION=true` means only admins create accounts. Size the volume for your repositories and LFS; it can grow but never shrink, and it limits the service to one instance.

## Step 7: Wait for the rollout and get your URL

The first rollout takes a minute or two. This loop checks every 10 seconds and prints your URL when Gitea is live:

```bash
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId gitea --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId gitea --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId gitea --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId gitea`. The troubleshooting table covers the common causes.

## Step 8: Set `ROOT_URL` and `DOMAIN`

Patch the complete `runtimeEnvironment` map with the host from step 7, and restart:

```bash
northflank patch service deployment --projectId "$PROJECT_ID" --serviceId gitea --noDefaults -o json --input "$(jq -n --arg h "$HOST" '{
  runtimeEnvironment: {
    GITEA__server__HTTP_PORT: "3000", GITEA__server__DISABLE_SSH: "true", GITEA__service__DISABLE_REGISTRATION: "true",
    GITEA__server__DOMAIN: $h, GITEA__server__ROOT_URL: ("https://" + $h + "/")
  }
}')"
northflank restart service --projectId "$PROJECT_ID" --serviceId gitea --noDefaults -o json
```

Then run the step 7 loop again and wait for `COMPLETED`. Repeat this step with your own domain when you add one.

## Step 9: Create the admin

**This step needs a human:** the command prints a generated password once, so the user should run it in their own terminal and store the password. Linux: `script -qefc '<cmd>' /dev/null`.

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId gitea --user git \
  --shell-cmd 'sh -c' --cmd '"gitea admin user create --admin --username <ADMIN_USER> --email <ADMIN_EMAIL> --random-password --must-change-password"'
```

The output ends with `New user '<ADMIN_USER>' has been successfully created!`. Gitea asks for a new password at first login.

## Optional: Git over SSH

Clone over HTTPS works with no extra setup. For `git@` SSH clones, Gitea's OpenSSH on container port 22 needs a public TCP route, which comes from a team-level Layer 4 load balancer with its own public IP. An API token needs the `Account > Networking > LoadBalancers` create and read permissions (check with `northflank list load-balancers -o json >/dev/null && echo LB_OK`).

1. Add a private TCP port for 22 and turn SSH on (the full `runtimeEnvironment` map again):
   ```bash
   northflank patch service deployment --projectId "$PROJECT_ID" --serviceId gitea --noDefaults -o json --input "$(jq -n --arg h "$HOST" '{
     ports: [ { name: "http", internalPort: 3000, public: true, protocol: "HTTP" },
              { name: "ssh", internalPort: 22, public: false, protocol: "TCP" } ],
     runtimeEnvironment: {
       GITEA__server__HTTP_PORT: "3000", GITEA__service__DISABLE_REGISTRATION: "true",
       GITEA__server__DOMAIN: $h, GITEA__server__ROOT_URL: ("https://" + $h + "/"),
       GITEA__server__DISABLE_SSH: "false"
     }
   }')"
   ```
2. Create a public TCP load balancer on port 22 in the project's region, with the `gitea` service as backend (`<projectId>/<serviceId>`), and wait for its IP:
   ```bash
   REGION=$(northflank get project --projectId "$PROJECT_ID" --noDefaults -o json | jq -r '(.data // .).deployment.region')
   jq -n --arg r "$REGION" --arg b "$PROJECT_ID/gitea" '{
     name: "gitea-ssh", description: "Public TCP endpoint for Git over SSH",
     spec: { type: "tcp", visibility: "public", target: { type: "region", targetId: $r },
       ports: [ { id: "port-22", port: "22",
         backends: [ { id: $b, type: "service", port: 22, weight: 100 } ] } ] }
   }' > gitea-ssh.json
   northflank put load-balancer --noDefaults -o json --file gitea-ssh.json

   OK=0
   for i in $(seq 1 60); do
     LB=$(northflank get load-balancer --loadBalancerId gitea-ssh --noDefaults -o json)
     S=$(echo "$LB" | jq -r '(.data // .).state.status // "NONE"'); echo "load-balancer=$S"
     [ "$S" = "created" ] && { OK=1; break; }
     [ "$S" = "error" ] && echo "LB_ERROR (the platform retries automatically)"
     sleep 10
   done
   [ "$OK" = 1 ] || { echo "LOAD_BALANCER_NOT_CREATED: stop here and check the load balancer in the UI"; false; }
   [ "$OK" = 1 ] && SSH_HOST=$(echo "$LB" | jq -r '(.data // .).state.endpoints[0] // empty') && echo "$SSH_HOST"
   ```
   Provisioning takes a few minutes. Ports 30000–32767 are reserved, and updating a load balancer re-provisions it with a short interruption. See [configure load balancers](https://northflank.com/docs/v1/application/network/configure-load-balancers).
3. Put the address into SSH clone URLs. Use `SSH_HOST`, or a DNS A record such as `git.example.com` that you point at it:
   ```bash
   northflank patch service deployment --projectId "$PROJECT_ID" --serviceId gitea --noDefaults -o json --input "$(jq -n --arg h "$HOST" --arg s "$SSH_HOST" '{
     runtimeEnvironment: {
       GITEA__server__HTTP_PORT: "3000", GITEA__service__DISABLE_REGISTRATION: "true",
       GITEA__server__DOMAIN: $h, GITEA__server__ROOT_URL: ("https://" + $h + "/"),
       GITEA__server__DISABLE_SSH: "false", GITEA__server__SSH_DOMAIN: $s
     }
   }')"
   northflank restart service --projectId "$PROJECT_ID" --serviceId gitea --noDefaults -o json
   ```
4. **This step needs a human:** add an SSH public key under **Settings → SSH / GPG Keys** in Gitea, then run `ssh -T git@$SSH_HOST`; Gitea answers with a greeting for that user.

The load balancer works at Layer 4 only, with no TLS termination, which suits SSH. Deleting it changes the IP in every SSH clone URL.

## How to know it worked

You're done when all of these hold:

1. Health passes, including the database:
   ```bash
   curl -s "https://$HOST/api/healthz" | jq -r '.status'    # pass
   ```
2. Gitea uses the public HTTPS URL. Its OpenID discovery document names it as issuer:
   ```bash
   curl -s "https://$HOST/.well-known/openid-configuration" | jq -r '.issuer'   # https://<HOST>/
   ```
3. Gitea is on PostgreSQL and the volume is attached:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId gitea \
     --shell-cmd 'sh -c' --cmd '"cd /data/gitea/conf; grep ^DB_TYPE app.ini; grep ^ROOT_URL app.ini; grep ^INSTALL_LOCK app.ini"'
   northflank get volume --projectId "$PROJECT_ID" --volumeId gitea-data --noDefaults -o json | jq '(.data // .) | {status, attachedObjects}'
   ```
   Expect `postgres`, `https://<HOST>/` and `true`.
4. After the admin exists, creating a repository and running `git clone https://$HOST/<ADMIN_USER>/<repo>.git` works.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Install page appears | `INSTALL_LOCK` not injected | Check the secret group; restart |
| `dial tcp 127.0.0.1:5432: connection refused` | `GITEA__database__HOST` empty | Check key names from step 5 |
| Clone URLs show `localhost:3000` | Step 8 not applied | Re-run step 8 |
| Repositories gone after a redeploy | Volume missing or mounted elsewhere | Mount at `/data` exactly |
| `Permission denied` under `/data` | Volume ownership | Restart once; if it persists, report it with the [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) docs |
| SSH connection refused or times out | Load balancer not `created`, wrong region, or wrong backend | Re-check the SSH step; the backend must be `<projectId>/gitea` on port 22 |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write: team, Gitea version, region and whether the project is new or reused.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Don't report success until checks 1 and 2 pass: `pass` from `/api/healthz` and your `https://` URL as the issuer.
- Never print `SECRET_KEY`, database passwords, generated admin passwords or the credentials response.
- Never change `SECRET_KEY` on a running instance.
- Leave the admin creation step to the human, or ask before running it, because the password appears in the output.
- Get explicit confirmation before deleting the volume, the addon, the service or the `gitea-ssh` load balancer; the volume holds every repository, and a new load balancer gets a new IP.

## FAQ

### Where should I host Gitea?
On a long-running container with PostgreSQL, a persistent volume for repositories, HTTPS in front and, if you want SSH, a TCP port. A container platform with managed databases and volumes does the HTTPS part out of the box, and a TCP load balancer adds SSH on port 22, as in this guide's optional step.

### Can I keep using SQLite?
Gitea supports it, but PostgreSQL gives you managed backups separate from the repository volume.

### Should I use the rootless image?
It works, but it listens for SSH on 2222, keeps data in `/var/lib/gitea` and `/etc/gitea`, and needs volumes owned by UID 1000. The standard image avoids the ownership issue.

### How do I upgrade Gitea?
Back up the database and volume, change the image tag with `northflank patch service deployment`, and wait for `COMPLETED`. Gitea migrates the database on start.

## Conclusion

You now have Gitea on the official image with PostgreSQL, repositories on a persistent volume, a locked installer, a fixed secret key and clone URLs that use your HTTPS address, with SSH available through a TCP load balancer.

**Next:** add single sign-on with [Keycloak](deploy-keycloak.md) as an OpenID Connect source, and set up volume backups. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Keycloak](deploy-keycloak.md)
- [How to self-host GitLab](self-host-gitlab.md)
- [How to deploy a Docker image](deploy-docker-image.md)
