<!--
agent-guide-meta:
  id: NF-DEP-009
  description: "How to deploy a Laravel project to production with Docker and managed MySQL, running the web server, queue worker and per-minute scheduler from one image."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/project/jobs/get-job-runs
    - https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/application/release/run-migrations
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-mysql-on-northflank
  related: [NF-DB-003, NF-JOB-002, NF-DEP-008]
-->

# How to deploy a Laravel project

To deploy a Laravel project to production, build one Docker image and run it three ways: a web server, a long-running `php artisan queue:work` worker, and `php artisan schedule:run` every minute. Connect all three to a managed MySQL database with the same `APP_KEY`, and run `php artisan migrate --force` as a separate step. Hosts that only serve web requests leave queued jobs and scheduled tasks silently unprocessed.

This guide does it on Northflank with the CLI: a web service on Apache, a queue worker service and a per-minute cron job, all from one repo, sharing MySQL and `APP_KEY` through one secret group, plus migrations as a one-off job. You end with a public HTTPS URL, passing scheduler runs and a queued test job the worker picks up. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to deploy a Laravel project

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a Laravel app, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Create MySQL with `northflank put addon` and one secret group linking its `HOST`, `PORT`, `DATABASE`, `USERNAME` and `PASSWORD` as `DB_*`, plus `APP_KEY`.
4. Push a `Dockerfile`, then deploy `web` (Apache on port 80) and `queue` (`php artisan queue:work`) with `northflank put service combined`, and a `scheduler` cron job (`* * * * *`, `php artisan schedule:run`, `concurrencyPolicy: forbid`) with `northflank put job`.
5. Run `php artisan migrate --force` as a manual job.
6. Check that the live build is your pushed commit, `/up` returns 200, the scheduler runs succeed and a dispatched test job shows up in the `queue` logs.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why three workloads?

Each Laravel process needs different handling:

- **Web** must always be up and behind a public port.
- **Queue worker** must always be up, with no port, and restart if it crashes.
- **Scheduler** must run once a minute and then exit.

Northflank maps these to a service, a service and a cron job. A cron job gives you a run history with `SUCCESS` or `FAILED` for every minute, which is much easier to check than a `cron` daemon hidden inside the web container.

Those three processes decide where to deploy a Laravel app. Shared PHP hosting serves web requests but usually has no way to keep a queue worker running. Static and serverless hosts are built for short request-scoped functions, so the worker and scheduler need somewhere else to live. A VM runs all three, but you maintain PHP, the web server, Supervisor, cron and MySQL yourself. A container platform runs them from one image, next to a managed database.

## Prerequisites

- **A Laravel app that runs locally:**
  ```bash
  composer install && php artisan --version && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The app trusts the load balancer**, so it generates `https://` URLs. In Laravel 11+, add this to `bootstrap/app.php`: `->withMiddleware(function (Middleware $middleware) { $middleware->trustProxies(at: '*'); })`.
- **The queue uses the database driver** (Laravel 11 default), with the `jobs` table migration present. If you'd rather use Redis, see [how to deploy Redis](deploy-redis.md) and set `QUEUE_CONNECTION=redis`.

Names used below: project `laravel-app`, addon `laravel-db`, secret group `laravel-env`, services `web` and `queue`, jobs `scheduler` and `migrate`.

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
jq -r '"laravel/framework: \(.require["laravel/framework"] // "missing")  php: \(.require.php // "unset")"' composer.json
ls composer.lock package-lock.json Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId laravel-app -o json >/dev/null 2>&1 \
  && echo "project: laravel-app exists and will be reused" \
  || echo "project: laravel-app will be created"
```

- If `laravel/framework` says `missing`, this isn't a Laravel app. Stop and use a guide for your framework.
- The `php` constraint tells you which `php:8.x-apache` tag to use in step 7. A `package-lock.json` means the app builds frontend assets, so add an `npm run build` stage.
- If a `Dockerfile` already exists, compare it with step 7 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "laravel-app", "region": "europe-west" }'
export PROJECT_ID=laravel-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

The jobs in step 8 use the same repository; `northflank get job repository-access` takes the same body.

## Step 5: Create the MySQL addon and wait for it

```bash
curl -s https://api.northflank.com/v1/addon-types | jq '.data.addonTypes[] | select(.type=="mysql") | {majors, versions}'
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "laravel-db", "type": "mysql", "version": "8-latest",
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "laravel" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId laravel-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

`customCredentials.dbName` can't be changed after creation. TLS and high availability are also fixed at creation, according to the MySQL docs. Storage can only be increased. Replicas can be reduced for standard MySQL but not for MySQL HA.

## Step 6: Create the shared secret group

First, confirm the credential key names. Print the names only, never the values:

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId laravel-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

The MySQL docs list `HOST`, `PORT`, `USERNAME`, `PASSWORD` and `DATABASE`, and a docs example links `HOST`, `PORT` and `DATABASE` by those exact names. If the output uses different casing, change the `keyName` values below to match.

Generate an app key without echoing it, and build the group with `jq` so the key never appears in your command line or output:

```bash
APP_KEY=$(php artisan key:generate --show)
jq -n --arg key "$APP_KEY" '{
  name: "laravel-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "laravel-db", keys: [
    { keyName: "HOST",     aliases: ["DB_HOST"] },
    { keyName: "PORT",     aliases: ["DB_PORT"] },
    { keyName: "DATABASE", aliases: ["DB_DATABASE"] },
    { keyName: "USERNAME", aliases: ["DB_USERNAME"] },
    { keyName: "PASSWORD", aliases: ["DB_PASSWORD"] }
  ] } ],
  secrets: { variables: {
    APP_KEY: $key, APP_ENV: "production", APP_DEBUG: "false",
    DB_CONNECTION: "mysql", LOG_CHANNEL: "stderr",
    QUEUE_CONNECTION: "database", SESSION_DRIVER: "database", CACHE_STORE: "database"
  } }
}' > laravel-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file laravel-env.json >/dev/null && rm laravel-env.json
unset APP_KEY
```

The group is unrestricted, so the web service, the worker and both jobs inherit it. After any later change, restart `web` and `queue` with `northflank restart service`.

## Step 7: Deploy the web and queue services

If the repository has no `Dockerfile`, add this one. It's the standard official-image setup:

```dockerfile
FROM composer:2 AS vendor
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --prefer-dist --no-interaction --optimize-autoloader

FROM php:8.3-apache
RUN docker-php-ext-install pdo_mysql && a2enmod rewrite
ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf \
 && sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf
WORKDIR /var/www/html
COPY . .
COPY --from=vendor /app/vendor ./vendor
RUN php artisan package:discover --ansi && chown -R www-data:www-data storage bootstrap/cache
EXPOSE 80
```

The image's default command starts Apache on port 80. Build your frontend assets (`npm run build`) in another stage if the app uses Vite.

Northflank builds what's on the remote branch, not your local working tree. Commit and push the `Dockerfile` and the proxy change from the prerequisites before creating the services:

```bash
git add Dockerfile bootstrap/app.php
git commit -m "Add Northflank deployment files"
git push
```

Write `web.json`:

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 80, "public": true, "protocol": "HTTP" } ],
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/up", "port": 80,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

`/up` is Laravel 11's built-in health route. On older versions, point `path` at any route that returns 200. Then write `queue.json`, which is the same build with no ports and a custom command:

```json
{
  "name": "queue",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "customCommand", "customCommand": "php artisan queue:work --tries=3 --max-time=3600" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } }
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file web.json
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file queue.json
```

Use the same repository URL, provider and branch as in step 4 in all four files. `--max-time=3600` makes the worker exit hourly, and the platform restarts it, so memory leaks and stale code don't build up. For more on worker services, see [how to deploy a background worker](deploy-celery-in-production.md).

## Step 8: Create the migrate and scheduler jobs

Both jobs build the same repo. Write `migrate.json`:

```json
{
  "name": "migrate",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "settings": { "backoffLimit": 0, "activeDeadlineSeconds": 900, "runOnSourceChange": "never" },
  "deployment": {
    "vcs": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
    "docker": { "configType": "customCommand", "customCommand": "php artisan migrate --force" }
  },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } }
}
```

Write `scheduler.json`:

```json
{
  "name": "scheduler",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "settings": {
    "backoffLimit": 0, "activeDeadlineSeconds": 300,
    "cron": { "schedule": "* * * * *", "timeZone": "UTC", "concurrencyPolicy": "forbid" }
  },
  "deployment": {
    "vcs": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
    "docker": { "configType": "customCommand", "customCommand": "php artisan schedule:run" }
  },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } }
}
```

```bash
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file migrate.json
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file scheduler.json
```

`put job` makes a cron job when `settings.cron` is present and a manual job otherwise; `put job manual` and `put job cron` are deprecated in the API. `concurrencyPolicy: forbid` stops a slow minute from overlapping the next. That's the platform version of Laravel's `withoutOverlapping()`. The schedule uses standard five-field cron syntax. For more on cron jobs, see [run an image once or on a schedule](https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule).

## Step 9: Wait for builds, then migrate

Wait for `web` and `queue` to reach build `SUCCESS` and deployment `COMPLETED`:

```bash
for SVC in web queue; do
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

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs of the service it stopped on: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web --lineLimit 200` (or `--serviceId queue`).

For each job, wait until its latest build is `SUCCESS`. If a job shows no build after a minute, start one:

```bash
for JOB in migrate scheduler; do
  northflank get job builds --projectId "$PROJECT_ID" --jobId "$JOB" --noDefaults -o json \
    | jq -r --arg j "$JOB" '"\($j): " + ((.data // .).builds | sort_by(.createdAt) | last | .status // "NONE")'
done
# if NONE: northflank start job build --projectId "$PROJECT_ID" --jobId <JOB> --noDefaults -o json --input '{}'
```

Then run the migration:

```bash
RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId migrate --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 90); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId migrate --runId "$RUN_ID" --noDefaults -o json)
  S=$(echo "$R" | jq -r '(.data // .).status'); echo "run=$S"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$S" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId migrate --runId "$RUN_ID" --lineLimit 200
[ "$OK" = 1 ] || { echo "MIGRATION_NOT_COMPLETE: stop here and read the logs above"; false; }
```

Until this succeeds, the scheduler and queue will log missing-table errors. That's expected.

## How to know it worked

`exec` needs `script` to keep output without a TTY (Linux: `script -qefc '<cmd>' /dev/null`). You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The `migrate` run's `status` is `SUCCESS`.
3. The web app answers, using the `HOST` from step 9:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/up"   # 200
   ```
4. The scheduler runs every minute and succeeds. After 3 minutes, the newest runs show `SUCCESS`:
   ```bash
   northflank get job runs --projectId "$PROJECT_ID" --jobId scheduler --per_page 5 --noDefaults -o json \
     | jq -r '(.data // .).runs[] | "\(.startedAt) \(.status)"'
   ```
5. The queue processes a job. From the web container, dispatch a queued closure that logs a fixed marker (`md5(42)` avoids nested string quotes), then look for the marker in the **queue** service's logs:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId web \
     --shell-cmd 'sh -c' --cmd "\"php artisan tinker --execute='dispatch(function () { logger(md5(42)); });'\""
   sleep 15
   northflank get service logs --projectId "$PROJECT_ID" --serviceId queue --lineLimit 200 \
     --textIncludes "a1d0c6e83f027327d8461063f4ac58a6"
   ```
   A matching line means the worker picked the job up. If your app disallows queued closures, dispatch an existing job class instead and look for its `DONE` line in the `queue` logs.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` (and to `deployment.vcs` in both job files) as `"vcsLinkId": "<ID>"` |
| `No application encryption key has been specified` | `APP_KEY` missing from the group | Re-create `laravel-env` with the key; restart services |
| `SQLSTATE[HY000] [2002] Connection refused` on `127.0.0.1` | `DB_HOST` alias not injected | Check the key names from step 6; restart |
| `SQLSTATE[HY000] [1045] Access denied` | Linked admin vs app user mismatch, or wrong key name | Re-check key names; never print the password |
| Mixed-content or `http://` links | Proxies not trusted | Add `trustProxies(at: '*')` |
| Scheduler runs `FAILED` every minute | Missing tables, or the command exits non-zero | `northflank get job logs --jobId scheduler --runId <id>` |
| `storage/logs` permission denied | Ownership | Keep the `chown` line and `LOG_CHANNEL=stderr` |
| Migration hangs and asks for confirmation | Missing `--force` in production | Use `php artisan migrate --force` |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the services or jobs until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print `APP_KEY`, `DB_PASSWORD` or the credentials response. Print key names only.
- Never rotate `APP_KEY` on a running app without asking. It invalidates sessions and anything encrypted with it.
- Back up MySQL before running migrations against data (see [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data)).
- Get explicit confirmation before deleting the addon, services or jobs. To stop the scheduler temporarily, pause the job instead.

## FAQ

### Where should I deploy my Laravel project?
On a host that can run the web server, a long-running queue worker and a per-minute scheduler, with MySQL alongside. A container platform with managed databases and cron jobs covers all of that from one `Dockerfile`. If your app has no queued jobs or scheduled tasks, simpler PHP hosting can be enough.

### Can I use `schedule:work` instead of a cron job?
Yes. Deploy a third service like `queue` with `customCommand: "php artisan schedule:work"`. It's a long-running process, so you lose the per-minute run history, but you skip per-minute container starts.

### Should I run `php artisan config:cache` in the image?
Not at build time. Secrets aren't injected during the build, so the cache would hold empty values. If you want it, run it at container start with a `sh -c` entrypoint override.

### Do I need Redis?
No. The database queue, session and cache drivers work with MySQL alone. Add Redis when queue throughput or cache latency needs it.

## Conclusion

You now have Laravel on Northflank as three workloads from one repo: a web service, a queue worker and a per-minute scheduler. They share MySQL and `APP_KEY` through one secret group, and migrations run as a tracked job.

**Next:** add your domain with [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank), and automate migrations with [run migrations](https://northflank.com/docs/v1/application/release/run-migrations). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a MySQL database](deploy-mysql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy a Rails app](deploy-rails-app.md)
