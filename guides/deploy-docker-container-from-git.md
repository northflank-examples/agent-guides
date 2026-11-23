<!--
agent-guide-meta:
  id: NF-DEP-001
  description: "How to deploy a Docker container from a Git repository: build from the Dockerfile on every push, run it as a service and get a public HTTPS URL via CLI or API."
  sources:
    - https://northflank.com/docs/v1/api/project/services/create-combined-service
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/getting-started/build-and-deploy-your-code
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/network/configure-ports
  related: [NF-DEP-002, NF-DEP-003, NF-DEP-004, NF-DB-001, NF-DEP-012]
-->

# How to deploy a Docker container from a Git repository

To deploy a Docker container from a Git repository, give the platform read access to the repo, build the image from its `Dockerfile` on every push to one branch, run it as a long-lived container listening on `0.0.0.0` at a known port, and route a public HTTPS hostname to that port. A port mismatch is the most common reason a healthy-looking deploy returns 502s.

This guide does it on Northflank with the CLI, with the matching REST calls alongside: one combined service that builds from Git and runs the container, a status poll that tells you when it's actually serving, and a public HTTPS URL checked with `curl`. Every step is a copy-pasteable command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy a Docker container from a Git repository

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repository has a `Dockerfile`, note the port it listens on, and confirm the team, repository, branch and project before changing anything.
3. Give Northflank access to your repository on the Git integrations page (once), then create a project and check access with `northflank get service repository-access`.
4. Create a combined service with `northflank put service combined --file service.json`, with `internalPort` set to the port the container listens on.
5. Wait until the build is `SUCCESS` and the deployment is `COMPLETED`, then check that the live build is your pushed commit and that the public URL answers.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Where to deploy a Docker container from Git

Deploying from a repo needs two things: somewhere to build the image from your `Dockerfile` on every push, and somewhere to run the container as a long-running process with restarts, environment variables and HTTPS routing to its port. A **container platform** can do both from a Git connection; **Kubernetes** can run the container but needs a separate CI pipeline, registry and manifests; a **VM** with Docker leaves the build, TLS, restarts and updates to your own scripts. Static and serverless function hosts generally don't run your `Dockerfile` as a long-running container.

When the repository already has a `Dockerfile`, you don't need a separate build pipeline and registry. On Northflank, one **combined service** does both the build (from Git) and the deployment. The result is a service that builds every push to one branch, runs the container, and is reachable on an HTTPS URL.

## Prerequisites

- **A `Dockerfile` that builds and runs locally:**
  ```bash
  docker build -t app-check . && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Check the app and confirm where it will deploy

Before changing anything, confirm that this guide fits the repository and print the deployment target. Run this from the repository root:

```bash
[ -f Dockerfile ] && grep -inE '^(FROM|EXPOSE|ENTRYPOINT|CMD)' Dockerfile || echo "Dockerfile: missing"
ls package-lock.json pnpm-lock.yaml yarn.lock requirements.txt poetry.lock uv.lock go.sum Gemfile.lock 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId docker-app -o json >/dev/null 2>&1 \
  && echo "project: docker-app exists and will be reused" \
  || echo "project: docker-app will be created"
```

- If it prints `Dockerfile: missing`, this guide doesn't fit. Stop and use a guide for your framework, or [deploy without a Dockerfile](deploy-app-without-dockerfile.md) using buildpacks.
- Note the port the container listens on: the `EXPOSE` line, or the port the `CMD` passes to the server (`--port`, `app.listen(...)`, a `PORT` default). This guide uses `8080`; replace it in step 5 if yours differs. The server must bind `0.0.0.0`, not `127.0.0.1`.
- If the `Dockerfile` lives in a subfolder (a monorepo), note its path and build context from the repository root, for example `/apps/web/Dockerfile` and `/apps/web`.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "docker-app", "region": "europe-west" }'
export PROJECT_ID=docker-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3. REST: `POST https://api.northflank.com/v1/projects/{projectId}/services/repo-access` with the same body.

## Step 5: Create the combined service

Northflank builds what's on the remote branch, not your local working tree, so commit and push the `Dockerfile` and any changes first:

```bash
git add Dockerfile
git commit -m "Add Northflank deployment files"
git push
```

Write `service.json`. Port names are max 8 characters and start with a letter. `protocol` is required on each port. Set `internalPort` and `PORT` to the port from step 2.

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": {
    "projectUrl": "https://github.com/<OWNER>/<REPO>",
    "projectType": "github",
    "projectBranch": "main"
  },
  "buildSettings": {
    "dockerfile": {
      "buildEngine": "buildkit",
      "dockerFilePath": "/Dockerfile",
      "dockerWorkDir": "/"
    }
  },
  "ports": [
    { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" }
  ],
  "runtimeEnvironment": { "PORT": "8080" }
}
```

Use the same repository URL, provider and branch as in step 4. `billing.buildPlan` is optional; leave it out to use the default build plan, and only set it if builds run out of memory, using a build plan your team has access to (an unavailable one fails with `Build plan not found`).

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file service.json
```

REST (upsert): `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined`. Use `POST` on the same path (`northflank create service combined`) if you want a `409 Conflict` instead of an update when the service already exists.

```bash
curl -s -X PUT "https://api.northflank.com/v1/projects/$PROJECT_ID/services/combined" \
  -H "Authorization: Bearer $NF_API_TOKEN" -H "Content-Type: application/json" \
  --data @service.json | jq '.data.id'
```

Creating a combined service with CI enabled starts a build of the latest commit on the branch. If no build appears within a minute, start one:

```bash
northflank start service build --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json --input '{}'
```

REST: `POST /v1/projects/{projectId}/services/{serviceId}/build`.

## Step 6: Wait for the deploy and get your URL

`status.deployment.status == "COMPLETED"` means rolled out and serving, not exited. This loop checks every 10 seconds (the API limit is 1,000 requests per hour) and prints your URL when the app is live.

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web`. The troubleshooting table covers the common causes. Report the URL to the user.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The service is still serving after the rollout (not restarting):
   ```bash
   northflank get service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
     | jq -r '(.data // .).status.deployment.status'
   ```
   It should print `COMPLETED`.
3. The public URL is answered by your app. Use a route your app serves (`/` here, or a health route such as `/healthz`); it should print `200` (or the status that route normally returns):
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"
   ```
   Retry a few times over about 30 seconds; TLS and routing can lag the rollout briefly. A `502` or `503` means nothing is answering on `internalPort`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `put service combined` rejects `vcsData` | Wrong `projectType`, or branch does not exist | Re-check the provider; `northflank list branches --help` to list the repo's branches |
| Validation error on `ports` | Name > 8 chars, starts with digit, or `protocol` missing | Use e.g. `http`, add `"protocol": "HTTP"` |
| Build status `FAILURE` | Dockerfile error, wrong path or context | `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web`; see [view builds](https://northflank.com/docs/v1/application/observe/see-builds) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| Deployment `FAILED` or restarting | App crashes or listens on another port / `127.0.0.1` | Bind `0.0.0.0:<PORT>`; see [view logs](https://northflank.com/docs/v1/application/observe/view-logs) |
| 502/503 from the URL while `COMPLETED` | `internalPort` does not match the listening port | Fix `internalPort` and re-run step 5 |
| `409 Conflict` on create | Service with same slug exists | Use `put` (upsert) or a new name |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

For logs and status details, see [view logs](https://northflank.com/docs/v1/application/observe/view-logs).

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 6 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- `put` overwrites the service spec with the body you send. Before re-running on an existing service, fetch it (`northflank get service … -o json`) and keep fields you didn't intend to change.
- Don't put secrets in `runtimeEnvironment` of a committed file; use a secret group (see [manage secret groups](https://northflank.com/docs/v1/application/secure/manage-secret-groups)).
- Don't delete an existing service to "start fresh" without explicit user confirmation for that service.

## FAQ

### Where should I deploy my Docker container?
If the code is in Git and you want push-to-deploy with an HTTPS URL, use a container platform that builds from the repository and runs the container in one place. Kubernetes fits when you already run a cluster and a CI pipeline that pushes images, and a VM with Docker fits when you want full control and are fine scripting builds, TLS and restarts yourself.

### What if my repository is private?
Run the repository-access check in step 4. If it returns `no-linked-access` or `not-found`, the user has to link their Git account or grant the Northflank app access in the UI. That can't be done through the CLI or API.

### Does every push redeploy the app?
Yes. The combined service builds every push to the branch in `vcsData.projectBranch`, then runs the new container. Creating the service with CI enabled also starts a build of the latest commit on that branch.

### Why does the URL return 502 or 503 when the deployment is `COMPLETED`?
Usually `internalPort` doesn't match the port the app listens on, or the app binds to `127.0.0.1` instead of `0.0.0.0`. Fix the port and re-run step 5.

### Should I use `put` or `create`?
`put` creates or updates, so it's safe to re-run. `create` (`POST`) returns `409 Conflict` if a service with the same slug already exists, which is useful when you don't want to overwrite anything.

## Conclusion

You now have a service that builds your repository from its `Dockerfile` on every push, runs it, and serves it on a public HTTPS URL, with a status check that tells you when it's actually serving.

**Next:** add a database and `DATABASE_URL` with [how to deploy a Node.js application](deploy-nodejs-application.md) or [how to deploy a PostgreSQL database](deploy-postgresql-database.md). Already have an image from CI? See [how to deploy a Docker image](deploy-docker-image.md). For the whole stack in one run, see [how to deploy a full stack app](deploy-full-stack-app.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy an app without a Dockerfile](deploy-app-without-dockerfile.md)
- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a full stack app](deploy-full-stack-app.md)
