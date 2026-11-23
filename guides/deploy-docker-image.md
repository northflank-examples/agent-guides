<!--
agent-guide-meta:
  id: NF-DEP-002
  description: "How to deploy a Docker image from Docker Hub, GHCR or a private registry, run it behind a public HTTPS URL and roll to new tags, using the CLI or REST API."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/create-deployment-service
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/team/integrations/add-registry
    - https://northflank.com/docs/v1/application/run/run-an-image-from-a-container-registry
    - https://northflank.com/docs/v1/application/run/save-registry-credentials
  related: [NF-DEP-001, NF-DEP-003, NF-DB-001, NF-JOB-002, NF-AI-003]
-->

# How to deploy a Docker image

To deploy a Docker image, point a container host at the full image path with an immutable tag, give it read-only registry credentials if the image is private, expose the port the container listens on over HTTPS, and roll out new versions by changing only the tag. No source code or build step is involved. Deploying `latest` instead of a fixed tag or digest is what makes rollbacks ambiguous later.

This guide does it on Northflank with the CLI, with the matching REST calls alongside: saved registry credentials, a deployment service that pulls and runs the image, and a public HTTPS URL verified with `curl` and the running image tag. Every step is a copy-pasteable command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end. For a private image, a human creates the registry token once.

## TL;DR: how to deploy a Docker image

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team, the image and tag, its port and the project before changing anything.
3. Create a project, and save registry credentials if the image is private.
4. Create a deployment service from the image with `northflank put service deployment --file service.json`.
5. Wait for the rollout, then check that the public URL answers and that the running `imagePath` has the tag you deployed.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Where to deploy a Docker image

To run a Docker image you need a host that can pull it (with credentials if the registry is private), keep the container running and restart it if it crashes, inject environment variables, and route HTTPS traffic to its port. Your options are roughly: a **container platform**, which does all of that from an image reference; **Kubernetes** (managed or your own), which does too but leaves you writing and operating the manifests; or a **VM** running Docker, where TLS, restarts and updates are yours to script. Static and most serverless function hosts don't run arbitrary long-running containers.

If CI already builds and pushes your image, building it again on the platform is wasted work. On Northflank, a **deployment service** pulls the external image, and saved **registry credentials** authorise private pulls. Changing the tag later updates only the image; ports, env and plan stay as they are.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The full image path with an immutable tag**, and the port the container listens on (step 2 shows how to check both).

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Check the image and confirm where it will deploy

Before changing anything, print the deployment target. Replace `<IMAGE>` with your image path and tag:

```bash
IMAGE=<IMAGE>
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "image:   $IMAGE"
northflank get project --projectId docker-image -o json >/dev/null 2>&1 \
  && echo "project: docker-image exists and will be reused" \
  || echo "project: docker-image will be created"
```

- The image path has the form `[registry-url]/[account]/[image]:[tag]`. Without a registry host the platform assumes Docker Hub; without an account it assumes `library` (`nginx:latest` → `docker.io/library/nginx:latest`). Prefer an immutable tag or digest (`@sha256:…`) over `latest`.
- Find the port the image listens on with `docker inspect --format '{{json .Config.ExposedPorts}}' "$IMAGE"` if Docker is available locally, or in the image docs. The steps below call it `<PORT>`.
- If `docker pull "$IMAGE"` works with no registry login, the image is public: skip step 4.

**If you're an agent:** show the user the team, the image and tag, the port, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Create a project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "docker-image", "region": "europe-west" }'
export PROJECT_ID=docker-image
```

## Step 4: Save registry credentials (private images only)

First look for an existing credential:

```bash
northflank list registry-credentials -o json
```

If none fits, you need a read-only token for the registry (GHCR: personal access token (classic) with `read:packages`; Docker Hub: access token with read permission; GitLab: token with `read_registry`).

**This step needs a human:** the user has to create that token in the registry's UI and give it to you through a secure channel (environment variable, secret manager). Don't ask them to paste it into a file that gets committed. Continue when `$REGISTRY_USER` and `$REGISTRY_TOKEN` are set.

```bash
northflank add registry-credentials --noDefaults -o json --input "$(jq -n \
  --arg u "$REGISTRY_USER" --arg p "$REGISTRY_TOKEN" --arg proj "$PROJECT_ID" '{
    name: "ghcr-acme",
    provider: "github",
    credentials: { username: $u, password: $p, scope: { pull: true, push: false } },
    restrictions: { restricted: true, projects: [$proj] }
  }')" | jq '(.data // .) | {id, name}'
```

The credential's ID is the slugified name, `ghcr-acme` here; `service.json` uses it as `<CREDENTIAL_ID>`. `provider` is one of `acr, ecr, gar, dockerhub, dhi, github, gitlab, custom, legacy`. `credentials.scope` is required by the schema.

It hasn't been confirmed whether `registryUrl` is needed for the `github` and `dockerhub` providers. If the call is rejected for a missing registry URL, add `"registryUrl": "ghcr.io"` (or the registry host) and retry. For `custom`, always set `registryUrl`.

REST: `POST https://api.northflank.com/v1/integrations/registries` with the same body; the new ID is at `.data.id`.

## Step 5: Create the deployment service

Write `service.json`. Port names are max 8 characters and start with a letter; `protocol` is required.

```json
{
  "name": "app",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": {
    "instances": 1,
    "external": {
      "imagePath": "<IMAGE>",
      "credentials": "<CREDENTIAL_ID>"
    },
    "docker": { "configType": "default" }
  },
  "ports": [
    { "name": "http", "internalPort": <PORT>, "public": true, "protocol": "HTTP" }
  ],
  "runtimeEnvironment": { "PORT": "<PORT>" }
}
```

Remove the `credentials` line for public images. List plan IDs with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[].id'`.

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file service.json
```

The service ID is the slugified name, `app`. REST (create-or-update):

```bash
curl -s -X PUT "https://api.northflank.com/v1/projects/$PROJECT_ID/services/deployment" \
  -H "Authorization: Bearer $NF_API_TOKEN" -H "Content-Type: application/json" \
  --data @service.json | jq '.data.id'
```

Use `northflank create service deployment` / `POST` on the same path if you want a `409` when the service already exists.

## Step 6: Wait for the rollout and get your URL

`status.deployment.status == "COMPLETED"` means rolled out and serving. This loop checks every 10 seconds and prints your URL when the app is live.

```bash
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
      | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
if [ "$OK" = 1 ]; then
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId app`. The troubleshooting table covers the common causes.

## Step 7: Roll to a new image tag later

Only the image changes; ports, env and plan stay as they are. The PATCH schema requires `deployment.instances` whenever you send `deployment`, so read the current count and send it back unchanged:

```bash
INSTANCES=$(northflank get service --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
  | jq -r '(.data // .).deployment.instances')
northflank patch service deployment --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
  --input "$(jq -n --argjson n "$INSTANCES" '{deployment: {instances: $n, external: {imagePath: "<IMAGE_NEW_TAG>", credentials: "<CREDENTIAL_ID>"}}}')"
```

REST: `PATCH https://api.northflank.com/v1/projects/{projectId}/services/deployment/{serviceId}` with the same body. Then run the step 6 loop again. (`update service deployment` and its `POST …/services/{serviceId}/deployment` endpoint are deprecated in the API.)

## How to know it worked

You're done when all three of these hold:

1. The step 6 loop ended with `deployment=COMPLETED` and printed a URL.
2. The running image is the one requested:
   ```bash
   northflank get service --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
     | jq -r '(.data // .).deployment.external.imagePath'
   ```
   The tag in `imagePath` must equal the tag you deployed.
3. The app answers on its public URL. This prints the status code; expect 200, or the status the app returns on `/` (some APIs return 404 there, so use a route you know exists):
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"
   ```
   Retry for about 30 seconds after the rollout if it returns 502 or 503.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Image cannot be verified / pull errors | Private image without credentials, wrong path or tag | Check `<IMAGE>` exists; add `credentials` (step 4) |
| Pull denied with credentials set | Token lacks read scope, or credential restricted to another project | Recreate token with read scope; include `$PROJECT_ID` in `restrictions.projects` |
| `imagePath` validation error | Missing tag/digest or invalid characters | Use `host/account/image:tag` or `@sha256:…` |
| Deployment `FAILED` or restarting | Container exits, wrong command, wrong architecture | `northflank get service logs --projectId … --serviceId …`; see [view logs](https://northflank.com/docs/v1/application/observe/view-logs) |
| 502/503 while `COMPLETED` | `internalPort` does not match the process, or it binds to `127.0.0.1` | Fix the port / bind `0.0.0.0`, re-run step 5 |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

For more on logs and status, see [view logs](https://northflank.com/docs/v1/application/observe/view-logs).

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write: team, image and tag, port, region and whether the project is new or reused.
- Don't report success until check 2 shows the tag you deployed and check 3 gets the expected status.
- If the step 6 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Registry tokens are secrets: never echo them, write them to the repo, or include them in `-o json` output you paste back to the user.
- `put service deployment` replaces the spec with your body; fetch the current spec first if the service already exists. For a tag change, use step 7 instead.
- Prefer immutable tags: redeploying `latest` makes rollbacks ambiguous.

## FAQ

### Where should I deploy my Docker image?
If you want an HTTPS URL without managing servers, use a container platform that deploys straight from an image reference and handles TLS, restarts and rollouts. Pick Kubernetes when you already operate a cluster or need its controls, and a single VM with Docker when you're fine scripting TLS, restarts and updates yourself. Whichever you choose, deploy an immutable tag so you can roll back.

### Do I need registry credentials for a public image?
No. If you can pull the image anonymously, skip step 4 and remove the `credentials` line from `service.json`.

### Which registries are supported?
The `provider` field accepts `acr, ecr, gar, dockerhub, dhi, github, gitlab, custom, legacy`. For `custom`, always set `registryUrl`.

### How do I deploy a new version?
Run `northflank patch service deployment` with the new `imagePath` (step 7), then wait for the rollout again. Only the image changes.

### Why should I avoid the `latest` tag?
Redeploying `latest` makes rollbacks ambiguous. Use an immutable tag or a digest (`@sha256:…`) so you always know what's running.

## Conclusion

You now have an existing container image running behind a public HTTPS URL, with private pulls authorised by saved credentials and a one-command path to roll to a new tag.

**Next:** to build from source instead, see [how to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md). Add env vars and secrets with [secret groups](https://northflank.com/docs/v1/application/secure/manage-secret-groups), and add a database with [how to deploy a PostgreSQL database](deploy-postgresql-database.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy an app without a Dockerfile](deploy-app-without-dockerfile.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy an MCP server](deploy-mcp-server.md)
