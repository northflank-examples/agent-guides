<!--
agent-guide-meta:
  id: NF-DEP-008
  description: "How to deploy a Rails app with Docker: Puma and a Sidekiq worker from one repo, managed Postgres and Redis, one shared secret group and migrations as a job."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/project/jobs/run-job
    - https://northflank.com/docs/v1/application/secure/manage-secret-groups
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/application/release/run-migrations
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-redis-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
  related: [NF-DB-001, NF-DB-003, NF-JOB-002, NF-DEP-009]
-->

# How to deploy a Rails app

To deploy a Rails app to production, build the Rails-generated production `Dockerfile`, run Puma and a Sidekiq worker as two processes from the same image, connect PostgreSQL and Redis through `DATABASE_URL` and `REDIS_URL`, supply `RAILS_MASTER_KEY` at runtime, and run `db:migrate` once per release. Giving the web server and the worker separate copies of that config is how they drift apart.

This guide does it on Northflank with the CLI: managed Postgres and Redis, one secret group shared by both services, `web` and `worker` deployed from one repo with only the start command changed, and migrations as a one-off job. You end with a public HTTPS URL where `/up` returns 200 and Sidekiq processes an enqueued job. Your Git repo must already be linked. Every step is a command a developer or a coding agent can run as-is.

## TL;DR: how to deploy a Rails app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a Rails app with Sidekiq and the production `Dockerfile`, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Create Postgres and Redis with `northflank put addon`, and one secret group that links `DATABASE_URL` and `REDIS_URL` and holds `RAILS_MASTER_KEY`.
4. Push, then deploy `web` (public port, `/up` health check) and `worker` (`bundle exec sidekiq`, no ports) with `northflank put service combined`.
5. Run `bin/rails db:migrate` as a manual job.
6. Check that the live build is your pushed commit, `/up` returns 200, and an enqueued job raises Sidekiq's processed count.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why one secret group for two services?

A secret group is a set of variables that any number of services and jobs inherit. Addon credentials can be linked into it with aliases, so Rails sees the names it expects. With one group, the web service, the worker and the migration job all read identical values, and rotating the master key means editing one place and restarting the dependents.

This shape also answers where to deploy a Rails app: the host has to keep Puma and Sidekiq running as long-lived processes, provide Postgres and Redis, and run `db:migrate` once per release. Static and serverless hosts are built for short request-scoped functions and don't suit a Sidekiq worker. A raw VM (the Capistrano or Kamal style) runs everything, but you manage the server, TLS and databases yourself. A container platform with managed databases runs the same `Dockerfile` as two services and a job.

## Prerequisites

- **A Rails app that boots locally:**
  ```bash
  bin/rails runner 'puts Rails.version' && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The app is configured for Sidekiq and the proxy.** This is standard Rails configuration:
  - `Gemfile` has `sidekiq`, and `config/application.rb` or `production.rb` sets `config.active_job.queue_adapter = :sidekiq`. Rails 8 defaults to Solid Queue, so change it explicitly.
  - `config/environments/production.rb` sets `config.assume_ssl = true` (TLS ends at the load balancer) and logs to STDOUT.
  - If `config/database.yml` defines extra production databases (Rails 8's `cache`, `queue`, `cable`), give each one `url: <%= ENV["DATABASE_URL"] %>` or remove the ones you don't use. `DATABASE_URL` only replaces the primary database.
  - A `/up` health route exists (Rails 7.1+ generates it).
- **`config/master.key` exists locally** and is git-ignored. **This step needs a human** if it's missing: ask the user for the key. Don't regenerate credentials.

Names used below: project `rails-app`, addons `rails-db` and `rails-redis`, secret group `rails-env`, services `web` and `worker`, job `rails-migrate`.

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
grep -E "^\s*gem ['\"](rails|sidekiq|pg)['\"]" Gemfile || echo "Gemfile: missing"
grep -E '^(EXPOSE|ENTRYPOINT|CMD)' Dockerfile
ls Gemfile.lock Dockerfile bin/docker-entrypoint config/master.key 2>/dev/null
git check-ignore -q config/master.key && echo "master.key: ignored by git"

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId rails-app -o json >/dev/null 2>&1 \
  && echo "project: rails-app exists and will be reused" \
  || echo "project: rails-app will be created"
```

- If `gem "rails"` doesn't print, this isn't a Rails app. Stop and use a guide for your framework. If `sidekiq` or `pg` doesn't print, add it (see the prerequisites).
- The `Dockerfile` must be the Rails production one (Rails 7.1 and later generate it with `bin/docker-entrypoint`). Check which port it exposes: Rails 7.1 and 7.2 use `3000`, Rails 8 uses `80` (Thruster). This guide shows `3000`; change `internalPort` and the health check port in step 7 if yours differs.
- If there's no `Dockerfile`, generate one with Rails' generator or the `dockerfile-rails` gem rather than writing one by hand.
- If `config/master.key` doesn't list, **a human has to provide it**: ask the user for the key. Don't regenerate credentials. It must say `ignored by git`.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "rails-app", "region": "europe-west" }'
export PROJECT_ID=rails-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

The job in step 10 uses the same repository; `northflank get job repository-access` takes the same body.

## Step 5: Create Postgres and Redis

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "rails-db", "type": "postgresql", "version": "16-latest",
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "rails-redis", "type": "redis", "version": "7-latest",
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'

for ADDON in rails-db rails-redis; do
  OK=0
  for i in $(seq 1 90); do
    A=$(northflank get addon --projectId "$PROJECT_ID" --addonId "$ADDON" --noDefaults -o json | jq -r '(.data // .).status')
    echo "$ADDON=$A"
    case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
    sleep 10
  done
  [ "$OK" = 1 ] || break
done
[ "$OK" = 1 ] || { echo "ADDONS_NOT_READY: stop here and check the addon in the UI"; false; }
```

Redis defaults to the `noeviction` memory policy (`typeSpecificSettings.redisMaxMemoryPolicy`), which is what Sidekiq needs. An evicting policy would silently drop queued jobs. The policy can't be changed after creation. For more Redis options, see [how to deploy Redis](deploy-redis.md).

## Step 6: Create the shared secret group

Read the master key into a variable and build the JSON with `jq`, so the key never appears in your command line or output:

```bash
RAILS_MASTER_KEY=$(tr -d '\n' < config/master.key)
jq -n --arg key "$RAILS_MASTER_KEY" '{
  name: "rails-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [
    { addonId: "rails-db",    keys: [ { keyName: "POSTGRES_URI",     aliases: ["DATABASE_URL"] } ] },
    { addonId: "rails-redis", keys: [ { keyName: "REDIS_MASTER_URL", aliases: ["REDIS_URL"] } ] }
  ],
  secrets: { variables: {
    RAILS_MASTER_KEY: $key, RAILS_ENV: "production",
    RAILS_LOG_TO_STDOUT: "1", RAILS_SERVE_STATIC_FILES: "1"
  } }
}' > rails-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file rails-env.json >/dev/null && rm rails-env.json
unset RAILS_MASTER_KEY
```

`REDIS_MASTER_URL` and `POSTGRES_URI` are the documented connection-string keys. Sidekiq reads `REDIS_URL` by default. `secretType: "environment"` means none of this reaches the build. That matches the Rails `Dockerfile`, which precompiles assets with `SECRET_KEY_BASE_DUMMY=1`.

If you change the group later, restart `web` and `worker` (`northflank restart service`) so they pick up the new values.

## Step 7: Deploy the web service

Northflank builds what's on the remote branch, not your local working tree. Commit and push the Sidekiq and production config changes from the prerequisites first (never `config/master.key`):

```bash
git add Dockerfile Gemfile Gemfile.lock config/application.rb config/environments/production.rb config/database.yml
git commit -m "Add Northflank deployment files"
git push
```

Write `web.json`:

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-50" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 3000, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/up", "port": 3000,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file web.json
```

Use the same repository URL, provider and branch as in step 4 here, in `worker.json` and in `migrate.json`. The default command runs the image's `ENTRYPOINT` (`bin/docker-entrypoint`) and `CMD` (`rails server`). Rails' generated entrypoint runs `db:prepare` before starting the server. That's fine for a single instance, but step 10 still runs migrations as a separate job, so you get a clear result and can run more than one web instance safely.

## Step 8: Deploy the Sidekiq worker

Same repo, same `Dockerfile`, no ports. Only the command changes. `configType: "customCommand"` replaces `CMD` and keeps the image's `ENTRYPOINT`, so `bin/docker-entrypoint` still sets up the environment. Write `worker.json`:

```json
{
  "name": "worker",
  "billing": { "deploymentPlan": "nf-compute-50" },
  "deployment": { "instances": 1, "docker": { "configType": "customCommand", "customCommand": "bundle exec sidekiq" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } }
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file worker.json
```

This builds the image twice, once per service. To build once, use a build service plus deployment services that reference it (`deployment.internal`). See [how to deploy a background worker](deploy-celery-in-production.md).

## Step 9: Wait for both rollouts

```bash
for SVC in web worker; do
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
  [ "$OK" = 1 ] || break
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

If the loop prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs of the service it stopped on: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web --lineLimit 200` (or `--serviceId worker`). The troubleshooting table covers the common causes.

`COMPLETED` means rolled out and running. For the worker, that means Sidekiq started, not that it exited.

## Step 10: Run migrations as a job

Write `migrate.json`. `put job` makes a manual job because `settings.cron` is absent (`put job manual` is deprecated in the API):

```json
{
  "name": "rails-migrate",
  "billing": { "deploymentPlan": "nf-compute-50" },
  "settings": { "backoffLimit": 0, "activeDeadlineSeconds": 900, "runOnSourceChange": "never" },
  "deployment": {
    "vcs": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
    "docker": { "configType": "customCommand", "customCommand": "bundle exec rails db:migrate" }
  },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } }
}
```

```bash
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file migrate.json
```

Wait for the job's image. If no build appears within a minute, the loop starts one:

```bash
OK=0
for i in $(seq 1 90); do
  JB=$(northflank get job builds --projectId "$PROJECT_ID" --jobId rails-migrate --noDefaults -o json \
       | jq -r '(.data // .).builds | sort_by(.createdAt) | last | .status // "NONE"')
  echo "job build=$JB"
  case "$JB" in
    SUCCESS) OK=1; break ;;
    FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo JOB_BUILD_FAILED; break ;;
    NONE) [ "$i" = 6 ] && northflank start job build --projectId "$PROJECT_ID" --jobId rails-migrate --noDefaults -o json --input '{}' ;;
  esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "JOB_BUILD_NOT_COMPLETE: stop here and read the job build logs"; false; }
```

Only if that loop succeeded, run the migration:

```bash
RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId rails-migrate --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 90); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId rails-migrate --runId "$RUN_ID" --noDefaults -o json)
  S=$(echo "$R" | jq -r '(.data // .).status'); echo "run=$S"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$S" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId rails-migrate --runId "$RUN_ID" --lineLimit 200
[ "$OK" = 1 ] || { echo "MIGRATION_NOT_COMPLETE: stop here and read the logs above"; false; }
```

## How to know it worked

`exec` drops output without a TTY, so wrap it with `script` (macOS shown; Linux: `script -qefc '<cmd>' /dev/null`). You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. `curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/up"` prints `200`.
3. The migration run's `status` is `SUCCESS`.
4. The worker log shows Sidekiq connected to Redis and no connection errors:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId worker --lineLimit 200 --textIncludes "Sidekiq"
   ```
5. An enqueued job is processed. Record the processed count, enqueue one of the app's jobs from the **web** container (replace `<SomeJob>` with an existing class in `app/jobs`), wait, and compare:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId web \
     --shell-cmd 'sh -c' --cmd '"bin/rails runner \"puts Sidekiq::Stats.new.processed; <SomeJob>.perform_later\""'
   sleep 15
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId web \
     --shell-cmd 'sh -c' --cmd '"bin/rails runner \"puts Sidekiq::Stats.new.processed\""'
   ```
   The second number is higher than the first. Because the job ran in the worker, this proves both services share the same Redis.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` (and to `deployment.vcs` in `migrate.json`) as `"vcsLinkId": "<ID>"` |
| `ActiveSupport::MessageEncryptor::InvalidMessage` at boot | Wrong or missing `RAILS_MASTER_KEY` | Re-create the group from the correct `config/master.key`; restart both services |
| Worker log: `Error connecting to Redis on localhost:6379` | `REDIS_URL` not injected | Check the alias in `rails-env`; restart `worker` |
| Worker log: `SSL_connect … certificate verify failed` | Redis URL is `rediss://` with a certificate the client rejects | In `config/initializers/sidekiq.rb`, pass `ssl_params: { verify_mode: OpenSSL::SSL::VERIFY_NONE }` for both server and client, or create Redis with `"tlsEnabled": false` (can't be changed later; ask first) |
| Jobs enqueue but never run | Active Job adapter is still `:solid_queue` or `:async` | Set `config.active_job.queue_adapter = :sidekiq` |
| `/up` health check fails with 301 | `force_ssl` redirecting plain-HTTP probes | Set `config.assume_ssl = true` |
| `ActiveRecord::ConnectionNotEstablished` for `cache`/`queue` | Rails 8 extra databases point to localhost | Add `url: <%= ENV["DATABASE_URL"] %>` to each, or remove them |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the services or the job until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print, commit or paste `RAILS_MASTER_KEY`, `DATABASE_URL` or `REDIS_URL`. Read the key from the file into `jq --arg` and unset it afterwards.
- Don't regenerate `config/credentials.yml.enc` or the master key to get past an error. That loses every stored secret. Ask the user.
- Take a backup before running migrations on a database with data (see [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data)).
- Don't scale the worker to zero or delete Redis while jobs are queued without asking. Queued jobs live in Redis.

## FAQ

### Where should I deploy my Rails app?
On a host that runs long-lived containers for both the web server and the worker, with Postgres and Redis close by. A container platform with managed databases gives you that from the Rails `Dockerfile` with no server to maintain. A VM gives you full control in exchange for patching, TLS and backups.

### Can the web and worker share one service?
Not cleanly. Each service runs one main process. Running Puma and Sidekiq in one container hides worker crashes and makes them scale together. Two services from one repo is the standard pattern.

### How do I scale Sidekiq separately?
Change `deployment.instances` on `worker` only, or tune Sidekiq's concurrency. See [autoscale deployments](https://northflank.com/docs/v1/application/scale/autoscale-deployments).

### Is `RAILS_MASTER_KEY` needed at build time?
Not with the Rails-generated `Dockerfile`, which uses `SECRET_KEY_BASE_DUMMY=1` for `assets:precompile`. If your build needs credentials, use a secret group with `secretType: "arguments"` and treat it as sensitive.

### Do I need the scheduler (sidekiq-cron, whenever)?
Sidekiq-cron runs inside the worker. For system cron, use a Northflank cron job instead. See [run an image once or on a schedule](https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule).

## Conclusion

You now have a Rails app with a web service and a Sidekiq worker built from one repo, sharing Postgres, Redis and the master key through a single secret group, with migrations running as a tracked job.

**Next:** move to separate staging and production with [release through staging](https://northflank.com/docs/v1/application/use-cases/release-through-staging), and schedule backups with [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy a Laravel project](deploy-laravel-project.md)
