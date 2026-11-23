<!--
agent-guide-meta:
  id: NF-JOB-004
  description: "How to deploy AI agents in production as always-on container services with secrets, Postgres or volume state, health checks and a restart test for memory."
  sources:
    - https://northflank.com/docs/v1/api/project/services/create-combined-service
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/project/volumes/get-volume
    - https://northflank.com/docs/v1/api/project/secrets/create-project-secret
    - https://northflank.com/docs/v1/api/project/secrets/get-project-secret
    - https://northflank.com/docs/v1/api/project/services/update-service-health-checks
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/services/get-service-logs
    - https://northflank.com/docs/v1/api/project/services/scale-service
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/application/run/run-an-image-continuously
    - https://northflank.com/docs/v1/application/ai-agents/agents-on-northflank
  related: [NF-AI-003, NF-JOB-002, NF-DB-001, NF-DEP-001, NF-DEP-002]
-->

# How to deploy AI agents in production

To deploy AI agents in production, run the agent's Docker image as an always-on container service rather than a time-limited function, keep model API keys in runtime secrets, store conversations and checkpoints in managed Postgres or a persistent volume, add startup, liveness and readiness checks on a model-free `/healthz`, and stream or go async for long runs. State written anywhere else is lost on the first restart.

This guide does it on Northflank with the CLI for any framework (LangGraph, Claude Agent SDK, OpenAI Agents SDK or your own loop): a secret group for keys, Postgres or a `/data` volume for state, and health checks. You end with a public HTTPS URL whose `/healthz` returns 200 and whose state survives a restart, verified by reading it back. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy AI agents in production

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the agent is an HTTP server with a `Dockerfile` and a model-free `/healthz`, that its model keys are set, and confirm the team, repository, branch and project. Give Northflank access to the repository (once).
3. Pick state storage, managed Postgres (`DATABASE_URL`) or a volume at `/data`, and put model keys (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, …) in a secret group restricted to the service.
4. Create the service with `northflank put service combined`, one public HTTP port and startup, liveness and readiness checks.
5. Wait for the deploy, check that the live build is your pushed commit and `/healthz` returns 200, then write state, run `northflank restart service`, and read it back.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy your AI agent as a container service?

Agent runs are long and stateful. A single task can take minutes of model calls and tool use, and the next request needs the memory of the last one. Request-scoped serverless platforms fight both: they cap execution time and throw away the filesystem between invocations. A container service gives you:

- **A process that stays up**: background loops, streaming responses and in-memory caches keep working.
- **State you control**: a database or disk that outlives every restart and deploy.
- **Secrets outside the image**: keys injected at runtime, rotated without a rebuild.
- **Health checks and logs**: the platform restarts a stuck agent, and you can read what it did.

So where should an agent run? It needs a process that can work for minutes per task, a database or disk for memory and checkpoints, secrets injected at runtime, and often extra memory for tools such as browsers or code execution. Serverless functions suit short, stateless agent calls but not long multi-step runs. A raw VM or your own Kubernetes cluster gives full control, but you handle TLS, restarts, secrets and scaling yourself. A container platform runs the agent's Docker image as an always-on service with managed Postgres, secrets and health checks built in.

## Prerequisites

- **An agent that runs locally as an HTTP server** on `0.0.0.0` and a fixed port (this guide uses `8080`), with a cheap `GET /healthz` that returns 200 without calling a model. With it running, this prints `200`:
  ```bash
  curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/healthz
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The model API keys in your shell environment** (for example `export ANTHROPIC_API_KEY=…`), never in the repo or chat.

Names used below: project `ai-agent`, service `agent-api`, addon `agent-db`, secret group `agent-secrets`, volume `agent-state`.

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

Before changing anything, confirm that this guide fits the app and print the deployment target. Run this from the repository root:

```bash
ls Dockerfile package.json requirements.txt pyproject.toml 2>/dev/null
grep -oiE 'langgraph|claude-agent-sdk|claude_agent_sdk|openai-agents|anthropic|openai' package.json requirements.txt pyproject.toml 2>/dev/null | sort -u
for K in ANTHROPIC_API_KEY OPENAI_API_KEY; do [ -n "${!K}" ] && echo "$K: set" || echo "$K: not set"; done

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId ai-agent -o json >/dev/null 2>&1 \
  && echo "project: ai-agent exists and will be reused" \
  || echo "project: ai-agent will be created"
```

- If there's no `Dockerfile`, add one that starts the agent's HTTP server on `0.0.0.0:8080` before you continue; this guide builds that image as-is. The `grep` shows which agent framework and model SDK the app uses; any works.
- The key you need must say `set`. The check never prints the values. Add any other provider's key to the loop.
- State must be written to one place: a database via `DATABASE_URL`, or files under one directory (this guide uses `/data`). Anything written elsewhere in the container is lost on restart.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "ai-agent", "region": "europe-west" }'
export PROJECT_ID=ai-agent

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Choose where state lives

| | Managed Postgres | Persistent volume |
|---|---|---|
| Good for | Conversations, checkpoints (LangGraph's Postgres checkpointer), task queues, anything queryable | SQLite files, vector index files, a workspace directory the agent edits |
| Instances | Any number | **One** with the default Single Read/Write access mode |
| Restarts | Zero-downtime rolling restarts | The docs say the old container is always stopped before the new one starts, so expect a short gap |
| Backups | Addon backups | Volume backups |

If you're unsure, use Postgres. Create the addon and wait until its `status` is `running`:

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "agent-db",
  "type": "postgresql",
  "version": "latest",
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'
ADDON_OK=0
for i in $(seq 1 60); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId agent-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"; [ "$A" = "running" ] && { ADDON_OK=1; break; }
  case "$A" in failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$ADDON_OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon before linking it"; false; }
```

Storage can only be increased later, so start small. For versions, TLS and backups, see [how to deploy a PostgreSQL database](deploy-postgresql-database.md).

## Step 6: Store model keys and connection strings in a secret group

Build the group from your shell environment so values never appear in a file you keep or in command output:

```bash
umask 077
jq -n --arg ak "$ANTHROPIC_API_KEY" '{
  name: "agent-secrets",
  type: "secret",
  secretType: "environment",
  priority: 10,
  restrictions: { restricted: true, nfObjects: [ {id: "agent-api", type: "service"} ] },
  secrets: { variables: { ANTHROPIC_API_KEY: $ak } },
  addonDependencies: [ { addonId: "agent-db", keys: [ { keyName: "POSTGRES_URI", aliases: ["DATABASE_URL"] } ] } ]
}' > /tmp/agent-secrets.json
northflank put secret --projectId "$PROJECT_ID" --file /tmp/agent-secrets.json -o json | jq '(.data // .).id'
rm -f /tmp/agent-secrets.json
```

Drop `addonDependencies` if you're using a volume. Add other providers' keys the same way (`--arg ok "$OPENAI_API_KEY"`). `secretType: environment` keeps the values out of builds, and the restriction means only `agent-api` inherits them. Confirm the database link with `northflank get secret-link --projectId "$PROJECT_ID" --secretId agent-secrets --addonId agent-db -o json` (see [connect database secrets to workloads](https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads)).

## Step 7: Create the service with health checks

Northflank builds what's on the remote branch, not your local working tree, so commit and push your changes first (for example the `Dockerfile` or `/healthz` route; skip `git commit` if there's nothing new):

```bash
git add Dockerfile
git commit -m "Add Northflank deployment files"
git push
```

Save as `agent-api.json` (combined service, built from your repo):

```json
{
  "name": "agent-api",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": {
    "instances": 1,
    "docker": { "configType": "default" },
    "gracePeriodSeconds": 300
  },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "8080", "STATE_DIR": "/data" },
  "healthChecks": [
    { "type": "startupProbe",   "protocol": "HTTP", "path": "/healthz", "port": 8080,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 30, "successThreshold": 1 },
    { "type": "livenessProbe",  "protocol": "HTTP", "path": "/healthz", "port": 8080,
      "initialDelaySeconds": 1, "periodSeconds": 30, "timeoutSeconds": 5, "failureThreshold": 3, "successThreshold": 1 },
    { "type": "readinessProbe", "protocol": "HTTP", "path": "/healthz", "port": 8080,
      "initialDelaySeconds": 1, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file agent-api.json | jq '(.data // .).id'
```

Use the same repository URL, provider and branch as in step 4. The first build starts automatically. `nf-compute-100-2` suits an agent that only calls model APIs; agents that run tools (browsers, code execution) need more memory. List plans with `curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | "\(.id) cpu=\(.cpuResource) ram=\(.ramResource)MB"'`.

For a prebuilt image, drop `vcsData` and `buildSettings`, add `"external": {"imagePath": "<IMAGE>:<TAG>"}` to `deployment`, and use `northflank put service deployment`. REST: `PUT https://api.northflank.com/v1/projects/"$PROJECT_ID"/services/combined` (or `/deployment`).

Why these settings:

- The **startup probe** gives slow boots (loading models, warming caches) up to 5 minutes before liveness can kill the container.
- **Liveness** restarts a hung process; **readiness** only takes it out of traffic. HTTP passes on 200–399. Keep `/healthz` free of model calls so a provider outage doesn't restart your agent.
- `gracePeriodSeconds: 300` gives in-flight agent runs up to 5 minutes to finish after `SIGTERM` during a deploy. Handle `SIGTERM` in your code by refusing new work and finishing the current task.
- Set `PORT` yourself; it isn't documented that the platform injects it.

See [configure health checks](https://northflank.com/docs/v1/application/observe/configure-health-checks) for more options.

## Step 8 (volume only): Create and attach the volume

Skip this if state is in Postgres. Otherwise, create a volume attached to the service and mounted at `/data`:

```bash
northflank create volume --projectId "$PROJECT_ID" -o json --input '{
  "name": "agent-state",
  "mounts": [ { "containerMountPath": "/data", "volumeMountPath": "" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageClassName": "ssd", "storageSize": 6144 },
  "attachedObjects": [ { "id": "agent-api", "type": "service" } ]
}' | jq '(.data // .) | {id, spec, attachedObjects}'
```

REST: `POST https://api.northflank.com/v1/projects/"$PROJECT_ID"/volumes`. Things to know, from the volume docs and schema:

- `storageSize` is in MB and **can't be reduced** later. The access mode can't be changed after creation.
- `ReadWriteOnce` limits the service to **one instance**. `ReadWriteMany` (Multi Read/Write in the docs) lets several replicas mount the same volume, but the API schema still says only `ReadWriteOnce` is generally available, and concurrent writers need their own coordination.
- Attaching a volume redeploys the service.
- If the app runs as a non-root user and gets permission errors on `/data`, fix ownership in the Dockerfile (`USER`) or with a `chown` at start-up, as the docs describe.

See [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) and [back up and clone volumes](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes) for backups and resizing.

## Step 9: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds and prints your URL when the agent is live, then checks its health route:

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId agent-api --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId agent-api --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId agent-api --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

```bash
URL="https://$HOST"
curl -sS -o /dev/null -w '%{http_code}\n' "$URL/healthz"
```

If it prints `ROLLOUT_NOT_COMPLETE`, don't continue: read `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId agent-api` and the runtime logs (step 11). `COMPLETED` means rolled out and serving, which isn't the same as healthy forever; step 11 checks restarts.

## Step 10: Keep long agent runs off a single blocking request

The process can run as long as it likes, but a single HTTP request still passes through the public load balancer, and its idle and request timeouts aren't documented in the pages this guide is based on. Don't rely on one synchronous request staying open for minutes. Pick one:

- **Stream** progress (Server-Sent Events or chunked responses) so bytes flow regularly.
- **Go async**: `POST /runs` returns a run ID immediately, the agent works in the background, and the client polls `GET /runs/<id>` or receives a webhook. Store run state in Postgres or `/data` so it survives restarts.

For heavy background work, run a separate [worker service](deploy-celery-in-production.md) that reads from a queue, with the API in front.

## Step 11: Prove state survives a restart

Write something through your own API, restart, and read it back. For example, with a sessions endpoint:

```bash
SID=$(curl -sS -X POST "$URL/sessions" -H 'Content-Type: application/json' -d '{"note":"persistence-check"}' | jq -r .id)
northflank restart service --projectId "$PROJECT_ID" --serviceId agent-api
sleep 20
OK=0
for i in $(seq 1 60); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId agent-api -o json | jq -r '(.data // .).status.deployment.status')
  [ "$S" = "COMPLETED" ] && { OK=1; break; }; [ "$S" = "FAILED" ] && { echo DEPLOY_FAILED; break; }; sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
[ "$OK" = 1 ] && curl -sS "$URL/sessions/$SID" | jq .
```

If your API has no such endpoint, check the volume directly (`script` is needed because `exec` drops output without a TTY):

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId agent-api \
  --shell-cmd 'sh -c' --cmd 'date > /data/.persist-check && cat /data/.persist-check'
# restart and wait as above, then:
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId agent-api \
  --shell-cmd 'sh -c' --cmd 'cat /data/.persist-check'
```

Then read recent logs:

```bash
northflank get service logs --projectId "$PROJECT_ID" --serviceId agent-api --types runtime \
  --duration 900 --lineLimit 200 --direction backward -o json | jq -r '(.data? // .)[] | "\(.ts) \(.log)"'
```

Add `--tail` to stream logs live while you test.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId agent-api --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The step 9 loop ended with `build=SUCCESS deployment=COMPLETED`, and `curl -s -o /dev/null -w '%{http_code}' "$URL/healthz"` prints `200`.
3. The state written before `restart service` in step 11 reads back identically afterwards.
4. `northflank get service health --projectId "$PROJECT_ID" --serviceId agent-api --duration 900 -o json` shows `restartCount` unchanged across two reads 5 minutes apart.
5. `northflank get secret --projectId "$PROJECT_ID" --secretId agent-secrets --show all -o json | jq '(.data // .) | {keys: (.secrets.variables | keys), restrictions}'` lists `ANTHROPIC_API_KEY` (and `DATABASE_URL` if used) and shows the group restricted to `agent-api`. Print keys, never values. (The service-level `get service runtime-environment` endpoints are deprecated in the API.)

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Container restarts during boot | Liveness fires before the app is ready | Keep the startup probe; raise its `failureThreshold` |
| Restarts when the model provider is slow | `/healthz` calls the model | Make `/healthz` local-only |
| State gone after restart | Written outside `/data`, or volume not attached | Check `northflank get volume --volumeId agent-state` → `attachedObjects`; fix the app's path |
| Can't scale above 1 | `ReadWriteOnce` volume | Move state to Postgres, then scale |
| `401` from the model API | Key missing or not inherited | Check the group's `restrictions` include `agent-api`; restart the service |
| Long requests cut off | Load balancer timeout | Stream or go async (step 10) |
| Permission denied on `/data` | Volume ownership vs container user | Set `USER` or `chown` at start-up |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print, log or commit model API keys or connection strings. Build secret files from environment variables and delete them.
- Confirm plan, instance count and volume size with the user. Volume size and access mode can't be reduced or changed later.
- Don't detach or delete a volume or database without explicit confirmation and a backup; that's the agent's memory.
- If the agent you're deploying can run tools or code, ask the user how much network and filesystem access it should have, and consider [sandboxes for untrusted code](https://northflank.com/docs/v1/application/use-cases/run-code-in-sandboxes).
- Restarting is part of the persistence test here; don't use it as a fix for crashes (read the logs instead; see [view logs](https://northflank.com/docs/v1/application/observe/view-logs)).

## FAQ

### Where should I deploy my AI agent?
On a host that runs it as a long-lived container rather than a time-limited function, with a managed database (or a volume) for its state and secrets kept outside the image. That applies whether it's built with LangGraph, an agent SDK or your own loop. A VM or Kubernetes cluster works if you'll operate it yourself; a container platform with managed Postgres, as in this guide, handles HTTPS, restarts and secrets.

### Do I need a volume if I use Postgres?
No. Keep the container stateless and put conversations, checkpoints and run records in Postgres. Use a volume only for file-shaped state.

### Can I run more than one instance?
Yes with Postgres. With a `ReadWriteOnce` volume, the service is limited to one instance.

### How do I rotate a model API key?
Update the secret group's `secrets.variables` with the full map (`northflank patch secret`), then `northflank restart service`. See [manage secret groups](https://northflank.com/docs/v1/application/secure/manage-secret-groups).

### Is there a request timeout?
The process itself has no time limit. Load balancer timeouts for a single HTTP request aren't documented in the sources used here, so stream or use an async pattern for long runs.

## Conclusion

Your agent now runs as an always-on service with keys in a restricted secret group, state in Postgres or on a volume, health checks that don't depend on the model provider, and a restart test that proves it remembers. The same definition redeploys identically from a laptop, CI or a coding agent.

**Next:** expose tools to other agents with [a remote MCP server](deploy-mcp-server.md), and keep [the logs docs](https://northflank.com/docs/v1/application/observe/view-logs) handy. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy an MCP server](deploy-mcp-server.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy a Docker image](deploy-docker-image.md)
