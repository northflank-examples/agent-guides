<!--
agent-guide-meta:
  id: NF-STK-CREWAI
  description: "How to deploy CrewAI on your own infrastructure: a crew behind a FastAPI endpoint with a bearer token, plus a one-off or scheduled job, LLM keys as secrets."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/jobs/check-repository-access-for-jobs
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://docs.crewai.com/en/installation
    - https://docs.crewai.com/en/concepts/crews
    - https://docs.crewai.com/en/concepts/llms
    - https://docs.crewai.com/en/learn/kickoff-async
    - https://docs.crewai.com/en/concepts/memory
    - https://docs.crewai.com/en/telemetry
    - https://pypi.org/project/crewai/
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/project/jobs/run-job
    - https://northflank.com/docs/v1/api/project/jobs/get-run-details
    - https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-JOB-004, NF-STK-LANGGRAPH, NF-DEP-006, NF-JOB-002]
-->

# How to deploy CrewAI

To deploy CrewAI, package your crew in a Python 3.10–3.13 Docker image and run it in one of two shapes: a small FastAPI service that calls `crew.akickoff(inputs=...)` behind a bearer token, or a job that runs the crew once (or on a schedule) and exits. Either way, model keys such as `OPENAI_API_KEY` and the `MODEL` name come from secrets at runtime, never from a committed `.env`.

This guide does both on Northflank with the CLI: a secret group for the LLM key and an API token, a service built from your repo that answers `POST /kickoff`, and a job from the same image for long crews. You end with an HTTPS endpoint that returns crew output only to authorised requests, and a job run that reports `SUCCESS`. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy CrewAI

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a CrewAI project and your model key is set, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Add `app/api.py` (FastAPI, `POST /kickoff`, `GET /health`), `app/run_once.py` and a `Dockerfile` that starts uvicorn on `0.0.0.0:$PORT`, then push.
4. Create a project and a secret group with `OPENAI_API_KEY`, `MODEL` and a generated `CREW_API_TOKEN`, deploy the API with `northflank put service combined`, and create a manual job from the same repo with `northflank put job`.
5. Check that the live build is your pushed commit, `/kickoff` returns 401 without the token and crew output with it, and a job run ends in `SUCCESS`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Service or job: where should a crew run?

CrewAI is a Python framework, not a server. `crewai run` executes a crew in your terminal and exits, and the docs' hosted option is CrewAI's own commercial platform. To self-host, you choose how the crew gets triggered:

- **An HTTP service** suits crews that other apps call with inputs and that finish in seconds to a couple of minutes. The service stays up, so the import cost and model clients are paid once.
- **A job** suits crews that research, write or crawl for many minutes, or that should run every morning. It runs to completion with its own status, logs and retry settings, and no HTTP connection has to stay open.

This guide builds both from one image. That's also the answer to where to deploy CrewAI: something that runs a Python process for as long as a crew takes, injects LLM keys at runtime and can run the same code on a schedule. Serverless functions often cap execution time below what a multi-agent crew needs. A VM works if you add a process manager, TLS and cron yourself. A container platform with services and jobs covers both shapes.

## Prerequisites

- **A crew that runs locally.** CrewAI needs Python `>=3.10,<3.14`:
  ```bash
  crewai run
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **A model provider key** exported in your shell, never in the repo (`OPENAI_API_KEY` below). For another provider, use its variable (`ANTHROPIC_API_KEY`, `GEMINI_API_KEY`…) and a matching `MODEL` such as `anthropic/<model-id>`. CrewAI reads `MODEL` in `provider/model-id` form.

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
grep -hiE '^[[:space:]"]*crewai' $(ls requirements.txt pyproject.toml 2>/dev/null) /dev/null || echo "crewai: missing"
ls requirements.txt pyproject.toml uv.lock poetry.lock Dockerfile 2>/dev/null
[ -n "$OPENAI_API_KEY" ] && echo "OPENAI_API_KEY: set" || echo "OPENAI_API_KEY: missing"

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId crewai-app -o json >/dev/null 2>&1 \
  && echo "project: crewai-app exists and will be reused" \
  || echo "project: crewai-app will be created"
```

- If `crewai` says `missing`, this isn't a CrewAI project. Stop and use a guide for your framework.
- The lockfile tells you the package manager. Step 5 installs from `requirements.txt` with pip; for a `uv` or Poetry project, export one (`uv export --no-hashes > requirements.txt`) or swap the install command.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the key says `missing`, ask the user to export it. Never print its value.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "crewai-app", "region": "europe-west" }'
export PROJECT_ID=crewai-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

Names used below: secret group `crew-env`, service `crew-api`, job `crew-run`.

## Step 5: Add an API entry point and a one-shot runner

If your crew came from the classic scaffold (`crewai create crew <name> --classic`), you already have a class with a `crew()` method; import it in place of `build_crew` below. Otherwise, define the crew in Python. `app/crew.py`:

```python
import os
from crewai import Agent, Crew, LLM, Process, Task

def build_crew() -> Crew:
    llm = LLM(model=os.environ["MODEL"])
    researcher = Agent(role="Researcher", goal="Find the most important facts about {topic}",
                       backstory="You check sources and keep only what matters.", llm=llm)
    writer = Agent(role="Writer", goal="Turn research on {topic} into a short brief",
                   backstory="You write plainly for busy engineers.", llm=llm)
    research = Task(description="Research {topic}.", expected_output="Ten factual bullet points.", agent=researcher)
    brief = Task(description="Write a brief on {topic} from the research.",
                 expected_output="Three short paragraphs in Markdown.", agent=writer)
    return Crew(agents=[researcher, writer], tasks=[research, brief], process=Process.sequential)
```

`app/api.py` builds a fresh crew per request, so concurrent requests don't share state, and uses `akickoff`, which the CrewAI docs recommend for concurrent workloads:

```python
import os, secrets
from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel
from app.crew import build_crew

app = FastAPI()

class KickoffRequest(BaseModel):
    topic: str

@app.get("/health")
def health():
    return {"status": "ok"}  # no model call

@app.post("/kickoff")
async def kickoff(req: KickoffRequest, authorization: str = Header(default="")):
    if not secrets.compare_digest(authorization, "Bearer " + os.environ["CREW_API_TOKEN"]):
        raise HTTPException(status_code=401, detail="unauthorized")
    result = await build_crew().akickoff(inputs={"topic": req.topic})
    return {"result": result.raw}
```

`app/run_once.py` is the job entry point. It exits non-zero on failure, so the job run shows `FAILED`:

```python
import os, sys
from app.crew import build_crew

if __name__ == "__main__":
    topic = os.environ.get("CREW_TOPIC")
    if not topic:
        sys.exit("CREW_TOPIC is not set")
    print(build_crew().kickoff(inputs={"topic": topic}).raw)
```

Add an empty `app/__init__.py`, then the `requirements.txt` and `Dockerfile`:

```text
crewai==<VERSION>   # pin what you tested, e.g. the version from `pip show crewai`
fastapi
uvicorn[standard]
```

```dockerfile
FROM python:3.12-slim
WORKDIR /srv
ENV PYTHONUNBUFFERED=1
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["sh", "-c", "exec uvicorn app.api:app --host 0.0.0.0 --port ${PORT:-8000}"]
```

Northflank builds what's in your repository, not the files on your machine, so commit and push them. Make sure `.env` is in `.gitignore`.

```bash
git add app/__init__.py app/crew.py app/api.py app/run_once.py requirements.txt Dockerfile
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create the secret group

```bash
CREW_API_TOKEN=$(openssl rand -hex 32)
jq -n --arg oai "$OPENAI_API_KEY" --arg tok "$CREW_API_TOKEN" '{
  name: "crew-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  secrets: { variables: {
    OPENAI_API_KEY: $oai, MODEL: "openai/<model-id>",
    CREW_API_TOKEN: $tok, CREWAI_DISABLE_TELEMETRY: "true"
  } }
}' > crew-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file crew-env.json >/dev/null && rm crew-env.json
```

Replace `<model-id>` with the model you tested. The group is unrestricted, so both the service and the job read it. `CREWAI_DISABLE_TELEMETRY` turns off CrewAI's anonymous usage telemetry; drop it if you want to keep telemetry on. Keep `CREW_API_TOKEN` in this shell for the checks and tell the user where it's stored.

## Step 7: Deploy the API service

Write `crew-api.json`:

```json
{
  "name": "crew-api",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "8000" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/health", "port": 8000,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

Use the same repository URL, provider and branch as in step 4. `billing.buildPlan` is optional; leave it out to use the default build plan, and only set it if builds run out of memory, using a build plan your team has access to (an unavailable one fails with `Build plan not found`).

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file crew-api.json

OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId crew-api --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId crew-api --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId crew-api --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. Don't continue if the guard printed `ROLLOUT_NOT_COMPLETE`. `/health` doesn't call a model, so a slow or rate-limited provider can't take the service out of rotation.

## Step 8: Create the crew job

Write `crew-run.json`. It builds the same repo and overrides the command. `put job` makes a manual job because `settings.cron` is absent:

```json
{
  "name": "crew-run",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "settings": { "backoffLimit": 0, "activeDeadlineSeconds": 3600, "runOnSourceChange": "never" },
  "deployment": {
    "vcs": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
    "docker": { "configType": "customCommand", "customCommand": "python -m app.run_once" }
  },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } }
}
```

Use the same repository as the service. `activeDeadlineSeconds` stops a runaway crew after an hour. `backoffLimit: 0` means a failed run isn't retried automatically, which avoids paying for the same failing crew twice.

## Step 9: Build and run the job

```bash
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file crew-run.json

JOK=0
for i in $(seq 1 90); do
  JB=$(northflank get job builds --projectId "$PROJECT_ID" --jobId crew-run --noDefaults -o json \
       | jq -r '(.data // .).builds | sort_by(.createdAt) | last | .status // "NONE"')
  echo "job build=$JB"
  case "$JB" in
    SUCCESS) JOK=1; break ;;
    FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo JOB_BUILD_FAILED; break ;;
    NONE) [ "$i" = 6 ] && northflank start job build --projectId "$PROJECT_ID" --jobId crew-run --noDefaults -o json --input '{}' ;;
  esac
  sleep 10
done
[ "$JOK" = 1 ] || { echo "JOB_BUILD_NOT_COMPLETE: stop here and read the job build logs"; false; }
```

Only start a run if the guard passed:

```bash
RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId crew-run --noDefaults -o json \
  --input '{"runtimeEnvironment": {"CREW_TOPIC": "vector databases"}}' | jq -r '(.data // .).id')
RUN_DONE=0
for i in $(seq 1 360); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId crew-run --runId "$RUN_ID" --noDefaults -o json)
  echo "run=$(echo "$R" | jq -r '(.data // .).status')"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { RUN_DONE=1; break; }
  sleep 10
done
[ "$RUN_DONE" = 1 ] || echo "RUN_NOT_CONCLUDED: the run is still going or the loop timed out; don't report success"
northflank get job logs --projectId "$PROJECT_ID" --jobId crew-run --runId "$RUN_ID" --lineLimit 200
```

Each run can take a different `CREW_TOPIC`. To run a crew on a schedule, create a second job from the same file with a different `name` and a `settings.cron` block (for example `{"schedule": "0 7 * * 1", "concurrencyPolicy": "forbid"}`) and set `CREW_TOPIC` in its `runtimeEnvironment`. See [run an image once or on a schedule](https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule).

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId crew-api --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The service is healthy:
   ```bash
   curl -s "https://$HOST/health"   # {"status":"ok"}
   ```
3. An unauthenticated kickoff is refused:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' -X POST "https://$HOST/kickoff" \
     -H 'Content-Type: application/json' -d '{"topic":"test"}'   # 401
   ```
4. An authorised kickoff returns crew output (this calls the model and takes a while):
   ```bash
   curl -s --max-time 300 -X POST "https://$HOST/kickoff" -H "Authorization: Bearer $CREW_API_TOKEN" \
     -H 'Content-Type: application/json' -d '{"topic":"vector databases"}' | jq -r '.result' | head -n 20
   ```
5. The job run from step 9 concluded with `status == "SUCCESS"`, and the logs printed in step 9 end with the brief:
   ```bash
   northflank get job run --projectId "$PROJECT_ID" --jobId crew-run --runId "$RUN_ID" --noDefaults -o json | jq -r '(.data // .).status'
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `KeyError: 'MODEL'` or an auth error from the provider | Secret group missing a key, or `MODEL` doesn't match the key's provider | Check `crew-env`; use `provider/model-id` form |
| `/kickoff` returns 502/504 after a long wait | Request held open longer than the ingress allows | Move that crew to the job, or return a run ID and do the work in the background |
| Build fails resolving `crewai` | Python version outside `>=3.10,<3.14` | Keep `python:3.12-slim` or another supported version |
| Job run `FAILED` with `CREW_TOPIC is not set` | Run started without the variable | Pass it in `--input` as in step 9 |
| Memory errors or `Permission denied` under `.crewai` | Crew uses `memory=True` and the default local store | Set `CREWAI_STORAGE_DIR` to a [volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) path, or use an external memory backend |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` (and the job's `deployment.vcs`) as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service or job until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Never print `OPENAI_API_KEY` (or other provider keys) or `CREW_API_TOKEN`. Build JSON with `jq --arg` and delete temp files.
- Each kickoff costs model tokens. Run the authorised check once, not in a loop, and keep the job's `backoffLimit` at 0 unless the user asks for retries.
- Never remove the bearer-token check to make a test pass.
- Get explicit confirmation before deleting the service, jobs or secret group.

## FAQ

### Where should I deploy CrewAI?
Anywhere that can run a Python container for as long as your crew takes, inject LLM keys at runtime and, ideally, run it on a schedule. That means a long-lived service for interactive calls and a job for long or batch crews. A container platform that runs both from one image is the least work. Serverless functions often time out on multi-agent runs.

### Do I need CrewAI's hosted platform to deploy a crew?
No. The open-source `crewai` package runs anywhere Python does. The hosted platform adds a UI, tracing and managed deployment, but a crew in your own container behaves the same.

### Does CrewAI need a database?
Not by default. Crews are stateless between runs unless you enable memory, which stores data in a local LanceDB directory (`./.crewai/memory`, or `$CREWAI_STORAGE_DIR/memory`). If you use memory in production, put that directory on a volume (single instance) or use an external backend.

### How do I stream progress instead of waiting?
Send progress over Server-Sent Events from FastAPI, or have `/kickoff` start a background task that stores the result and returns a run ID. The patterns are in [how to deploy AI agents in production](deploy-ai-agents-in-production.md).

## Conclusion

You now have a CrewAI crew behind an authenticated HTTPS endpoint, plus a job that runs the same crew to completion with its own status and logs, with LLM keys held in a secret group.

**Next:** add a cron job for scheduled crews, then a custom domain with [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy AI agents in production](deploy-ai-agents-in-production.md)
- [How to deploy a LangGraph agent](deploy-langgraph-agent.md)
- [How to deploy a FastAPI app](deploy-fastapi-app.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
