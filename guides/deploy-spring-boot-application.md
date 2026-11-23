<!--
agent-guide-meta:
  id: NF-STK-SPRING
  description: "How to deploy a Spring Boot application with Docker and managed PostgreSQL, sizing the JVM to its plan and using Actuator probes, with every step scripted."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/miscellaneous/miscellaneous/list-plans
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/billing/resource-plans-and-quotas
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-postgresql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
    - https://docs.spring.io/spring-boot/reference/features/external-config.html
    - https://docs.spring.io/spring-boot/reference/actuator/endpoints.html
    - https://jdbc.postgresql.org/documentation/use/
  related: [NF-DEP-001, NF-DB-001, NF-STK-DOTNET, NF-JOB-002]
-->

# How to deploy a Spring Boot application

To deploy a Spring Boot application, build the jar in a multi-stage Docker image and run it on a JRE base, give the container at least 1 GB of memory with `-XX:MaxRAMPercentage=75`, connect PostgreSQL through a `jdbc:postgresql://` URL and `SPRING_DATASOURCE_*` variables, and use Actuator's liveness and readiness endpoints as health checks. A too-small memory limit or a plain `postgresql://` URL is what breaks most first deploys.

This guide does it on Northflank with the CLI: a Maven or Gradle `Dockerfile` built from Git, a managed Postgres database whose credentials a secret group maps onto Spring's variable names, and startup, readiness and liveness probes. You end with a public HTTPS URL where `/actuator/health/readiness` reports `UP`, including the database check. A human links your Git account once. Every step is a command a developer or a coding agent can run as-is.

## TL;DR: how to deploy a Spring Boot application

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo is a Spring Boot app, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Set `spring.datasource.url=jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}`, enable the actuator probes, and push a Docker build on `eclipse-temurin:21-jdk` that runs on `eclipse-temurin:21-jre`.
4. Create Postgres with `northflank put addon` and link its `HOST`, `PORT`, `DATABASE`, `USERNAME` and `PASSWORD` keys as `DB_*` and `SPRING_DATASOURCE_*`.
5. Deploy with `northflank put service combined` on a 1 GB plan with `JAVA_TOOL_OPTIONS=-XX:MaxRAMPercentage=75`.
6. Check that the live build is your pushed commit and that `/actuator/health/readiness` returns `{"status":"UP"}`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why size the JVM to the plan?

A modern JVM reads the container's memory limit. By default it caps the heap at 25% of that limit, which wastes most of a small container. If you raise the cap too far, heap plus metaspace, thread stacks and native memory go over the limit and the container is killed with no Java stack trace. `-XX:MaxRAMPercentage=75` is a common middle ground that leaves room for non-heap memory.

On Northflank, the plan sets the container's CPU and memory, and you can list the options without authenticating. The database is a PostgreSQL **addon**. A **secret group** links its credentials into the service under the names Spring already understands. Spring's relaxed binding maps `SPRING_DATASOURCE_USERNAME` to `spring.datasource.username`, so no code change is needed for those.

That memory sizing is also the main factor in where to deploy a Spring Boot application. It's a long-running JVM process that wants a predictable amount of memory and a database next to it. Serverless function hosts can run Spring with extra adapters, but cold starts and per-request execution don't suit a typical Spring web app. A VM or Tomcat server runs the jar directly, with Java, TLS and patching left to you. A container platform runs the Docker image on a fixed plan with a managed database.

## Prerequisites

- **A Spring Boot app that builds locally** (Maven shown; for Gradle, run `./gradlew bootJar`):
  ```bash
  ./mvnw -q package -DskipTests && ls target/*.jar && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The app has these dependencies:** `spring-boot-starter-web`, `spring-boot-starter-actuator`, `spring-boot-starter-data-jpa` (or `-jdbc`) and `org.postgresql:postgresql`. For migrations, add Flyway (on Spring Boot 4, `spring-boot-starter-flyway`; on Spring Boot 3, `org.flywaydb:flyway-core` plus `org.flywaydb:flyway-database-postgresql`).

Names used below: project `spring-app`, addon `spring-db`, secret group `spring-db-env`, service `api`.

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
grep -hoE 'spring-boot-starter[a-z-]*' pom.xml build.gradle build.gradle.kts 2>/dev/null | sort -u
grep -qs 'spring-boot' pom.xml build.gradle build.gradle.kts && echo "spring-boot: found" || echo "spring-boot: missing"
grep -hoE '<java.version>[0-9]+|languageVersion[^)]*[0-9]+|sourceCompatibility.*' pom.xml build.gradle build.gradle.kts 2>/dev/null
ls mvnw gradlew Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId spring-app -o json >/dev/null 2>&1 \
  && echo "project: spring-app exists and will be reused" \
  || echo "project: spring-app will be created"
```

- If it says `spring-boot: missing`, this isn't a Spring Boot app. Stop and use a guide for your framework.
- `mvnw` means Maven and `gradlew` means Gradle; step 6 has a `Dockerfile` for each. Match the image tag (`21`) to the Java version printed.
- If `spring-boot-starter-actuator` isn't listed, add it: the health checks in step 9 use it.
- If a `Dockerfile` already exists, compare it with step 6 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "spring-app", "region": "europe-west" }'
export PROJECT_ID=spring-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Configure the app for the container

Add these lines to `src/main/resources/application.properties`:

```properties
server.port=${PORT:8080}
server.shutdown=graceful

spring.datasource.url=jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}?sslmode=prefer
# spring.datasource.username / password come from SPRING_DATASOURCE_USERNAME / _PASSWORD

management.endpoint.health.probes.enabled=true
management.endpoint.health.group.readiness.include=readinessState,db
```

Spring resolves `${...}` placeholders from environment variables. The probe groups expose `/actuator/health/liveness` and `/actuator/health/readiness`. Including `db` in readiness takes an instance out of rotation when it can't reach Postgres. Liveness stays independent of the database, so a database blip doesn't restart every pod.

Spring Boot binds all interfaces by default, so you don't need to set `server.address`. Flyway runs pending migrations from `src/main/resources/db/migration` on startup, before the app reports ready.

## Step 6: Write a multi-stage Dockerfile

For **Maven**:

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /src
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN chmod +x mvnw && ./mvnw -B -q dependency:go-offline
COPY src ./src
RUN ./mvnw -B -q package -DskipTests && cp target/*.jar /app.jar

FROM eclipse-temurin:21-jre
RUN useradd --uid 10001 --no-create-home spring
COPY --from=build /app.jar /app/app.jar
USER spring
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

For **Gradle**, replace the build stage. If the `jar` task also runs (for example through `build` or `assemble`), Gradle writes a `-plain.jar` next to the boot jar that can't run, so skip it when copying:

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /src
COPY gradlew settings.gradle* build.gradle* ./
COPY gradle/ gradle/
RUN chmod +x gradlew && ./gradlew --no-daemon dependencies > /dev/null
COPY src ./src
RUN ./gradlew --no-daemon bootJar -x test \
 && cp "$(ls build/libs/*.jar | grep -v -- '-plain.jar')" /app.jar
```

Copying the build files before `src` keeps dependency downloads in a cached layer. Match the image tag (`21`) to the project's Java version. Add a `.dockerignore` with `target/`, `build/`, `.gradle/` and `.git`.

Northflank builds what's on the remote branch, not your local working tree, so commit and push these files before creating the service:

```bash
git add Dockerfile .dockerignore src/main/resources/application.properties
git commit -m "Add Northflank deployment files"
git push
```

## Step 7: Create the PostgreSQL database

Storage is in MB and can only be increased later, so start small.

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "spring-db",
  "type": "postgresql",
  "version": "17-latest",
  "tlsEnabled": true,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId spring-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

## Step 8: Map the credentials onto `SPRING_DATASOURCE_*`

The secret group links individual connection keys and gives each one the name Spring or your `application.properties` expects. It also sets the JVM memory flag, so every workload that uses the group gets it:

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "spring-db-env",
  "type": "secret",
  "secretType": "environment",
  "priority": 10,
  "restrictions": { "restricted": false },
  "addonDependencies": [
    { "addonId": "spring-db", "keys": [
      { "keyName": "HOST",     "aliases": ["DB_HOST"] },
      { "keyName": "PORT",     "aliases": ["DB_PORT"] },
      { "keyName": "DATABASE", "aliases": ["DB_NAME"] },
      { "keyName": "USERNAME", "aliases": ["SPRING_DATASOURCE_USERNAME"] },
      { "keyName": "PASSWORD", "aliases": ["SPRING_DATASOURCE_PASSWORD"] }
    ] }
  ],
  "secrets": { "variables": { "JAVA_TOOL_OPTIONS": "-XX:MaxRAMPercentage=75" } }
}'
```

The addon also provides a `JDBC_POSTGRES_URI`. Its exact format isn't documented, so this guide builds the URL from its parts instead. To inspect what the addon offers, print key names only:

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId spring-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

## Step 9: Create the service

Spring Boot can take a while to start on a fraction of a CPU. A **startup probe** gives it about three minutes (18 checks, 10 seconds apart) before liveness checks begin. The API requires `periodSeconds` of at least 10 and `initialDelaySeconds` of at least 1. The plan matters too: `nf-compute-20` (512 MB) is usually too small for Spring Boot with JPA, so start at `nf-compute-50` (0.5 vCPU, 1024 MB) and use `nf-compute-100-2` (1 vCPU, 2048 MB) for production traffic or faster startup. Write `spring-api.json`:

```json
{
  "name": "api",
  "billing": { "deploymentPlan": "nf-compute-50" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "8080" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "startupProbe", "path": "/actuator/health/liveness", "port": 8080,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 18 },
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/actuator/health/readiness", "port": 8080,
      "initialDelaySeconds": 1, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/actuator/health/liveness", "port": 8080,
      "initialDelaySeconds": 1, "periodSeconds": 20, "timeoutSeconds": 3, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file spring-api.json
```

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined`. The first build starts automatically; if none appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json --input '{}'`. When you change plan later, `MaxRAMPercentage` scales the heap with it.

## Step 10: Wait for the build and rollout

```bash
OK=0
for i in $(seq 1 120); do
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

`COMPLETED` means rolled out and serving. Poll every 10 seconds to stay under the API limit of 1,000 requests per hour. If the loop prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId api --lineLimit 200`. The troubleshooting table covers the common causes.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId api --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. Readiness, which includes the database check, reports `UP`. Actuator exposes the `health` endpoint over HTTP by default:
   ```bash
   curl -s "https://$HOST/actuator/health/readiness"   # {"status":"UP"}
   ```
3. The JVM picked up the memory flag and the app started (Flyway also logs applied migrations here):
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId api --lineLimit 300 --textIncludes "JAVA_TOOL_OPTIONS"
   northflank get service logs --projectId "$PROJECT_ID" --serviceId api --lineLimit 300 --textIncludes "Started"
   ```
   The JVM prints `Picked up JAVA_TOOL_OPTIONS: -XX:MaxRAMPercentage=75` and Spring prints `Started <App> in N seconds`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| Container restarts with no Java stack trace | Killed for exceeding the memory limit | Lower `MaxRAMPercentage` to 65, or use `nf-compute-100-2` |
| `java.lang.OutOfMemoryError: Java heap space` | Heap too small for the workload | Raise the plan; keep the percentage |
| Pod restarts before `Started` appears | Startup probe gives up on a slow CPU | Raise `failureThreshold` on the startup probe, or the plan's CPU |
| `Failed to configure a DataSource: 'url' attribute is not specified` | `spring.datasource.url` missing, or `DB_*` not injected | Check step 5 and the secret group, then `northflank restart service --projectId "$PROJECT_ID" --serviceId api` |
| `Driver org.postgresql.Driver claims to not accept jdbcUrl, postgresql://…` | `POSTGRES_URI` used as the JDBC URL | Use the `jdbc:postgresql://` form from step 5 |
| `/actuator/health/readiness` returns 404 | Actuator missing or probes not enabled | Add the actuator starter and `management.endpoint.health.probes.enabled=true` |
| Gradle build copies two jars | `-plain.jar` matched | Use the `grep -v` copy from step 6 |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 10 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print `SPRING_DATASOURCE_PASSWORD` or full credentials. Print key names only.
- Plan size affects the bill. Agree on the plan with the user before moving above `nf-compute-100-2`.
- Back up the database before deploying a release with destructive Flyway migrations.
- Addon storage can only be increased. Don't delete the addon, service or project without explicit, per-resource confirmation.

## FAQ

### Where should I deploy my Spring Boot application?
On a host that runs a long-lived container with a fixed memory limit you choose, and a Postgres database nearby. A container platform gives you that from your `Dockerfile` with HTTPS included. A VM works too if you want to manage the JVM, OS and database yourself.

### Can I set `SPRING_DATASOURCE_URL` as an environment variable instead?
Yes, if you have a full `jdbc:postgresql://host:port/db` value. Spring maps it to `spring.datasource.url`, and an environment variable overrides `application.properties`. This guide composes the URL in properties because the addon gives host, port and database as separate keys.

### Should I use Spring Boot's buildpack image (`bootBuildImage`) instead?
That works locally, but this guide builds from a `Dockerfile` so the build is the same on every machine and in the platform's builder.

### Why is liveness separate from the database?
If liveness included the database, a short database outage would restart every instance at once. Readiness takes instances out of rotation instead, and they come back when the database does.

## Conclusion

You now have a Spring Boot app built from Git with Maven or Gradle, running on a JRE image sized to its plan, connected to managed PostgreSQL through `SPRING_DATASOURCE_*`, with Actuator probes that tell the platform when it's alive and ready.

**Next:** add Redis for caching or sessions, or a background worker for scheduled and queued work. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
- [How to deploy an ASP.NET Core web application](deploy-aspnet-core-web-application.md)
