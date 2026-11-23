<!--
agent-guide-meta:
  id: NF-STK-POCKETBASE
  description: "How to deploy PocketBase with the documented Dockerfile, a pb_data volume, one instance, a scripted superuser and a public HTTPS URL checked with curl."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/stacks/deploy-pocketbase
    - https://pocketbase.io/docs/
    - https://pocketbase.io/docs/going-to-production/
    - https://pocketbase.io/docs/api-health/
    - https://pocketbase.io/docs/api-records/
    - https://pocketbase.io/faq/
    - https://northflank.com/docs/v1/application/network/networking-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/execute-command
    - https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port
  related: [NF-DEP-001, NF-DEP-002, NF-STK-PAYLOAD, NF-STK-STRAPI, NF-DB-001]
-->

# How to deploy PocketBase

To deploy PocketBase, build the PocketBase binary into a small Docker image that runs `pocketbase serve --http=0.0.0.0:8080`, mount a persistent volume at `pb_data`, run exactly one instance behind an HTTPS proxy, and create the first superuser with `pocketbase superuser create`. PocketBase keeps everything in embedded SQLite, so a second instance or a lost `pb_data` directory means split or lost data.

This guide does it on Northflank with the CLI: a service built from PocketBase's documented Dockerfile, a public port with `/api/health` checks, a `pb_data` volume that pins it to one instance, and a superuser from a secret group. You end with a public URL where health and superuser login pass `curl` checks. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy PocketBase

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check the repository and confirm the team, repository, branch and project before changing anything, then give Northflank access to the repository (once).
3. Commit the Dockerfile from PocketBase's production docs and push.
4. Build and run it with `northflank put service combined` (one instance, public port 8080, `/api/health` probes), store the superuser email and a generated password with `northflank put secret`, and attach a volume at `/pb/pb_data`.
5. Wait for `COMPLETED`, then create the superuser with `script -q /dev/null northflank exec service … superuser create`.
6. Check that the live build is your pushed commit, `https://<HOST>/api/health` returns 200 and the superuser gets a token from `auth-with-password`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

PocketBase is one Go binary with an embedded SQLite database, auth, file storage and an admin dashboard. That makes it easy to run, with three hard requirements:

- **One instance, always on.** PocketBase's FAQ says it scales "only on a single server, aka. vertical", and it uses "embedded SQLite (in WAL mode)". Two instances would each write their own copy of the data. Serverless platforms that scale to zero or out to many copies don't fit.
- **A persistent `pb_data`.** The database, uploaded files and settings live in `pb_data`. The production docs say to mount a volume at `/pb/pb_data` when using Docker.
- **A first superuser.** On first start PocketBase prints an installer link to create the superuser. On a public URL, create it yourself straight away with `superuser create`.

Where to host PocketBase follows from that: a small always-on container with a persistent disk and HTTPS. A VPS works if you run the reverse proxy, TLS and backups yourself. A container platform runs the Docker image with a volume and a TLS endpoint, and a single read/write volume enforces the one-instance rule for you.

PocketBase has no official Docker image and isn't a Northflank addon type, so this builds the Dockerfile from its docs. If you'd rather click through it, Northflank has a [PocketBase stack template](https://northflank.com/stacks/deploy-pocketbase) that deploys PocketBase with a persistent volume from the UI.

## Prerequisites

- **A GitHub, GitLab or Bitbucket repository** that you can push to, to hold the PocketBase `Dockerfile` (a new, empty one is fine) plus any `pb_migrations` or `pb_hooks` folders.
- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output), **`openssl`** and **`curl`**.
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The superuser's email address** from the user, without spaces or quotes.

Names used below: project `pocketbase`, secret group `pb-admin`, service `pocketbase`, volume `pb-data`.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Check the repository and confirm where it will deploy

Before changing anything, confirm that this guide fits the repository and print the deployment target. Run this from the repository root:

```bash
ls Dockerfile go.mod package.json pb_migrations pb_hooks 2>/dev/null
grep -n 'pocketbase' Dockerfile 2>/dev/null || echo "Dockerfile: no PocketBase download (step 5 adds it)"
grep -q 'github.com/pocketbase/pocketbase' go.mod 2>/dev/null && echo "go.mod: PocketBase used as a Go framework"

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId pocketbase -o json >/dev/null 2>&1 \
  && echo "project: pocketbase exists and will be reused" \
  || echo "project: pocketbase will be created"
```

- If it prints `go.mod: PocketBase used as a Go framework`, this guide doesn't fit: its `Dockerfile` downloads the prebuilt binary and would leave out your Go code. Stop and use a guide that builds a Go app.
- If a `Dockerfile` exists but doesn't mention `pocketbase`, or there's a `package.json`, the repository holds another app. This isn't a PocketBase repo: stop and use a guide for your framework, or put PocketBase in its own repository.
- If `pb_migrations` or `pb_hooks` exist, uncomment the matching `COPY` lines in step 5.
- If a PocketBase `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "pocketbase", "region": "europe-west" }'
export PROJECT_ID=pocketbase

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Add the `Dockerfile` and push

Add this `Dockerfile` at the repository root. It's taken from PocketBase's [going to production](https://pocketbase.io/docs/going-to-production/) page (the docs pin `PB_VERSION=0.42.3`; check the page for a newer one). The download URL is the docs' own; the GitHub repository is archived and its releases stop at an older version, so don't swap in a GitHub release URL:

```dockerfile
FROM alpine:latest

ARG PB_VERSION=0.42.3

RUN apk add --no-cache \
    unzip \
    ca-certificates

# download and unzip PocketBase
ADD https://ganigeorgiev.com/meta/pocketbase/pocketbase/download/v${PB_VERSION}/pocketbase_${PB_VERSION}_linux_amd64.zip /tmp/pb.zip
RUN unzip /tmp/pb.zip -d /pb/

# uncomment to copy the local pb_migrations dir into the image
# COPY ./pb_migrations /pb/pb_migrations

# uncomment to copy the local pb_hooks dir into the image
# COPY ./pb_hooks /pb/pb_hooks

EXPOSE 8080

# start PocketBase
CMD ["/pb/pocketbase", "serve", "--http=0.0.0.0:8080"]
```

If you use JS hooks or migrations, uncomment those `COPY` lines and commit the folders too. Northflank builds what's in your repository, not the files on your machine, so commit and push:

```bash
git add Dockerfile
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Deploy the PocketBase service

Write `pocketbase.json`:

```json
{
  "name": "pocketbase",
  "billing": { "deploymentPlan": "nf-compute-50" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/api/health", "port": 8080,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 6, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/api/health", "port": 8080,
      "initialDelaySeconds": 30, "periodSeconds": 30, "timeoutSeconds": 3, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file pocketbase.json
```

Use the same repository URL, provider and branch as in step 4. `GET /api/health` returns `{"status":200,"message":"API is healthy."…}` according to the PocketBase API docs. `nf-compute-50` (0.5 vCPU, 1 GB) is plenty to start; PocketBase scales vertically, so raise the plan rather than the instance count. `billing.buildPlan` is optional; leave it out to use the default build plan, and only set it if builds run out of memory, using a build plan your team has access to (an unavailable one fails with `Build plan not found`). The first build starts automatically.

## Step 7: Store the superuser credentials

```bash
PB_ADMIN_EMAIL="<SUPERUSER_EMAIL>"
PB_ADMIN_PASSWORD=$(openssl rand -hex 16)
jq -n --arg e "$PB_ADMIN_EMAIL" --arg p "$PB_ADMIN_PASSWORD" '{
  name: "pb-admin", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "pocketbase", type: "service" } ] },
  secrets: { variables: { PB_ADMIN_EMAIL: $e, PB_ADMIN_PASSWORD: $p } }
}' > pb-admin.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file pb-admin.json >/dev/null && rm pb-admin.json
```

The group is restricted to the `pocketbase` service, which is why it comes after the service exists. The running container picks it up on its next restart, which the volume attachment in step 8 triggers. Keep `PB_ADMIN_PASSWORD` in this shell for the login check at the end, and never echo it. **Tell the user to copy the password from the secret group in the UI.**

## Step 8: Attach the `pb_data` volume

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId pb-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "pb-data",
  "mounts": [ { "containerMountPath": "/pb/pb_data" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 6144 },
  "attachedObjects": [ { "id": "pocketbase", "type": "service" } ]
}'
```

The Northflank volume docs say a single read/write volume limits the service to one instance and stops the old container before the new one starts on every restart. That's exactly what SQLite needs: two processes never write to `pb_data` at once. Expect a few seconds of downtime per deploy. Storage is in MB and can only grow.

## Step 9: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds and prints your URL when PocketBase is live. You can also watch the build in the Northflank dashboard.

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId pocketbase --noDefaults -o json)
  B=$(echo "$S" | jq -r '(.data // .).status.build.status // "NONE"')
  D=$(echo "$S" | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "build=$B deployment=$D"
  case "$B" in FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo BUILD_FAILED; break;; esac
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  [ "$B" = "SUCCESS" ] && [ "$D" = "COMPLETED" ] && { OK=1; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId pocketbase --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId pocketbase --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId pocketbase`. The troubleshooting table covers the common causes. Don't continue to step 10 until the URL prints.

## Step 10: Create the superuser

Run the documented `superuser create EMAIL PASS` inside the container. The single quotes keep `$PB_ADMIN_EMAIL` and `$PB_ADMIN_PASSWORD` unexpanded locally, so they come from the secret group inside the container and never appear on your command line. `exec` drops output without a TTY, so wrap it with `script` (macOS shown; on Linux use `script -qefc '<cmd>' /dev/null`):

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId pocketbase \
  --shell-cmd 'sh -c' --cmd '"/pb/pocketbase superuser create $PB_ADMIN_EMAIL $PB_ADMIN_PASSWORD"'
```

Run it once. If the superuser already exists, it reports an error and changes nothing. Creating it now closes the window in which anyone who found the installer link in the logs could claim the instance.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId pocketbase --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health endpoint answers over HTTPS:
   ```bash
   curl -s "https://$HOST/api/health" | jq -r '.message'   # API is healthy.
   ```
3. The superuser can log in. This prints only whether a token came back:
   ```bash
   jq -n --arg e "$PB_ADMIN_EMAIL" --arg p "$PB_ADMIN_PASSWORD" '{identity: $e, password: $p}' \
     | curl -s -X POST "https://$HOST/api/collections/_superusers/auth-with-password" \
         -H 'Content-Type: application/json' -d @- | jq -r 'if .token then "SUPERUSER_OK" else . end'
   ```
4. The dashboard loads at `https://$HOST/_/` and accepts the same login (**this step needs a human**).
5. Data survives a restart: `northflank restart service --projectId "$PROJECT_ID" --serviceId pocketbase --noDefaults -o json`, wait for `COMPLETED` with the step 9 loop, and check 3 still prints `SUPERUSER_OK`. Then run `unset PB_ADMIN_PASSWORD`.
6. The service runs one instance: `northflank get service --projectId "$PROJECT_ID" --serviceId pocketbase --noDefaults -o json | jq '(.data // .).deployment.instances'` prints `1`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |
| Build fails at `ADD https://…/pocketbase_…zip` | Version or URL changed | Copy the current Dockerfile from PocketBase's production docs |
| Superuser and data gone after a restart | Volume missing or not at `/pb/pb_data` | Check `attachedObjects` and the mount path |
| `superuser create` prints nothing | `exec` without a TTY | Wrap with `script` as shown |
| Superuser login returns 400 | Wrong email/password, or created earlier with another password | Reset it in the dashboard, or update the secret group |
| Logs show `database is locked` | Two processes writing `pb_data` | Keep `instances: 1` and the single read/write volume |
| `Too many open files` with many realtime clients | File descriptor limit | Raise the plan; report it if it persists |
| Rate limits or logs show the proxy's IP | Proxy headers not configured | In the dashboard, set **User IP proxy headers** (human step) |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print `PB_ADMIN_PASSWORD` or the auth token.
- Never set `instances` above 1 or switch the volume to multi read/write. PocketBase only scales vertically.
- Create the superuser right after the first rollout, before sharing the URL.
- Get explicit confirmation before deleting the `pb-data` volume or the service. Back up `pb_data` first.

## FAQ

### Where should I deploy PocketBase?
On a host that keeps one container running with a persistent disk and HTTPS in front. A small VPS works if you'll manage the reverse proxy, TLS and backups. A container platform, as here, builds the Dockerfile, mounts `pb_data` on a volume and terminates TLS for you. Serverless platforms that scale to zero or run many copies don't suit an embedded SQLite database.

### Can I deploy PocketBase for free?
Many platforms have free tiers, but PocketBase needs a persistent volume and an always-on container, which free tiers often limit. Check what yours includes before relying on it for real data.

### How do I back up PocketBase?
Use the built-in backups under **Settings > Backups** (local or S3-compatible storage), or [volume backups](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes). The production docs suggest `sqlite3 .backup` plus `rsync` for very large `pb_data` folders. For continuous replication, the FAQ recommends Litestream.

### How do I upgrade PocketBase?
Back up first, change `PB_VERSION` in the Dockerfile, and push. The service rebuilds and restarts on the new binary.

## Conclusion

You now have PocketBase on a public HTTPS URL, built from the documented Dockerfile, with SQLite and uploads on a persistent `pb_data` volume, one instance by design, and a superuser created before anyone else could claim it.

**Next:** [link your own domain](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port), set the Application URL and SMTP settings in the dashboard, and schedule backups. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy Payload CMS](deploy-payload-cms.md)
- [How to deploy Strapi](deploy-strapi.md)
- [How to self-host Supabase](self-host-supabase.md)
