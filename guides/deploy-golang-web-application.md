<!--
agent-guide-meta:
  id: NF-STK-GO
  description: "How to deploy a Golang web application as a static binary in a distroless Docker image, with optional managed PostgreSQL and a public HTTPS URL via CLI or API."
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
    - https://github.com/GoogleContainerTools/distroless
    - https://github.com/jackc/pgx
    - https://pkg.go.dev/net/http
  related: [NF-DEP-001, NF-DB-001, NF-STK-RUST, NF-JOB-002]
-->

# How to deploy a Golang web application

To deploy a Golang web application, compile a static binary with `CGO_ENABLED=0` in a multi-stage `Dockerfile`, ship it in a distroless image, listen on `0.0.0.0:$PORT` with a `/healthz` route, and pass an optional PostgreSQL connection string in as `DATABASE_URL`. The same rules apply to `net/http`, Gin, Echo and Chi. A server bound to `localhost` is the usual reason a Go deploy fails its health check.

This guide does it on Northflank with the CLI, with the matching REST calls alongside: one service that builds from Git on every push, and an optional managed Postgres linked as `DATABASE_URL`. You end with a public HTTPS URL where `/healthz` returns 200, which also proves the database connection. Every step is a copy-pasteable command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end.

## TL;DR: how to deploy a Golang web application

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a Go module, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Listen on `0.0.0.0:$PORT`, serve `GET /healthz`, and push a two-stage `Dockerfile` (`golang:1.27` with `CGO_ENABLED=0`, then distroless `static`).
4. Optionally create Postgres with `northflank put addon` and link `POSTGRES_URI` as `DATABASE_URL` with `northflank put secret`.
5. Deploy with `northflank put service combined --file go-api.json` and wait until `COMPLETED`.
6. Check that the live build is your pushed commit and that `curl https://$HOST/healthz` returns 200.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy Go as a static binary in a container?

Go compiles to a single binary. With `CGO_ENABLED=0` that binary has no libc dependency, so the runtime image needs nothing except the binary and CA certificates. A distroless `static` image gives you exactly that, runs as a non-root user and has no shell for an attacker to use.

On Northflank, one **combined service** builds the `Dockerfile` from your Git branch and runs the result behind HTTPS. The optional database is a PostgreSQL **addon**. A **secret group** injects its connection string as `DATABASE_URL`, so the password never passes through your terminal.

As for where to deploy a Go app: a Go web service is a long-running HTTP server, so it needs a host that keeps a process up, plus a database if you have one. Serverless function hosts can run Go but expect a handler per request rather than your own server. A raw VM or Kubernetes cluster runs the binary as-is, with the OS, TLS and rollouts left to you. A container platform runs the same image with HTTPS and a managed database included.

## Prerequisites

- **A Go web app that builds locally:**
  ```bash
  go build ./... && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

Names used below: project `go-app`, addon `go-db`, secret group `go-db-env`, service `api`. IDs are slugified names.

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
grep -E '^(module|go) ' go.mod || echo "go.mod: missing"
grep -rl --include='*.go' '^package main' . | head -n5
ls go.sum Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId go-app -o json >/dev/null 2>&1 \
  && echo "project: go-app exists and will be reused" \
  || echo "project: go-app will be created"
```

- If `go.mod` is missing, this isn't a Go module. Stop and use a guide for your framework.
- The `package main` files tell you the main package path (`.` or `./cmd/server`); step 6 builds `./cmd/server`. The `go` line tells you which `golang` image tag to use.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "go-app", "region": "europe-west" }'
export PROJECT_ID=go-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

REST: `POST https://api.northflank.com/v1/projects` and `POST https://api.northflank.com/v1/projects/{projectId}/services/repo-access`.

## Step 5: Make the app container-ready

The server must bind every interface, not `localhost`, and read the port from `PORT`. Here's a `net/http` example using pgx; the same rules apply to Gin (`r.Run(":" + port)`) and Echo (`e.Start(":" + port)`):

```go
port := os.Getenv("PORT")
if port == "" { port = "8080" }

var pool *pgxpool.Pool
if dsn := os.Getenv("DATABASE_URL"); dsn != "" {
    p, err := pgxpool.New(context.Background(), dsn)
    if err != nil { log.Fatal(err) }
    pool = p
}

mux := http.NewServeMux()
mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
    if pool != nil {
        ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
        defer cancel()
        if err := pool.Ping(ctx); err != nil { http.Error(w, "db down", http.StatusServiceUnavailable); return }
    }
    w.Write([]byte("ok"))
})
log.Fatal(http.ListenAndServe(":"+port, mux)) // ":8080" listens on all interfaces
```

`pgxpool.New` accepts the `postgresql://` URL the database provides, so you don't need to parse it. If you use `lib/pq` instead, the same URL works with `sql.Open("postgres", dsn)`.

If you use golang-migrate, run it from `main` before `ListenAndServe`. Its Postgres driver takes an advisory lock, so more than one instance can start at once.

## Step 6: Write a multi-stage Dockerfile

Put this at the repo root, and adjust `./cmd/server` to your main package:

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.27 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /out/server ./cmd/server

FROM gcr.io/distroless/static-debian13:nonroot
COPY --from=build /out/server /server
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

Copying `go.mod` and `go.sum` before the source keeps `go mod download` in a cached layer until your dependencies change. Add a `.dockerignore` with `.git` and any local `.env` files. If the code needs cgo (for example `mattn/go-sqlite3`), drop `CGO_ENABLED=0` and use `gcr.io/distroless/base-debian13:nonroot`, which includes glibc and matches the Debian 13 (trixie) base of the `golang:1.27` image. Match the `golang` tag to the `go` line in `go.mod`; Go supports only the two most recent major releases (1.27 and 1.26).

Northflank builds what's on the remote branch, not your local working tree, so commit and push both files (plus any source changes from step 5) before creating the service:

```bash
git add Dockerfile .dockerignore
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Create the PostgreSQL database (optional)

Skip steps 7 and 8 if the app has no database. Storage is in MB and can only be increased later, so start small.

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "go-db",
  "type": "postgresql",
  "version": "17-latest",
  "tlsEnabled": true,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId go-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

Stop and report to the user if it ends in a failure state. Don't delete and recreate without asking.

## Step 8: Link the connection string as `DATABASE_URL`

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "go-db-env",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": false },
  "addonDependencies": [
    { "addonId": "go-db", "keys": [ { "keyName": "POSTGRES_URI", "aliases": ["DATABASE_URL"] } ] }
  ]
}'
```

`POSTGRES_URI` has the form `postgresql://user:password@host:port/database`. `secretType: "environment"` injects it at runtime only, not into the build. The group is unrestricted, so every workload in the project can read it.

With `tlsEnabled: true`, pgx's default (`sslmode=prefer`) and lib/pq's default (`sslmode=require`) both negotiate TLS. Whether the certificate passes `sslmode=verify-full` hasn't been confirmed, so don't add that without testing.

## Step 9: Create the service

Write `go-api.json`. Port names are at most 8 characters and start with a letter.

```json
{
  "name": "api",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
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
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file go-api.json
```

Use the same repository URL, provider and branch as in step 4. If the app has no database, the service still starts: `/healthz` skips the ping when `DATABASE_URL` is unset. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined`. The first build starts automatically. If none appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json --input '{}'`.

If the liveness probe pings the database, a database outage restarts your pods. If you'd rather not, point the liveness probe at a path that doesn't touch the database.

## Step 10: Wait for the build and rollout

`COMPLETED` means rolled out and serving. Poll every 10 seconds to stay under the API limit of 1,000 requests per hour.

```bash
OK=0
for i in $(seq 1 90); do
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

On `BUILD_FAILED` or `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId api --lineLimit 200`. The troubleshooting table covers the common causes.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health endpoint returns 200. With a database, that also proves the connection works:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/healthz"   # 200
   ```
3. The runtime log has no connection errors:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId api --lineLimit 200 --textIncludes "error"
   ```

The distroless image has no shell, so `northflank exec service` won't work in it. Debug from logs and the health endpoint instead.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| Health check fails, logs show `listening on 127.0.0.1` | Server bound to `localhost` | Listen on `":" + port` or `0.0.0.0` |
| `exec format error` or `no such file or directory` at start | Binary built with cgo or for the wrong OS | Set `CGO_ENABLED=0 GOOS=linux`, or switch to `distroless/base` |
| `x509: certificate signed by unknown authority` on outbound calls | Runtime image without CA certificates | Keep a distroless image (it includes them), don't use `scratch` without copying certs |
| `/healthz` returns 503 | Wrong or missing `DATABASE_URL`, or the addon isn't `running` | Re-check steps 7 and 8; restart with `northflank restart service --projectId "$PROJECT_ID" --serviceId api` |
| `SSL is not enabled on the server` | lib/pq defaults to `require` and the addon has TLS off | Enable TLS on the addon, or add `sslmode=disable` only for internal traffic |
| Build `FAILURE` at `go mod download` | Private modules | Pass a token as a build secret and set `GOPRIVATE` |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 10 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print `DATABASE_URL` or passwords. Check that a variable is set, not what it contains.
- Addon storage can only be increased, never reduced. Don't oversize "just in case".
- Don't delete the addon, service or project without explicit, per-resource confirmation. Prefer pausing, and back up the database before risky schema changes.
- Don't change the project's region; it's fixed at creation.

## FAQ

### Where should I deploy my Go application?
Anywhere that runs a container and keeps it running, because the whole app is one small image. A container platform gives you builds from Git, HTTPS and a managed Postgres database without a server to maintain. A VM or Kubernetes cluster works too if you want to manage those parts yourself.

### Does this work for Gin, Echo, Chi and Fiber?
Yes. The platform only needs the process to listen on `0.0.0.0:$PORT` and answer the health path. Use the framework's own start call with `":" + port`.

### Why distroless instead of `scratch` or Alpine?
`scratch` has no CA certificates, timezone data or non-root user unless you copy them in. Distroless `static` includes those and is still a few megabytes. Alpine works too, but it adds a shell and a package manager that a static Go binary doesn't need.

### Is `DATABASE_URL` available at build time?
No. `secretType: "environment"` injects it at runtime only. Don't connect to the database during `go build`.

## Conclusion

You now have a Go service built from Git into a small distroless image, listening on `$PORT`, with an optional managed Postgres database injected as `DATABASE_URL` and a health check that tests the connection.

**Next:** add a background worker or a cache, or compare the Rust setup if you're choosing between the two. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy a Rust application](deploy-rust-application.md)
