<!--
agent-guide-meta:
  id: NF-STK-KEYCLOAK
  description: "How to deploy Keycloak in production with an optimized Quarkus image, managed PostgreSQL, KC_HOSTNAME set to the HTTPS URL, proxy headers and a bootstrap admin."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://www.keycloak.org/server/containers
    - https://www.keycloak.org/server/configuration-production
    - https://www.keycloak.org/server/db
    - https://www.keycloak.org/server/hostname
    - https://www.keycloak.org/server/reverseproxy
    - https://www.keycloak.org/server/health
    - https://www.keycloak.org/server/bootstrap-admin-recovery
    - https://quay.io/repository/keycloak/keycloak?tab=tags
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/create-path-based-security-policies
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DB-001, NF-DEP-001, NF-STK-GRAFANA, NF-STK-PROMETHEUS]
-->

# How to deploy Keycloak

To deploy Keycloak in production, build an optimized image from `quay.io/keycloak/keycloak` with `kc.sh build` (Postgres, health and metrics baked in), run it with `start --optimized`, point it at PostgreSQL through `KC_DB_*`, set `KC_HOSTNAME` to the full public HTTPS URL, enable HTTP behind the TLS-terminating proxy with `KC_PROXY_HEADERS`, and create the first admin from `KC_BOOTSTRAP_ADMIN_USERNAME` and `KC_BOOTSTRAP_ADMIN_PASSWORD`. Missing the hostname is the usual reason `start` refuses to boot.

This guide does it on Northflank with the CLI: managed PostgreSQL, a secret group for the database and bootstrap admin, and a service built from a two-stage `Dockerfile` in your repository, probed on the private management port. You end with Keycloak on a public HTTPS URL whose OpenID discovery document names that URL, verified with `curl`. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run; a human links the Git repository once.

## TL;DR: how to deploy Keycloak

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check the repo, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Create PostgreSQL with `northflank put addon` and a secret group that links it as `KC_DB_URL_HOST`, `KC_DB_URL_PORT`, `KC_DB_URL_DATABASE`, `KC_DB_USERNAME` and `KC_DB_PASSWORD`, plus the bootstrap admin.
4. Push a `Dockerfile` that runs `kc.sh build` and starts `kc.sh start --optimized`, and deploy it with `northflank put service combined`: public port 8080, private management port 9000, probes on `/health/ready`.
5. Set `KC_HOSTNAME=https://<host>` with `northflank patch service combined`, restart, and wait for the rollout.
6. Check that the live build is your pushed commit, the discovery document's `issuer` is `https://<host>/realms/master` and the bootstrap admin can get a token.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

Keycloak (the Quarkus distribution, 26.x at the time of writing) separates build-time options from runtime options. The containers docs recommend an optimized image: run `kc.sh build` once in a builder stage, so each start skips the re-augmentation step and boots faster. The other pieces come from the production docs:

- **Database.** `KC_DB=postgres` selects the vendor. `KC_DB_URL_HOST`, `KC_DB_URL_PORT` and `KC_DB_URL_DATABASE` build the JDBC URL for you, so linked credentials work without assembling a `jdbc:` string. Keycloak supports PostgreSQL 14 to 18.
- **Hostname.** `KC_HOSTNAME` "can be a full URL, or just a hostname". Behind a proxy on port 443, the docs recommend the full URL. `hostname-strict` defaults to `true`, so `start` needs it.
- **Proxy.** TLS ends at the load balancer, so `KC_HTTP_ENABLED=true` serves plain HTTP on 8080, and `KC_PROXY_HEADERS=xforwarded` tells Keycloak to read `X-Forwarded-*`.
- **Health on port 9000.** With health enabled, `/health/ready`, `/health/live` and `/health/started` are served on the management port, which the reverse proxy docs say not to expose.
- **Bootstrap admin.** Containers need `KC_BOOTSTRAP_ADMIN_USERNAME` and `KC_BOOTSTRAP_ADMIN_PASSWORD`, because otherwise the first admin can only be created from localhost. This account is temporary by design.

So where to deploy Keycloak? On something that runs a JVM container all the time, next to PostgreSQL, behind HTTPS, with a private port for health and metrics. Serverless platforms don't fit a stateful identity server with long boot times. A VM works if you run the proxy, TLS and database yourself; Kubernetes has the Keycloak Operator but needs a cluster. A container platform that builds your `Dockerfile` and provides managed PostgreSQL covers it.

## Prerequisites

- **Optional: Docker, to check the image builds locally** after step 8 adds the `Dockerfile`:
  ```bash
  docker build -t keycloak-local . && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl`** for generating the admin password.

Names used below: project `keycloak`, addon `keycloak-db`, secret group `keycloak-env`, service `keycloak`.

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
grep -E '^FROM' Dockerfile 2>/dev/null
ls Dockerfile conf/ themes/ providers/ 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId keycloak -o json >/dev/null 2>&1 \
  && echo "project: keycloak exists and will be reused" \
  || echo "project: keycloak will be created"
```

- This guide builds Keycloak from a repository that holds its `Dockerfile` (plus any themes or providers). If the repo already has a `Dockerfile` that isn't `FROM quay.io/keycloak/keycloak`, it belongs to another app. Stop and use a separate repo (or folder) for Keycloak, or use a guide for your framework.
- If a `FROM quay.io/keycloak/keycloak` line prints, compare that `Dockerfile` with step 8: keep your themes and providers, and make sure `KC_DB=postgres` and `KC_HEALTH_ENABLED=true` are set in the builder stage.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "keycloak", "region": "europe-west" }'
export PROJECT_ID=keycloak

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Create PostgreSQL and wait for it

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "keycloak-db", "type": "postgresql", "version": "16-latest",
  "tlsEnabled": false, "externalAccessEnabled": false,
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "keycloak" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId keycloak-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

The database only accepts traffic from inside the project. TLS and `dbName` are fixed at creation; storage can only grow.

## Step 6: Check the credential key names

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId keycloak-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

The PostgreSQL docs list `HOST`, `PORT`, `USERNAME`, `PASSWORD` and `DATABASE`. Adjust `keyName` in step 7 if the casing differs.

## Step 7: Create the secret group

```bash
KC_ADMIN_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=')
jq -n --arg pw "$KC_ADMIN_PASSWORD" '{
  name: "keycloak-env", type: "secret", secretType: "environment", priority: 10,
  restrictions: { restricted: false },
  addonDependencies: [ { addonId: "keycloak-db", keys: [
    { keyName: "HOST",     aliases: ["KC_DB_URL_HOST"] },
    { keyName: "PORT",     aliases: ["KC_DB_URL_PORT"] },
    { keyName: "DATABASE", aliases: ["KC_DB_URL_DATABASE"] },
    { keyName: "USERNAME", aliases: ["KC_DB_USERNAME"] },
    { keyName: "PASSWORD", aliases: ["KC_DB_PASSWORD"] }
  ] } ],
  secrets: { variables: { KC_BOOTSTRAP_ADMIN_USERNAME: "temp-admin", KC_BOOTSTRAP_ADMIN_PASSWORD: $pw } }
}' > keycloak-env.json
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --file keycloak-env.json >/dev/null && rm keycloak-env.json
```

The password is generated without `$`, because Keycloak expands `$` in option values (the db docs use a `KCRAW_` prefix to pass such values as-is). Keep `KC_ADMIN_PASSWORD` in your shell for the checks.

## Step 8: Deploy the Keycloak service

Add this `Dockerfile` at the repository root. Pin the tag to the current 26.x release from [Quay](https://quay.io/repository/keycloak/keycloak?tab=tags):

```dockerfile
FROM quay.io/keycloak/keycloak:26.8.2 AS builder
ENV KC_DB=postgres
ENV KC_HEALTH_ENABLED=true
ENV KC_METRICS_ENABLED=true
WORKDIR /opt/keycloak
RUN /opt/keycloak/bin/kc.sh build

FROM quay.io/keycloak/keycloak:26.8.2
COPY --from=builder /opt/keycloak/ /opt/keycloak/
ENTRYPOINT ["/opt/keycloak/bin/kc.sh"]
CMD ["start", "--optimized"]
```

This is the containers docs' example without the demo keystore, which isn't needed when TLS ends at the load balancer.

Northflank builds what's on the remote branch, not your local working tree, so commit and push it first:

```bash
git add Dockerfile
git commit -m "Add Northflank deployment files"
git push
```

Write `keycloak.json`. The first start runs with `KC_HOSTNAME_STRICT=false` because the public hostname isn't known yet; step 9 replaces it.

```json
{
  "name": "keycloak",
  "billing": { "deploymentPlan": "nf-compute-100-2" },
  "deployment": { "instances": 1, "docker": { "configType": "default" } },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/Dockerfile", "dockerWorkDir": "/" } },
  "ports": [
    { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" },
    { "name": "mgmt", "internalPort": 9000, "public": false, "protocol": "HTTP" }
  ],
  "runtimeEnvironment": { "KC_HTTP_ENABLED": "true", "KC_PROXY_HEADERS": "xforwarded", "KC_HOSTNAME_STRICT": "false" },
  "healthChecks": [
    { "protocol": "HTTP", "type": "startupProbe", "path": "/health/started", "port": 9000,
      "initialDelaySeconds": 10, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 30, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/health/ready", "port": 9000,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "HTTP", "type": "livenessProbe", "path": "/health/live", "port": 9000,
      "initialDelaySeconds": 10, "periodSeconds": 20, "timeoutSeconds": 3, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file keycloak.json
```

Use the same repository URL, provider and branch as in step 4. The startup probe gives the first boot, which creates the schema, up to five minutes. `nf-compute-100-2` (1 vCPU, 2 GB) is a starting point for a JVM; size up for heavy login traffic. The management port is private, so `/metrics` can be scraped by [Prometheus](deploy-prometheus.md) at `keycloak:9000`.

## Step 9: Set `KC_HOSTNAME` to the public URL

```bash
for i in $(seq 1 30); do
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId keycloak --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId keycloak --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns // empty' | head -n1)
  [ -n "$HOST" ] && break; sleep 10
done; echo "$HOST"
[ -n "$HOST" ] || { echo "NO_PUBLIC_HOST: stop here; the public port has no DNS name yet"; false; }

[ -n "$HOST" ] && northflank patch service combined --projectId "$PROJECT_ID" --serviceId keycloak --noDefaults -o json --input "$(jq -n --arg h "$HOST" '{
  runtimeEnvironment: { KC_HTTP_ENABLED: "true", KC_PROXY_HEADERS: "xforwarded", KC_HOSTNAME: ("https://" + $h) }
}')"
northflank restart service --projectId "$PROJECT_ID" --serviceId keycloak --noDefaults -o json
```

The patch sends the complete map, which drops `KC_HOSTNAME_STRICT=false`, so strict hostname checking is back on. Repeat this step with your own domain when you add one.

## Step 10: Wait for the build and rollout

```bash
OK=0
for i in $(seq 1 120); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId keycloak --noDefaults -o json)
  B=$(echo "$S" | jq -r '(.data // .).status.build.status // "NONE"')
  D=$(echo "$S" | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "build=$B deployment=$D"
  case "$B" in FAILURE|CRASHED|ABORTED|SUBMISSION_FAILURE) echo BUILD_FAILED; break;; esac
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  [ "$B" = "SUCCESS" ] && [ "$D" = "COMPLETED" ] && { OK=1; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
```

If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the build logs: `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId keycloak --lineLimit 200`. The troubleshooting table covers the common causes.

`COMPLETED` means the readiness probe on `/health/ready` passed, which includes the database connection.

**This step needs a human:** sign in at `https://$HOST/admin/` as `temp-admin` (password in the secret group), create a permanent admin user in the `master` realm, sign in as that user, and delete `temp-admin`. Keycloak flags the bootstrap account as temporary until then.

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId keycloak --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The discovery document uses the public HTTPS URL:
   ```bash
   curl -s "https://$HOST/realms/master/.well-known/openid-configuration" | jq -r '.issuer'   # https://<HOST>/realms/master
   ```
3. The admin account can get a token (database and admin are working). This prints only the token length:
   ```bash
   curl -s "https://$HOST/realms/master/protocol/openid-connect/token" \
     -d grant_type=password -d client_id=admin-cli -d username=temp-admin --data-urlencode "password=$KC_ADMIN_PASSWORD" \
     | jq '.access_token | length'
   ```
   After the human deletes `temp-admin`, use the permanent admin instead.
4. The service is on the optimized image and PostgreSQL. Logs should show the start with no build step and no H2 warnings:
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId keycloak --lineLimit 200 --textIncludes postgres
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `hostname is not configured` at start | `KC_HOSTNAME` missing and strict mode on | Redo step 9 |
| Admin console loops or loads `http://` resources | `KC_HOSTNAME` not the full `https://` URL | Set the full URL; restart |
| `Connection to localhost:5432 refused` | `KC_DB_URL_*` not injected | Check key names from step 6; restart |
| Warning that build options differ from the optimized image | A build-time option (`KC_DB`, `KC_HEALTH_ENABLED`) set at runtime with another value | Remove it from the runtime environment, or change the `Dockerfile` and rebuild |
| Readiness never passes, port 8080 serves | Probes not on 9000, or health not built in | Check `KC_HEALTH_ENABLED=true` in the builder stage |
| Locked out after deleting `temp-admin` | No permanent admin was created | Follow Keycloak's [bootstrap admin recovery](https://www.keycloak.org/server/bootstrap-admin-recovery) docs |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 10 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print `KC_BOOTSTRAP_ADMIN_PASSWORD`, `KC_DB_PASSWORD`, tokens or the credentials response.
- Hand the permanent-admin step to the user right after the first rollout.
- Don't make the management port public. The reverse proxy docs also advise keeping `/admin/` internal; if the user wants that, add a [path-based security policy](https://northflank.com/docs/v1/application/network/create-path-based-security-policies).
- Back up the database before a Keycloak major upgrade; migrations run on start. Get explicit confirmation before deleting the addon or service.

## FAQ

### Where should I deploy Keycloak?
On a host that runs the container permanently with PostgreSQL alongside it, terminates TLS for you and keeps the management port private. A container platform with Dockerfile builds and managed PostgreSQL does this with the official image; a VM or Kubernetes cluster works if you'll run the proxy and database yourself.

### Do I need the optimized build?
No, but it's the recommended production setup. Without it, `start` re-runs the build on every boot, which is slower.

### Can I run more than one instance?
Keycloak caches sessions in Infinispan, and instances must discover each other to share them. That clustering setup isn't covered here, so keep one instance until you've configured it.

### How do I upgrade Keycloak?
Change both `FROM` tags in the `Dockerfile`, push, and wait for the build and `COMPLETED`. Back up the database first.

## Conclusion

You now have Keycloak on an optimized Quarkus image, backed by managed PostgreSQL, with its hostname set to the public HTTPS URL, proxy headers on, health checks on a private port and a bootstrap admin ready to be replaced.

**Next:** create a realm and an OIDC client for your app, and point Grafana's login at it. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy Grafana](deploy-grafana.md)
- [How to deploy Prometheus](deploy-prometheus.md)
