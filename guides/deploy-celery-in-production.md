<!--
agent-guide-meta:
  id: NF-JOB-002
  description: "How to deploy Celery in production for Django, Flask or FastAPI: port-less workers, a single beat instance and a managed Redis broker, with a task check."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://docs.celeryq.dev/en/stable/userguide/workers.html
    - https://docs.celeryq.dev/en/stable/userguide/periodic-tasks.html
    - https://docs.celeryq.dev/en/stable/userguide/configuration.html
    - https://docs.celeryq.dev/en/stable/django/first-steps-with-django.html
    - https://flask.palletsprojects.com/en/stable/patterns/celery/
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/use-cases/run-background-tasks
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-redis-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
  related: [NF-DB-003, NF-DEP-006, NF-STK-FLASK, NF-DEP-007, NF-DB-001]
-->

# How to deploy Celery in production

To deploy Celery in production, run the worker and Celery beat as separate long-running processes from the same image as your web app, with no HTTP port or health check, share one broker URL (Redis or RabbitMQ) across web, worker and beat, run beat as exactly one instance, and set worker concurrency and shutdown grace period explicitly. A second beat schedules every task twice.

This guide does it on Northflank with the CLI for Django, Flask or FastAPI: a managed Redis broker linked through one secret group, and the worker and beat as two port-less services built from your existing `Dockerfile`. You end with a scalable worker whose logs show a test task `succeeded` and a single beat sending due tasks. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy Celery in production

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo has a Celery app, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Read the broker from `CELERY_BROKER_URL`, add a test task, and push.
4. Create Redis with `northflank put addon` (type `redis`) and link `REDIS_MASTER_URL` as `CELERY_BROKER_URL` in a secret group restricted to `web`, `celery-worker` and `celery-beat`.
5. Deploy `celery-worker` (`"ports": []`, `celery -A proj worker`) and `celery-beat` (exactly one instance) with `northflank put service combined`.
6. Check that both live builds are your pushed commit, then enqueue a test task and find `succeeded` in the worker logs.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy Celery on Northflank

Celery needs long-running processes with no HTTP port, a broker on a private network, and the same code and secrets as the web app. Serverless and static hosts don't run processes that wait on a queue. A raw VM or Kubernetes cluster can, but you write the process supervision, restarts and secret distribution yourself. A container platform runs the worker and beat as their own services from your existing image, each with its own logs, restarts and instance count.

Two rules shape the setup. A worker that copies the web app's port and HTTP health check gets restarted forever, because nothing listens on that port, so workers have `"ports": []`. And the Celery docs say only one beat scheduler may run at a time, otherwise tasks are scheduled twice, so beat is a separate service pinned to one instance rather than an extra process in every worker.

## Prerequisites

- **A Celery app that loads locally.** `report` imports the app without connecting to the broker; replace `proj` with your `-A` value (see step 5):
  ```bash
  pip install -r requirements.txt && CELERY_BROKER_URL=redis://localhost:6379/0 celery -A proj report >/dev/null && echo CELERY_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **If the web app that enqueues tasks already runs on Northflank**, note its service ID and project. Step 7 shares the broker URL with it; the worker and beat don't need it.

Below, replace `proj` with your `-A` value. Names: project `celery-app` (use the web app's project instead if it already runs on Northflank), addon `celery-broker`, secret group `celery-env`, services `celery-worker` and `celery-beat`.

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
grep -ioE '\b(celery(\[redis\])?|redis|django|flask|fastapi)\b' requirements.txt pyproject.toml 2>/dev/null | sort -u
ls requirements.txt pyproject.toml poetry.lock uv.lock Pipfile.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId celery-app -o json >/dev/null 2>&1 \
  && echo "project: celery-app exists and will be reused" \
  || echo "project: celery-app will be created"
```

- If `celery` is missing, this isn't a Celery app. Stop and add Celery to the app first, or use a guide for your framework. The other names tell you which row of the step 5 table applies.
- The files tell you the package manager. The worker and beat reuse the repo's `Dockerfile`, so it must install `celery[redis]` with whichever one you use.
- If there's no `Dockerfile`, write one that installs your dependencies and copies the code; the worker and beat only override its command.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused (reuse the web app's project if it's on Northflank). Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "celery-app", "region": "europe-west" }'
export PROJECT_ID=celery-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3. If the web app already runs on Northflank, skip `create` and set `PROJECT_ID` to its project. Then set its service ID, which step 7 uses: `export WEB_ID=<WEB_SERVICE_ID>`. Leave `WEB_ID` unset if there's no web app on Northflank yet.

## Step 5: Read the broker URL from the environment and add a test task

Make the Celery app read its broker from `CELERY_BROKER_URL` and give it a harmless test task:

| Framework | Celery app | `-A` value |
|---|---|---|
| Django | `proj/celery.py` with `config_from_object("django.conf:settings", namespace="CELERY")` and `autodiscover_tasks()`; in `settings.py`, `CELERY_BROKER_URL = os.environ["CELERY_BROKER_URL"]` | `proj` |
| Flask | `celery_init_app(app)` from the Flask Celery pattern with `CELERY=dict(broker_url=os.environ["CELERY_BROKER_URL"])`, plus a `make_celery.py` for app factories | `make_celery` |
| FastAPI or plain Python | `app/worker.py`: `celery = Celery("app", broker=os.environ["CELERY_BROKER_URL"])` | `app.worker` |

```python
from celery import shared_task

@shared_task
def ping():
    return "pong"
```

The broker URL uses `rediss://` when Redis TLS is on. Celery's Redis TLS setting is `broker_use_ssl` with `ssl_cert_reqs` (Django: `CELERY_BROKER_USE_SSL`), for example `{"ssl_cert_reqs": ssl.CERT_REQUIRED}` when the URL starts with `rediss://`.

Make sure the repo's `Dockerfile` installs `celery[redis]` alongside the app. The worker and beat reuse it and only override the command.

Northflank builds what's in your repository, not the files on your machine, so commit and push the Celery files (adjust the paths to your framework):

```bash
git add proj/celery.py proj/tasks.py requirements.txt   # the Celery app, test task and dependency changes
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create the Redis broker and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "celery-broker",
  "type": "redis",
  "version": "latest",
  "tlsEnabled": true,
  "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 4096, "replicas": 1 },
  "typeSpecificSettings": { "redisMaxMemoryPolicy": "noeviction", "redisSentinelEnabled": false }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId celery-broker --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in
    running) OK=1; break ;;
    failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break ;;
  esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_RUNNING: stop here and check the addon in the UI"; false; }
```

Use `noeviction` for a broker: an eviction policy would silently drop queued tasks when memory fills. The policy can't be changed after creation. More options are in [how to deploy Redis](deploy-redis.md). To use RabbitMQ instead, run the same command and loop with `"type": "rabbitmq"`, `"version": "latest"` and no `typeSpecificSettings`, and link its `AMQP_CONNECTION_STRING` under the same alias in step 7 (the Dockerfile then needs plain `celery` rather than `celery[redis]`). More on RabbitMQ is in [how to deploy RabbitMQ](deploy-rabbitmq.md).

## Step 7: Link the broker URL into web, worker and beat

The three services need the same URL. Service IDs are slugified names, so you can list `celery-worker` and `celery-beat` before they exist.

**No web app on Northflank yet:** delete the `{ "id": "'"$WEB_ID"'", … }` line from the `nfObjects` list before running the first command, and skip the web app restart at the end of this step. When you deploy the web app later, add its service ID to `restrictions.nfObjects` of `celery-env` so it gets `CELERY_BROKER_URL` too.

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "celery-env",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": true, "nfObjects": [
    { "id": "'"$WEB_ID"'", "type": "service" },
    { "id": "celery-worker", "type": "service" },
    { "id": "celery-beat", "type": "service" } ] },
  "addonDependencies": [
    { "addonId": "celery-broker", "keys": [ { "keyName": "REDIS_MASTER_URL", "aliases": ["CELERY_BROKER_URL"] } ] }
  ]
}'
northflank get secret-link --projectId "$PROJECT_ID" --secretId celery-env --addonId celery-broker -o json \
  | jq '(.data // .) | {linked, linkedKeys, availableKeys}'
```

If `linkedKeys` is empty, pick the URL key from `availableKeys` and re-link with `northflank update secret-link … --input '{"keys":[{"keyName":"<KEY>","aliases":["CELERY_BROKER_URL"]}]}'`. If the worker also touches the database, add it to the group that holds `DATABASE_URL` too.

Then restart the web app so it picks up the new variable:

```bash
northflank restart service --projectId "$PROJECT_ID" --serviceId "$WEB_ID"
```

## Step 8: Deploy the Celery worker

Save as `worker.json`:

```json
{
  "name": "celery-worker",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": {
    "instances": 1,
    "docker": { "configType": "customCommand", "customCommand": "celery -A proj worker --loglevel=INFO --concurrency=2" },
    "gracePeriodSeconds": 120
  },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": []
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file worker.json
```

- Use the same repository URL, provider and branch as in step 4.
- `--concurrency` defaults to the number of CPUs Celery sees, which in a container may not match the plan's CPU allocation. Set it explicitly.
- On `SIGTERM`, a Celery worker finishes the tasks it's running before exiting (warm shutdown). `gracePeriodSeconds` (15–86400, default 30) is how long it gets before it's killed; set it above your longest task.
- `customCommand` runs in exec form. If the image has an `ENTRYPOINT`, or you need `$VARS`, use `"configType": "customEntrypointCustomCommand"` with `"customEntrypoint": "sh -c"` and the command in quotes. See [override command and entrypoint](https://northflank.com/docs/v1/application/run/override-command-entrypoint).

## Step 9: Deploy Celery beat as a single instance

Beat is long-running and must never have more than one copy. Save as `beat.json`:

```json
{
  "name": "celery-beat",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": {
    "instances": 1,
    "docker": { "configType": "customCommand", "customCommand": "celery -A proj beat --loglevel=INFO -s /tmp/celerybeat-schedule" }
  },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": []
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file beat.json
```

Beat stores last run times in a local file (`celerybeat-schedule`) and needs write access to it; `-s /tmp/…` keeps it writable for non-root images. That file doesn't survive a restart. With Django, `django-celery-beat` keeps schedules in the database instead: `celery -A proj beat -l INFO --scheduler django_celery_beat.schedulers:DatabaseScheduler`.

## Step 10: Wait for both rollouts and confirm there are no ports

```bash
ALL_OK=1
for SVC in celery-worker celery-beat; do
  OK=0
  for i in $(seq 1 90); do
    S=$(northflank get service --projectId "$PROJECT_ID" --serviceId "$SVC" --noDefaults -o json)
    B=$(echo "$S" | jq -r '(.data // .).status.build.status // "NONE"')
    D=$(echo "$S" | jq -r '(.data // .).status.deployment.status // "NONE"')
    echo "$SVC build=$B deployment=$D"
    case "$B" in FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo BUILD_FAILED; break;; esac
    [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
    [ "$B" = "SUCCESS" ] && [ "$D" = "COMPLETED" ] && { OK=1; break; }
    sleep 10
  done
  [ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: $SVC; stop here and read its build or deployment logs"; ALL_OK=0; }
  echo "$S" | jq '(.data // .).ports'
done
[ "$ALL_OK" = 1 ]
```

Don't continue if either service printed `ROLLOUT_NOT_COMPLETE`; read its build logs with `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId <SVC>`. `COMPLETED` means rolled out and running. Each ports list should be `[]`. If ports appear (Northflank can detect them from `EXPOSE`), remove them with `northflank patch service combined --projectId "$PROJECT_ID" --serviceId <SVC> --noDefaults -o json --input '{"ports": []}'` (the older `update service ports` endpoint is deprecated). Whether an empty list removes them hasn't been confirmed, so check again and tell the user if they remain.

## Step 11: Prove a task gets processed

Enqueue the test task with Celery's `call` command from inside the worker container, which already has the broker URL. `exec` drops output without a TTY, so wrap it with `script` (macOS shown; Linux: `script -qefc '<cmd>' /dev/null`):

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId celery-worker \
  --cmd 'celery -A proj call proj.tasks.ping'

northflank get service logs --projectId "$PROJECT_ID" --serviceId celery-worker --types runtime \
  --duration 600 --lineLimit 200 --direction backward --textIncludes "succeeded" -o json \
  | jq -r '(.data? // .)[] | "\(.ts) \(.log)"'
```

Celery usually logs `Task proj.tasks.ping[<id>] succeeded in …s`. Adjust the task name to your project. For beat, search its logs for `Sending due task` after the first schedule interval.

## Step 12: Scale the workers

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId celery-worker --noDefaults -o json \
  --input '{"deployment": {"instances": 3}}'
```

The `scale service` endpoint is deprecated, so this patches the service. For more CPU or memory per worker, patch `{"billing": {"deploymentPlan": "<PLAN>"}}`. Tasks must be safe to run on any worker. Never scale `celery-beat` above 1. More instances cost more, so confirm the count with the user.

## How to know it worked

You're done when all of these hold:

1. The live build of both services is the commit you pushed. Each line should show the same SHA as `git rev-parse HEAD`, with `SUCCESS`:
   ```bash
   git rev-parse HEAD
   for SVC in celery-worker celery-beat; do
     northflank get service builds --projectId "$PROJECT_ID" --serviceId "$SVC" --noDefaults -o json \
       | jq -r --arg s "$SVC" '(.data // .).builds | sort_by(.createdAt) | last | "\($s) \(.sha) \(.status)"'
   done
   ```
2. Both services report `status.deployment.status == "COMPLETED"` and an empty ports list.
3. The worker's logs contain the `succeeded` line for your test task, and any real side effect exists.
4. `celery-beat` has `deployment.instances == 1` and its logs show due tasks being sent.
5. `northflank get service health --projectId "$PROJECT_ID" --serviceId celery-worker --duration 600` shows `restartCount` unchanged across two reads a few minutes apart.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Worker restarts every minute or two | HTTP health check copied from the web app | `northflank patch service combined --serviceId <SVC> --input '{"healthChecks": []}'` |
| `Connection refused` or `KeyError: 'CELERY_BROKER_URL'` | Service not in the group's `restrictions.nfObjects` | Add it, then `northflank restart service` |
| TLS or `ssl_cert_reqs` error on startup | `rediss://` URL without TLS settings | Set `broker_use_ssl` as in step 5 |
| Periodic tasks run twice | Beat in more than one place (`-B` on workers, or 2 beat instances) | Run beat only in `celery-beat` with one instance |
| Tasks killed mid-run on deploy | Grace period too short | Raise `gracePeriodSeconds` |
| `Received unregistered task` | Worker built from an older commit or tasks not autodiscovered | Redeploy from the same branch; check imports |
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` in `worker.json` and `beat.json` as `"vcsLinkId": "<ID>"` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the services until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows both live builds are the commit you pushed.
- If the step 10 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs before changing anything.
- Don't copy the web app's `ports` or `healthChecks` into the worker or beat.
- Never print broker URLs or passwords; print key names only.
- Enqueue harmless test tasks only, never real customer work.
- Confirm instance counts and plans with the user. Never run beat with more than one instance.
- Prefer `northflank pause service` or scaling to 0 over deleting.

## FAQ

### Where should I deploy Celery?
Next to your web app, on a host that runs long-lived containers without a port and gives them a private broker. A container platform runs the worker and beat from the same image as the app, each restarted and scaled on its own. Serverless functions don't fit because the worker must keep a broker connection open.

### Should Celery beat be a service or a cron job?
Beat is a scheduler that runs continuously, so run it as a one-instance service. If you'd rather drop beat, a Northflank [cron job](https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule) can run a short command on a schedule that enqueues the task.

### Can I run beat inside the worker with `-B`?
The Celery docs say embedding beat with `celery worker -B` isn't recommended for production. Every worker instance would also start a scheduler.

### Redis or RabbitMQ?
Both are supported brokers. Redis is simpler and can double as the result backend; RabbitMQ is a dedicated message broker. Either works with this setup.

### I use Sidekiq, not Celery.
Sidekiq is Ruby's equivalent. See [how to deploy a Rails app](deploy-rails-app.md), which deploys a Sidekiq worker the same way.

### How do I autoscale workers?
See [autoscale deployments](https://northflank.com/docs/v1/application/scale/autoscale-deployments). Scaling on queue length isn't covered here.

## Conclusion

You now have Celery in production on Northflank: a managed Redis broker, its URL shared through one restricted secret group, a port-less worker you can scale, a single beat scheduler, and a log check that proves tasks complete.

**Next:** keep [the logs docs](https://northflank.com/docs/v1/application/observe/view-logs) handy. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy Redis](deploy-redis.md)
- [How to deploy RabbitMQ](deploy-rabbitmq.md)
- [How to deploy a Django project](deploy-django-project.md)
- [How to deploy a Flask app](deploy-flask-app.md)
- [How to deploy a FastAPI app](deploy-fastapi-app.md)
