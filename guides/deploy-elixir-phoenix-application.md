<!--
agent-guide-meta:
  id: NF-STK-PHOENIX
  description: "How to deploy an Elixir Phoenix application as a Mix release with managed PostgreSQL, PHX_HOST set to the public URL and Ecto migrations run as a one-off job."
  sources:
    - https://northflank.com/stacks/deploy-phoenix-elixir
    - https://phoenix.hexdocs.pm/releases.html
    - https://phoenix.hexdocs.pm/deployment.html
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/services/restart-service
    - https://northflank.com/docs/v1/api/project/jobs/put-job
    - https://northflank.com/docs/v1/api/project/jobs/run-job
    - https://northflank.com/docs/v1/api/project/jobs/get-run-details
    - https://northflank.com/docs/v1/application/release/run-migrations
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/docs/v1/application/domains/add-a-domain-to-your-account
    - https://northflank.com/docs/v1/api/use-the-cli
  related: [NF-DB-001, NF-DEP-001, NF-JOB-002, NF-DB-003]
-->

# How to deploy an Elixir Phoenix application

To deploy an Elixir Phoenix application, build a Mix release in a multi-stage `Dockerfile`, give it `DATABASE_URL`, `SECRET_KEY_BASE` and `PHX_HOST` at runtime, connect a managed PostgreSQL database, and run Ecto migrations from the release with `bin/migrate` as a separate step. The final image has no `mix`, and a `PHX_HOST` that doesn't match the public hostname makes LiveView reconnect forever.

This guide does it on Northflank with the CLI: the release built from Git on port 4000, Postgres linked as `DATABASE_URL` through a secret group, `PHX_HOST` set from the assigned hostname, and `bin/migrate` as a one-off job with a clear pass or fail. You end with a public HTTPS URL, verified with `curl` and the logs. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to deploy an Elixir Phoenix application

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a Phoenix app, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Run `mix phx.gen.release --docker` and push the generated `Dockerfile`, `bin/server`, `bin/migrate` and `release.ex`.
4. Create Postgres with `northflank put addon` and one secret group with `POSTGRES_URI` as `DATABASE_URL` plus a generated `SECRET_KEY_BASE`.
5. Deploy with `northflank put service combined` on port 4000, set `PHX_HOST`, then run `/app/bin/migrate` as a manual job.
6. Check that the live build is your pushed commit, the migrate run reports `SUCCESS` and the home page returns 200.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy Phoenix as a release?

A Mix release bundles your compiled app and the Erlang runtime into one directory, so the final image needs neither Elixir nor Mix. That keeps the image small, but it also means `mix ecto.migrate` isn't available in production. Phoenix solves this with a generated `MyApp.Release.migrate/0` function and a `bin/migrate` script that calls it ([Phoenix releases guide](https://phoenix.hexdocs.pm/releases.html)).

On Northflank, the release image runs as a **combined service** (build from Git plus deployment), the database is a PostgreSQL **addon**, and `bin/migrate` runs as a **manual job** built from the same repository. If you'd rather click through the UI, there's a one-click [Phoenix Elixir stack template](https://northflank.com/stacks/deploy-phoenix-elixir).

That shape answers where to deploy a Phoenix app: the BEAM runs as one long-lived process holding open WebSocket connections for channels and LiveView, next to a Postgres database, with a separate migration step. Serverless function hosts end each request and don't fit that model. A raw VM runs the release directly, with the OS, TLS and database left to you. A container platform runs the release image with WebSocket-capable HTTPS and a managed database.

## Prerequisites

- **A Phoenix app that compiles locally:**
  ```bash
  mix deps.get && mix compile && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`** for generating the secret key.

Names used below: project `phoenix-app`, addon `phoenix-db`, secret group `phoenix-env`, service `web`, job `web-migrate`. This guide uses the OTP app name `my_app` and module `MyApp`; use yours.

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
grep -E 'app: :|\{:phoenix,|\{:ecto_sql,|\{:postgrex,' mix.exs || echo "mix.exs: missing"
ls mix.lock Dockerfile rel/overlays/bin/server lib/*/release.ex 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId phoenix-app -o json >/dev/null 2>&1 \
  && echo "project: phoenix-app exists and will be reused" \
  || echo "project: phoenix-app will be created"
```

- If `{:phoenix,` doesn't print, this isn't a Phoenix app. Stop and use a guide for your framework.
- The `app:` line is your OTP app name; replace `my_app` and `MyApp` below with it. `{:postgrex,` means the app uses Postgres, as this guide assumes.
- If `rel/overlays/bin/server` and `release.ex` already list, the release files exist; step 5 only needs to confirm them.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "phoenix-app", "region": "europe-west" }'
export PROJECT_ID=phoenix-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

The job in step 10 uses the same repository; `northflank get job repository-access` takes the same body.

## Step 5: Generate the release files and Dockerfile

From the project root, run the official generator:

```bash
mix phx.gen.release --docker
```

It creates `rel/overlays/bin/server`, `rel/overlays/bin/migrate`, `lib/my_app/release.ex`, a `Dockerfile` and a `.dockerignore`. Commit them all, and prefer the generated `Dockerfile`, since its Elixir, Erlang and Debian versions match. Without Mix locally, use this equivalent (base tags from the current Phoenix docs example, checked on Docker Hub; match them to your versions at [bob.hex.pm](https://bob.hex.pm/docker?repo=hexpm/elixir&os=debian)):

```dockerfile
FROM hexpm/elixir:1.20.2-erlang-29.0.3-debian-trixie-20260623-slim AS build
RUN apt-get update -y && apt-get install -y --no-install-recommends build-essential git && rm -rf /var/lib/apt/lists/*
WORKDIR /app
RUN mix local.hex --force && mix local.rebar --force
ENV MIX_ENV=prod
COPY mix.exs mix.lock ./
RUN mix deps.get --only $MIX_ENV
RUN mkdir config
COPY config/config.exs config/${MIX_ENV}.exs config/
RUN mix deps.compile
RUN mix assets.setup
COPY priv priv
COPY lib lib
RUN mix compile
COPY assets assets
RUN mix assets.deploy
COPY config/runtime.exs config/
COPY rel rel
RUN mix release

FROM debian:trixie-20260623-slim
RUN apt-get update -y && apt-get install -y --no-install-recommends libstdc++6 openssl libncurses6 locales ca-certificates \
  && rm -rf /var/lib/apt/lists/* && sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen
ENV LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 MIX_ENV=prod
WORKDIR /app
RUN chown nobody /app
COPY --from=build --chown=nobody:root /app/_build/prod/rel/my_app ./
USER nobody
CMD ["/app/bin/server"]
```

The generated `config/runtime.exs` reads `DATABASE_URL` and `SECRET_KEY_BASE` (both required), `PHX_HOST` (URL generation and the websocket origin check, default `example.com`), `PORT` (default 4000) and optional `POOL_SIZE`. The endpoint already binds to all interfaces. `runtime.exs` isn't evaluated at build time, so the build needs no secrets.

Northflank builds what's on the remote branch, not your local working tree, so commit and push these files before creating the service:

```bash
git add Dockerfile .dockerignore rel lib/my_app/release.ex
git commit -m "Add Northflank deployment files"
git push
```

## Step 6: Create the PostgreSQL database

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "phoenix-db",
  "type": "postgresql",
  "version": "17-latest",
  "tlsEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId phoenix-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

`tlsEnabled: false` (the API default) matches the generated `runtime.exs`, where `ssl: true` is commented out; the database stays private either way. Storage is in MB and can only be increased.

## Step 7: Create the secret group

Generate the key locally and build the JSON with `jq`, so the key never appears on the command line or in logs. `mix phx.gen.secret` works too.

```bash
SECRET_KEY_BASE=$(openssl rand -base64 64 | tr -d '\n')
jq -n --arg skb "$SECRET_KEY_BASE" '{
  name: "phoenix-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "phoenix-db", keys: [ { keyName: "POSTGRES_URI", aliases: ["DATABASE_URL"] } ] } ],
  secrets: { variables: { SECRET_KEY_BASE: $skb } }
}' > phoenix-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file phoenix-env.json >/dev/null && rm phoenix-env.json
unset SECRET_KEY_BASE
```

Ecto accepts `POSTGRES_URI` (`postgresql://user:password@host:port/database`) as `DATABASE_URL`; the `ecto://` scheme in Phoenix's docs is a convention. The group is unrestricted, so the web service and the migration job can both read it at runtime.

## Step 8: Deploy the web service

Phoenix has no health route by default, so this uses a TCP probe on port 4000. Write `web.json`:

```json
{
  "name": "web",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 4000, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "4000", "PHX_SERVER": "true" },
  "healthChecks": [
    { "protocol": "TCP", "type": "readinessProbe", "port": 4000,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file web.json
```

Use the same repository URL, provider and branch as in step 4, here and in `migrate.json`. The first build starts automatically. If none appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json --input '{}'`. Pages that query new tables fail until step 10 runs.

## Step 9: Wait for the rollout and set `PHX_HOST`

`COMPLETED` means rolled out and serving. Poll every 10 seconds to stay under the API limit of 1,000 requests per hour.

```bash
OK=0
for i in $(seq 1 120); do
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

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId web --lineLimit 200`. The troubleshooting table covers the common causes.

Now set `PHX_HOST`. LiveView and channel connections are rejected when the browser's origin doesn't match it:

```bash
northflank patch service combined --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
  --input "$(jq -n --arg h "$HOST" '{ runtimeEnvironment: { PORT: "4000", PHX_SERVER: "true", PHX_HOST: $h } }')"
northflank restart service --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json
```

Send the complete `runtimeEnvironment` map every time, because existing keys may be replaced. When you add a custom domain later, set `PHX_HOST` to it ([add a domain](https://northflank.com/docs/v1/application/domains/add-a-domain-to-your-account)).

## Step 10: Run migrations from the release

Write `migrate.json`. It builds the same repo and replaces the image's `CMD` with the generated `bin/migrate` script, which calls `MyApp.Release.migrate`. `put job` makes a manual job because `settings.cron` is absent (`put job manual` is deprecated in the API):

```json
{
  "name": "web-migrate",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "settings": { "backoffLimit": 0, "activeDeadlineSeconds": 900, "runOnSourceChange": "never" },
  "deployment": {
    "vcs": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
    "docker": { "configType": "customCommand", "customCommand": "/app/bin/migrate" }
  },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } }
}
```

```bash
northflank put job --projectId "$PROJECT_ID" --noDefaults -o json --file migrate.json

OK=0
for i in $(seq 1 120); do
  JB=$(northflank get job builds --projectId "$PROJECT_ID" --jobId web-migrate --noDefaults -o json \
       | jq -r '(.data // .).builds | sort_by(.createdAt) | last | .status // "NONE"')
  echo "job build=$JB"
  case "$JB" in
    SUCCESS) OK=1; break ;;
    FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo JOB_BUILD_FAILED; break ;;
    NONE) [ "$i" = 6 ] && northflank start job build --projectId "$PROJECT_ID" --jobId web-migrate --noDefaults -o json --input '{}' ;;
  esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "JOB_BUILD_NOT_COMPLETE: stop here and read the job build logs"; false; }

RUN_ID=$(northflank start job run --projectId "$PROJECT_ID" --jobId web-migrate --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 90); do
  R=$(northflank get job run --projectId "$PROJECT_ID" --jobId web-migrate --runId "$RUN_ID" --noDefaults -o json)
  S=$(echo "$R" | jq -r '(.data // .).status'); echo "run=$S"
  [ "$(echo "$R" | jq -r '(.data // .).concluded')" = "true" ] && { [ "$S" = "SUCCESS" ] && OK=1; break; }
  sleep 10
done
northflank get job logs --projectId "$PROJECT_ID" --jobId web-migrate --runId "$RUN_ID" --lineLimit 200
[ "$OK" = 1 ] || { echo "MIGRATION_NOT_COMPLETE: stop here and read the logs above"; false; }
```

Run status is `RUNNING`, `SUCCESS` or `FAILED`. The job loads `runtime.exs`, so it reads `DATABASE_URL` and `SECRET_KEY_BASE` from the secret group too. Start a new run after each release that adds migrations ([run migrations](https://northflank.com/docs/v1/application/release/run-migrations)).

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The migrate run concluded with `status == "SUCCESS"`, and its log shows Ecto migration lines (or "Migrations already up").
3. The home page returns 200:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"
   ```
4. The runtime log has no database connection errors:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId web --lineLimit 200 --textIncludes "DBConnection"
   ```
   This prints nothing when the pool is healthy.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` (and to `deployment.vcs` in `migrate.json`) as `"vcsLinkId": "<ID>"` |
| Container exits with `environment variable DATABASE_URL is missing` | Secret group missing or created after the service started | Re-run step 7, then `northflank restart service` |
| `environment variable SECRET_KEY_BASE is missing` | Same, for the key | Re-run step 7 and restart |
| LiveView reconnects forever, log shows `Could not check origin` | `PHX_HOST` doesn't match the URL in the browser | Redo the end of step 9 with the exact `dns` value or your custom domain |
| URL returns 502/503 while `COMPLETED` | App not started as a server, or wrong port | Use `CMD ["/app/bin/server"]`, keep `PORT` and `internalPort` at 4000 |
| Migrate run `FAILED` with an SSL error | Addon has TLS on but the repo has no `ssl` option | Set `ssl` on the repo, or keep `tlsEnabled: false` |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service or the job until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print `SECRET_KEY_BASE` or `DATABASE_URL`; build JSON with `jq --arg`.
- Rotating `SECRET_KEY_BASE` invalidates every session cookie. Ask before changing it.
- Take a backup before migrating a database that holds data, and ask before running rollbacks (`MyApp.Release.rollback/2`).
- Storage can only be increased. Don't delete the addon, service or job without explicit, per-resource confirmation.

## FAQ

### Where should I deploy my Phoenix app?
On a host that runs a long-lived container and supports WebSockets, so LiveView and channels work. A container platform with managed Postgres and one-off jobs fits the release workflow in this guide. A VM works too if you want to manage the server and database yourself.

### Why not run migrations when the server starts?
With more than one instance, every instance would try to migrate on each restart. A job runs once per release and reports success or failure.

### Do I need `PHX_SERVER=true`?
`bin/server` already sets it. It's in `runtimeEnvironment` as a safeguard in case the start command is changed to `bin/my_app start`.

## Conclusion

You now have a Phoenix release built from Git, running on port 4000 behind a public HTTPS URL, with `SECRET_KEY_BASE` and `DATABASE_URL` injected from a secret group and Ecto migrations running as a tracked job.

**Next:** schedule database backups, then add a custom domain and update `PHX_HOST`. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy Redis](deploy-redis.md)
