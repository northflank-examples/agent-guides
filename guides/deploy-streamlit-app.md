<!--
agent-guide-meta:
  id: NF-STK-STREAMLIT
  description: "How to deploy a Streamlit app with Docker: headless on 0.0.0.0:8501, WebSockets over HTTPS, secrets from the environment and built-in health checks."
  sources:
    - https://docs.streamlit.io/develop/api-reference/configuration/config.toml
    - https://docs.streamlit.io/develop/concepts/configuration/options
    - https://docs.streamlit.io/develop/concepts/connections/secrets-management
    - https://docs.streamlit.io/deploy/tutorials/docker
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/load-balancing
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/secure/upload-secret-files
  related: [NF-DEP-001, NF-DB-001, NF-DEP-006, NF-STK-SVELTEKIT]
-->

# How to deploy a Streamlit app

To deploy a Streamlit app, run it in a long-lived Docker container with `streamlit run app.py --server.port=8501 --server.address=0.0.0.0 --server.headless=true`, serve it behind a proxy that passes WebSockets through, read API keys from environment variables instead of a committed `secrets.toml`, and health-check `/_stcore/health`. Each session's state lives in one server process, so serverless hosts don't fit and extra instances need sticky routing.

This guide does it on Northflank with the CLI: a small `Dockerfile` built from Git and served on port 8501, secrets in a secret group, and readiness and liveness probes on Streamlit's health endpoint. You end with a public HTTPS URL where `/_stcore/health` returns 200 and the page loads; a person or browser tool confirms the WebSocket connects. A human only has to log in once and share the repository with Northflank. Every step is a command a developer or a coding agent can run as-is.

## TL;DR: how to deploy a Streamlit app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the app is a Streamlit app and confirm the team, repository, branch and project before changing anything, then give Northflank access to the repository (once).
3. Add a `Dockerfile` that runs `streamlit run app.py --server.port=8501 --server.address=0.0.0.0 --server.headless=true`, read secrets with `os.environ`, and push.
4. Create a project, a secret group for API keys, and a service with `northflank put service combined --file app.json`: HTTP port 8501 and health checks on `/_stcore/health`.
5. Wait for the deploy, then check that the live build is your pushed commit, `/_stcore/health` returns 200 and the WebSocket connects.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why a container for Streamlit?

A Streamlit app is a long-running Python server. The browser loads a page over HTTP, then keeps a WebSocket open for the whole session, and each session's state lives in that server process. That rules out serverless platforms and makes a long-running container the natural fit. On Northflank, one **combined service** builds the image from Git on every push, runs it, and exposes it on an HTTPS URL. Public HTTP ports support WebSockets, so no extra proxy configuration is needed.

So when choosing where to deploy a Streamlit app, look for a host that keeps a process running and passes WebSockets through. A raw VM does this, but you set up the reverse proxy, TLS and restarts yourself. Streamlit-specific hosting is quick for public demos; a general-purpose host gives you more control over the runtime, secrets and private dependencies. A container platform runs your own `Dockerfile` with HTTPS and WebSockets working by default.

## Prerequisites

- **A Streamlit app that runs locally.** This guide uses `app.py` as the entry script and assumes dependencies are pinned in `requirements.txt`, including `streamlit`. Run it, then open `http://localhost:8501`:
  ```bash
  pip install -r requirements.txt && streamlit run app.py --server.headless=true
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

Before changing anything, confirm that this guide fits the app and print the deployment target. Run this from the repository root:

```bash
grep -hiE '^[[:space:]"]*streamlit\b' $(ls requirements.txt pyproject.toml 2>/dev/null) /dev/null || echo "streamlit: missing"
ls app.py requirements.txt pyproject.toml poetry.lock uv.lock Pipfile.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId streamlit-app -o json >/dev/null 2>&1 \
  && echo "project: streamlit-app exists and will be reused" \
  || echo "project: streamlit-app will be created"
```

- If it prints `streamlit: missing`, this isn't a Streamlit app. Stop and use a guide for your framework.
- The lockfile tells you the package manager. Step 5 assumes `requirements.txt`; for Poetry, uv or Pipenv, export a `requirements.txt` or swap the install command. If the entry script isn't `app.py`, use its name in step 5.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "streamlit-app", "region": "europe-west" }'
export PROJECT_ID=streamlit-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Write the `Dockerfile` and read secrets from the environment

This follows Streamlit's own Docker example (port 8501, address `0.0.0.0`), adds headless mode and runs as a non-root user:

```dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 \
    STREAMLIT_BROWSER_GATHER_USAGE_STATS=false
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN useradd --create-home appuser && chown -R appuser /app
USER appuser
EXPOSE 8501
CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0", "--server.headless=true"]
```

What the settings do, from Streamlit's configuration reference:

- `server.port` defaults to 8501. It must match the port in step 7.
- `server.address` binds the server to one address. `0.0.0.0` makes it reachable from outside the container.
- `server.headless=true` stops Streamlit from trying to open a browser on start.
- Every option can also be set as an environment variable: upper snake case, section included, `STREAMLIT_` prefix (`server.port` → `STREAMLIT_SERVER_PORT`). Command-line flags take precedence over environment variables. Exec-form `CMD` doesn't expand `$PORT`, which is why the port is a literal.

Leave `server.enableCORS` and `server.enableXsrfProtection` at their defaults (`true`). The app and its WebSocket are served from the same origin, so you don't need to turn them off. Add a `.dockerignore` with `.streamlit/secrets.toml`, `.venv` and `__pycache__`.

Locally, Streamlit reads secrets from `.streamlit/secrets.toml` through `st.secrets`. In the container, step 6 injects them as environment variables, so read them with `os.environ`:

```python
import os
OPENAI_API_KEY = os.environ["OPENAI_API_KEY"]
```

Make sure `.streamlit/secrets.toml` is not committed: `git ls-files .streamlit` shouldn't list it. Add it to `.gitignore`.

Northflank builds what's in your repository, not the files on your machine, so commit and push:

```bash
git add Dockerfile .dockerignore .gitignore app.py requirements.txt
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Put secrets in a secret group

Skip this step if the app reads no secrets. Create the group without echoing values. `secretType: "environment"` injects them at runtime only, never into the build:

```bash
jq -n --arg k "$OPENAI_API_KEY" '{
  name: "app-secrets", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: true, nfObjects: [ { id: "app", type: "service" } ] },
  secrets: { variables: { OPENAI_API_KEY: $k } }
}' > app-secrets.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file app-secrets.json >/dev/null && rm app-secrets.json
```

Restricting the group to service `app` before it exists hasn't been confirmed to work. If the call is rejected, use `"restricted": false`, or run it after step 7 and restart the service.

If the app depends on `st.secrets` (for example `st.connection` reading a `[connections.*]` section), you can mount a whole `secrets.toml` as a secret file in the group instead. **This step needs a human:** add the file in the secret group's UI with mount path `/app/.streamlit/secrets.toml`. The API's encoding for secret files hasn't been confirmed, so this guide doesn't script it.

## Step 7: Create the service

Write `app.json`. The health checks use Streamlit's built-in `/_stcore/health` endpoint. Data apps often need more memory than a web API, so `nf-compute-50` (1 GB) is a reasonable start (`curl -s https://api.northflank.com/v1/plans` lists the others):

```json
{
  "name": "app",
  "billing": { "deploymentPlan": "nf-compute-50" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8501, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "STREAMLIT_SERVER_MAX_UPLOAD_SIZE": "200" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/_stcore/health", "port": 8501,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/_stcore/health", "port": 8501,
      "initialDelaySeconds": 30, "periodSeconds": 20, "timeoutSeconds": 3, "failureThreshold": 3 }
  ]
}
```

`STREAMLIT_SERVER_MAX_UPLOAD_SIZE` maps to `server.maxUploadSize` (MB, default 200); it shows where non-secret settings go. If the app loads a large model or dataset on start, raise `initialDelaySeconds` so the liveness probe doesn't restart it mid-load.

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file app.json
```

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined` with the same body. The first build starts automatically.

**Instances and WebSockets.** Each browser session holds a WebSocket to one server process, and its `st.session_state` lives there. Keep `instances: 1` unless you need more. To run several, add sticky routing to `app.json` so a client keeps reaching the same instance: `"loadBalancing": { "mode": "consistentHash", "consistentHash": { "mode": "ip" } }`. Whether this keeps sessions intact across reconnects hasn't been confirmed, so test it before relying on it.

## Step 8: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds and prints your URL when the app is live. You can also watch the build in the Northflank dashboard.

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId app`. The troubleshooting table covers the common causes.

`COMPLETED` means rolled out and serving. To change a setting later, patch the **complete** `runtimeEnvironment` map with `northflank patch service combined`, or edit the secret group, then run `northflank restart service --projectId "$PROJECT_ID" --serviceId app`. A restart closes open WebSockets, so users see Streamlit reconnect and lose their session state.

## How to know it worked

You're done when all of these hold. Skip check 5 if the app reads no secrets.

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health endpoint returns 200 (the body is `ok`):
   ```bash
   curl -s -w ' %{http_code}\n' "https://$HOST/_stcore/health"
   ```
3. The ingress upgrades Streamlit's WebSocket endpoint. This should print `101`; a `4xx` or `5xx` means WebSockets aren't reaching the app:
   ```bash
   curl -s --http1.1 --max-time 5 -o /dev/null -w '%{http_code}\n' \
     -H 'Connection: Upgrade' -H 'Upgrade: websocket' -H 'Sec-WebSocket-Version: 13' \
     -H 'Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==' "https://$HOST/_stcore/stream" || true
   ```
4. The app renders: open the URL in a browser. The app shows its widgets and doesn't stay on "Connecting". An HTTP check can't confirm your script runs, so ask the user or use a browser tool.
5. Secrets are injected (names only, no values):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId app \
     --shell-cmd 'sh -c' --cmd '"env | cut -d= -f1 | grep -x OPENAI_API_KEY"'
   ```
   On Linux use `script -qefc '<command>' /dev/null`; without a TTY wrapper, `exec` may lose its output.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |
| Health check fails, logs show `localhost:8501` only | Bound to localhost | Keep `--server.address=0.0.0.0` |
| 502/503 while `COMPLETED` | Port mismatch | Make `--server.port`, `internalPort` and the health check port all 8501 |
| Page loads but stays on "Connecting" | WebSocket blocked or routed to another instance | Use one instance or sticky routing; check browser devtools for the WebSocket request |
| `StreamlitSecretNotFoundError` / `No secrets found` | App calls `st.secrets` and no `secrets.toml` exists | Read `os.environ`, or mount `secrets.toml` as a secret file |
| Container restarts during startup | Liveness probe fires while loading a model or data | Raise `initialDelaySeconds`, or use a bigger plan |
| `OOMKilled` / restarts under load | Data or model too large for the plan | Move to a plan with more memory, cache with `st.cache_resource` |
| Upload fails for large files | Over `server.maxUploadSize` | Raise `STREAMLIT_SERVER_MAX_UPLOAD_SIZE`, restart |

Logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId app` and `northflank get service logs --projectId "$PROJECT_ID" --serviceId app --lineLimit 200`.

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never commit `.streamlit/secrets.toml` or print secret values. Print variable names only.
- Don't disable `enableCORS` or `enableXsrfProtection` to "fix" a connection problem without telling the user; it's rarely the cause here.
- The URL is public. If the app exposes private data or a paid API key, ask the user whether it needs authentication before sharing the link.
- Don't delete the service to "start fresh" without explicit confirmation. Patch and restart instead.

## FAQ

### Where should I deploy my Streamlit app?
On a host that runs a long-lived container and supports WebSockets, which rules out serverless function platforms. A container platform is the simplest fit when you need private secrets, a database or a custom domain. A VM works if you want to manage the proxy and TLS yourself.

### Can I use `st.secrets` instead of `os.environ`?
Yes, if a `secrets.toml` exists in the container. Mount it as a secret file (step 6). Streamlit also exposes root-level `secrets.toml` keys as environment variables, so code that reads `os.environ` works with either approach.

### Can I serve the app under a path such as `/dashboard`?
Set `server.baseUrlPath` (`STREAMLIT_SERVER_BASE_URL_PATH`) to that path. This hasn't been confirmed, but the health endpoint is expected to move under the same prefix, so check `https://$HOST/<path>/_stcore/health` and update the health check paths to match.

### How do I connect it to a database?
Create a managed database in the same project, link its connection string into the secret group as an environment variable, and read it with `os.environ`. See [how to deploy a PostgreSQL database](deploy-postgresql-database.md).

### Does the app need persistent storage?
Not usually. Session state and caches live in memory and reset on restart. Store anything durable in a database or object storage.

## Conclusion

You now have a Streamlit app built from Git on every push, running headless on port 8501 behind HTTPS, with WebSockets working through the platform's ingress, secrets injected from a secret group and health checks on `/_stcore/health`.

**Next:** add a database with [how to deploy a PostgreSQL database](deploy-postgresql-database.md), put a model behind the app with [how to deploy Ollama](host-ollama-in-the-cloud.md), or split heavy logic into an API with [the FastAPI guide](deploy-fastapi-app.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a FastAPI app](deploy-fastapi-app.md)
- [How to host Ollama in the cloud](host-ollama-in-the-cloud.md)
- [How to deploy LiteLLM proxy](deploy-litellm-proxy.md)
- [How to deploy MinIO (S3-compatible object storage)](deploy-minio.md)
