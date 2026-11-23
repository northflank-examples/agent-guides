<!--
agent-guide-meta:
  id: NF-STK-RUST
  description: "How to deploy a Rust application from Git with a cargo-chef Dockerfile, cached builds, a slim runtime image and managed PostgreSQL, every step scripted."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://northflank.com/stacks/deploy-rust-nickel
    - https://github.com/LukeMathWalker/cargo-chef
    - https://docs.rs/sqlx/latest/sqlx/
    - https://docs.rs/axum/latest/axum/
  related: [NF-DEP-001, NF-DB-001, NF-STK-GO, NF-JOB-002]
-->

# How to deploy a Rust application

To deploy a Rust application such as an Axum or Actix Web service, build it with a multi-stage `Dockerfile` that caches compiled dependencies with cargo-chef, ship the release binary in a slim Debian image, bind `0.0.0.0:$PORT`, and pass a PostgreSQL connection string as `DATABASE_URL` at runtime. Without dependency caching, every push recompiles every crate, and a clean `cargo build --release` can take many minutes.

This guide does it on Northflank with the CLI: a service that builds from Git with BuildKit's layer cache turned on, and an optional managed Postgres database linked in as `DATABASE_URL`. You end with a public HTTPS URL whose `/healthz` endpoint runs `SELECT 1` and returns 200. A human links your Git account once, and the first build can take up to 30 minutes. Every step is a command a developer or a coding agent can run as-is.

## TL;DR: how to deploy a Rust application

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a Cargo project, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Bind `0.0.0.0:$PORT`, serve `GET /healthz`, and push a cargo-chef `Dockerfile` (`planner` → `cargo chef cook` → `cargo build --release` → `debian:trixie-slim`).
4. Optionally create Postgres with `northflank put addon` and link `POSTGRES_URI` as `DATABASE_URL` with `northflank put secret`.
5. Deploy with `northflank put service combined --file rust-api.json`, with `buildkit.useCache: true`, and wait until `COMPLETED`.
6. Check that the live build is your pushed commit and that `curl https://$HOST/healthz` returns 200.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why cache dependencies separately?

Docker reuses a layer only when its inputs are unchanged. If you `COPY . .` before building, every source edit invalidates the layer that compiles your dependencies. cargo-chef reads `Cargo.toml` and `Cargo.lock` and writes a "recipe" that builds only the dependencies. That layer stays cached until the lockfile changes, so a typical push only recompiles your own crate.

On Northflank, one **combined service** builds the `Dockerfile` from Git and runs it behind HTTPS. Setting `buildkit.useCache` keeps build layers on persistent storage between builds. The database is a PostgreSQL **addon**, and a **secret group** injects its connection string. If you'd rather click through a UI, Northflank has a one-click [Rust (Nickel) stack template](https://northflank.com/stacks/deploy-rust-nickel). This guide is the scripted path for any Rust framework.

On where to deploy a Rust web app: it's a long-running server binary that needs a host to keep it up, and a builder with enough CPU and cache for `cargo build --release`. Serverless function hosts can run Rust but expect a per-request handler and give you little control over the build. A raw VM runs the binary directly, with the OS, TLS and deploys left to you. A container platform builds the image from Git, caches layers and serves it over HTTPS.

## Prerequisites

- **A Rust web app that builds locally, with `Cargo.lock` committed** (cargo-chef needs it for reproducible dependency builds):
  ```bash
  cargo build --release --locked && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

Names used below: project `rust-app`, addon `rust-db`, secret group `rust-db-env`, service `api`.

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
grep -E '^(name|axum|actix-web|rocket|warp|sqlx) *=' Cargo.toml || echo "Cargo.toml: missing"
grep -A2 '^\[\[bin\]\]' Cargo.toml
ls Cargo.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId rust-app -o json >/dev/null 2>&1 \
  && echo "project: rust-app exists and will be reused" \
  || echo "project: rust-app will be created"
```

- If `Cargo.toml` is missing, this isn't a Rust project. Stop and use a guide for your framework.
- The web framework line (`axum`, `actix-web`…) tells you which server code applies in step 5. If none prints, the app may not be a web server.
- The `[[bin]]` name, or the package `name`, is the `<BIN>` used in step 6. If `ls` doesn't show `Cargo.lock`, run `cargo generate-lockfile` and commit it.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "rust-app", "region": "europe-west" }'
export PROJECT_ID=rust-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Make the app container-ready

Bind every interface and read `PORT`. Here's Axum 0.8 with sqlx:

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let pool = sqlx::postgres::PgPoolOptions::new()
        .max_connections(5)
        .connect(&std::env::var("DATABASE_URL")?).await?;
    sqlx::migrate!().run(&pool).await?;   // applies ./migrations; remove if you don't use sqlx migrations

    let app = axum::Router::new()
        .route("/healthz", axum::routing::get(healthz))
        .with_state(pool);

    let port = std::env::var("PORT").unwrap_or_else(|_| "8080".into());
    let listener = tokio::net::TcpListener::bind(format!("0.0.0.0:{port}")).await?;
    axum::serve(listener, app).await?;
    Ok(())
}

async fn healthz(axum::extract::State(pool): axum::extract::State<sqlx::PgPool>) -> axum::http::StatusCode {
    match sqlx::query("SELECT 1").execute(&pool).await {
        Ok(_) => axum::http::StatusCode::OK,
        Err(_) => axum::http::StatusCode::SERVICE_UNAVAILABLE,
    }
}
```

For Actix Web, the equivalent is `HttpServer::new(...).bind(("0.0.0.0", port))?`.

In `Cargo.toml`, enable a TLS backend for sqlx, for example `sqlx = { version = "0.8", features = ["runtime-tokio", "tls-rustls", "postgres", "migrate"] }`. rustls means the runtime image doesn't need OpenSSL. `sqlx::migrate!` takes a Postgres advisory lock, so several instances can start at once without applying a migration twice.

If you use sqlx's compile-time checked macros (`query!`), the build has no database. Run `cargo sqlx prepare` locally, commit the `.sqlx` directory, and the build sets `SQLX_OFFLINE=true` (see the `Dockerfile` below).

## Step 6: Write a multi-stage Dockerfile with dependency caching

This follows the cargo-chef README. Replace `<BIN>` in both places:

```dockerfile
FROM lukemathwalker/cargo-chef:latest-rust-1 AS chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
# Compiles only dependencies; cached until Cargo.toml/Cargo.lock change
RUN cargo chef cook --release --recipe-path recipe.json
COPY . .
ENV SQLX_OFFLINE=true
RUN cargo build --release --bin <BIN>

FROM debian:trixie-slim AS runtime
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/* && useradd --uid 10001 --no-create-home app
COPY --from=builder /app/target/release/<BIN> /usr/local/bin/app
USER app
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/app"]
```

Keep the runtime image on the same Debian release as the build image, or the binary may fail with a glibc version error. Add a `.dockerignore` containing `target/` and `.git`, so local build output never enters the build context.

Northflank builds what's on the remote branch, not your local working tree, so commit and push these files (plus any source changes from step 5) before creating the service:

```bash
git add Dockerfile .dockerignore Cargo.lock
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Create the PostgreSQL database (optional)

Skip steps 7 and 8 if the app has no database, and remove the pool code from step 5. Storage is in MB and can only be increased later.

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "rust-db",
  "type": "postgresql",
  "version": "17-latest",
  "tlsEnabled": true,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId rust-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

## Step 8: Link the connection string as `DATABASE_URL`

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "rust-db-env",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": false },
  "addonDependencies": [
    { "addonId": "rust-db", "keys": [ { "keyName": "POSTGRES_URI", "aliases": ["DATABASE_URL"] } ] }
  ],
  "secrets": { "variables": { "RUST_LOG": "info" } }
}'
```

`POSTGRES_URI` has the form `postgresql://user:password@host:port/database`, which `PgPoolOptions::connect` accepts directly. sqlx defaults to `sslmode=prefer`, so it uses TLS when the addon has it enabled. The variable is injected at runtime only, never into the build.

## Step 9: Create the service with build caching on

Write `rust-api.json`:

```json
{
  "name": "api",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": {
    "dockerfile": {
      "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/",
      "buildkit": { "useCache": true }
    }
  },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "8080" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/healthz", "port": 8080,
      "initialDelaySeconds": 3, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/healthz", "port": 8080,
      "initialDelaySeconds": 15, "periodSeconds": 20, "timeoutSeconds": 3, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file rust-api.json
```

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined`. `buildkit.useCache` will "Use persistent storage to cache build layers", per the API schema. The optional `buildkit.cacheStorageSize` sets that storage per build in MB. The first build starts automatically; if none appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json --input '{}'`.

## Step 10: Wait for the build and rollout

The first build compiles every dependency, so allow up to 30 minutes. Poll every 10 seconds.

```bash
OK=0
for i in $(seq 1 180); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

`COMPLETED` means rolled out and serving. On `BUILD_FAILED` or `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId api --lineLimit 200`. The troubleshooting table covers the common causes.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health endpoint, which runs `SELECT 1`, returns 200:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/healthz"   # 200
   ```
3. `DATABASE_URL` is set in the container, checked without printing it. `exec` drops output without a TTY, so wrap it in `script` (Linux: `script -qefc '<cmd>' /dev/null`):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId api \
     --shell-cmd 'sh -c' --cmd '"printenv DATABASE_URL | cut -c1-13"'   # postgresql://
   ```
4. Optional: after a source-only change, the next build log shows the `cargo chef cook` step as cached. It hasn't been confirmed that the build log prints BuildKit's `CACHED` marker, so treat a much shorter build time as the signal if this finds nothing:
   ```bash
   northflank get service build-logs --projectId "$PROJECT_ID" --serviceId api --lineLimit 400 --textIncludes "CACHED"
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| Build killed or `signal: 9` during compile | Build plan out of memory | Add `ENV CARGO_BUILD_JOBS=2` before `cargo build`, or set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| Every build recompiles all crates | `COPY . .` before `cargo chef cook`, or `Cargo.lock` not committed | Keep the stage order above; commit the lockfile |
| `GLIBC_2.xx not found` at start | Runtime Debian older than the build image | Use matching releases (`trixie` with `latest-rust-1`) |
| `set DATABASE_URL to use query macros online` | sqlx macros at build time | Commit `.sqlx` from `cargo sqlx prepare`; keep `SQLX_OFFLINE=true` |
| Health check fails, logs show `127.0.0.1:8080` | Bound to localhost | Bind `0.0.0.0:{PORT}` |
| `/healthz` returns 503 | Addon not `running` or secret group missing | Re-check steps 7–8, then `northflank restart service --projectId "$PROJECT_ID" --serviceId api` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 10 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print `DATABASE_URL` or passwords; truncate as in check 3.
- Don't set or raise `billing.buildPlan` without asking; it affects the bill.
- Addon storage can only be increased, never reduced.
- Don't delete the addon, service or project without explicit, per-resource confirmation. Back up the database before running destructive migrations.

## FAQ

### Where should I deploy my Rust application?
On a host that runs a container long-term and caches build layers between pushes, since Rust builds are slow from scratch. A container platform with build caching and managed databases gives you both. A VM works if you'd rather build the binary yourself and manage the server.

### Can I use a `scratch` or distroless image?
Yes, if you build a fully static binary with the `x86_64-unknown-linux-musl` target. The glibc build shown here needs a Debian runtime, which is still small.

### Does BuildKit's `RUN --mount=type=cache` work too?
That hasn't been confirmed on this platform. cargo-chef relies on ordinary layer caching, which works with `buildkit.useCache`.

### Is `DATABASE_URL` available at build time?
No. It's injected at runtime only, which is why sqlx macros need offline mode.

### How do I run migrations without running them at startup?
Remove `sqlx::migrate!` from `main`, add `sqlx-cli` to the image, and run `sqlx migrate run` through `northflank exec service` wrapped in `script`, as in check 3.

## Conclusion

You now have a Rust web service built from Git with cached dependencies, running in a slim image on `$PORT`, connected to a managed Postgres database through `DATABASE_URL`, with a health check that tests the connection.

**Next:** add Redis for caching or a background worker for queued jobs. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy a Golang web application](deploy-golang-web-application.md)
