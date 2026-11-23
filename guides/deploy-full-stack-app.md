<!--
agent-guide-meta:
  id: NF-DEP-012
  description: "How to deploy a full stack app (frontend, backend API and PostgreSQL) with Docker from one re-runnable template, wiring DATABASE_URL, the API URL and CORS."
  sources:
    - https://northflank.com/docs/v1/application/infrastructure-as-code/infrastructure-as-code
    - https://northflank.com/docs/v1/application/infrastructure-as-code/write-a-template
    - https://northflank.com/docs/v1/application/infrastructure-as-code/make-a-template-dynamic
    - https://northflank.com/docs/v1/application/infrastructure-as-code/template-nodes
    - https://northflank.com/docs/v1/application/infrastructure-as-code/run-a-template
    - https://northflank.com/docs/v1/api/team/templates/create-template
    - https://northflank.com/docs/v1/api/team/templates/run-template
    - https://northflank.com/docs/v1/api/team/templates/get-template-run
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
  related: [NF-DB-001, NF-DEP-004, NF-DEP-005, NF-DEP-001]
-->

# How to deploy a full stack app

To deploy a full stack app, run the frontend and the backend API as separate containers built from their Dockerfiles, give the API a managed PostgreSQL database through `DATABASE_URL`, pass the API's public URL to the frontend at build time and runtime, and allow CORS from the frontend origin. Declaring all of it in one file keeps the wiring repeatable instead of hand-assembled.

This guide does it on Northflank with the CLI, with the matching REST calls alongside: one JSON template declares the database, API, secrets and frontend; re-running it updates the stack in place. You end with public HTTPS URLs for both apps, checked with `curl` for health and CORS. Every step is a copy-pasteable command a developer or a coding agent (Claude Code, Codex, Cursor) can run end to end. Private repos need a human to link Git once.

## TL;DR: how to deploy a full stack app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo has a backend and a frontend `Dockerfile`, and confirm the team, repository, branch and project before changing anything.
3. Give Northflank access to your repository on the Git integrations page (once).
4. Declare the Postgres addon, backend, secret group and frontend in `template.json`, then create it with `northflank create template` and run it with `northflank run template`.
5. Wait for the run, then check that both live builds are your pushed commit, the backend answers `/health` and a database-backed endpoint, the frontend loads, and CORS allows the frontend origin.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why deploy a full stack app from a template?

A template turns your infrastructure into one file you can review, re-run and copy. On Northflank, **templates** are a `Workflow` of nodes (`Project`, `Addon`, `CombinedService`, `SecretGroup`, `Condition`, `Action`) that create or update resources by name. Running the template again after an edit updates the matching resources instead of creating new ones.

On where to host a full stack web app: the three parts need different things. The frontend can be static or server-rendered, the API is a long-running process, and the database needs persistent storage and backups. Static and serverless hosts handle the frontend well but usually leave the API and database to other providers, which means wiring URLs and credentials across accounts. A VM or Kubernetes cluster can run all three, with the setup and upkeep on you. A container platform with managed databases runs all three in one project, which is what lets one template describe the whole stack.

## Prerequisites

- **Both apps build locally with Docker** (use your own Dockerfile folders if they differ):
  ```bash
  docker build -q apps/api >/dev/null && docker build -q apps/web >/dev/null && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **The apps follow these rules:** the backend reads `DATABASE_URL`, binds `0.0.0.0:$PORT`, serves `GET /health` and allows CORS from the frontend origin; the frontend reads the API base URL from `API_URL` (rename it to `VITE_API_URL`, `NEXT_PUBLIC_API_URL` and so on to match the framework).

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

Before changing anything, confirm that this guide fits the repository and print the deployment target. Run this from the repository root:

```bash
ls apps/api/Dockerfile apps/web/Dockerfile
grep -rlF 'DATABASE_URL' apps/api --exclude-dir=node_modules | head -n3
grep -rlF '/health' apps/api --exclude-dir=node_modules | head -n3

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId fullstack-app -o json >/dev/null 2>&1 \
  && echo "project: fullstack-app exists and will be reused" \
  || echo "project: fullstack-app will be created"
```

- If there is no backend and frontend each with its own `Dockerfile`, this isn't a two-container full stack app. Stop and use a guide for your framework, or add the Dockerfiles first. If they live elsewhere (for example `/server` and `/client`), use those paths in step 5.
- If the `DATABASE_URL` or `/health` greps print nothing, add them to the backend first: the template injects `DATABASE_URL` and probes `/health`.
- Note each app's listening port. The template assumes 8080 for the backend and 3000 for the frontend.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch. The repository check is per project, which is why the project is created before the template: the template's `Project` node then matches it by name, so keep `PROJECT_NAME` and `REGION` in step 5 the same as here.

```bash
northflank create project -o json --input '{ "name": "fullstack-app", "region": "europe-west" }'
export PROJECT_ID=fullstack-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Write `template.json`

Keep resource `name`s stable: a re-run updates resources with the same name (ID); renaming creates a duplicate alongside the old one.

```json
{
  "apiVersion": "v1.2",
  "name": "Fullstack App",
  "description": "Postgres, backend API and frontend",
  "options": { "autorun": false, "concurrencyPolicy": "forbid" },
  "arguments": {
    "PROJECT_NAME": "fullstack-app",
    "REGION": "europe-west",
    "GIT_REPO": "https://github.com/<OWNER>/<REPO>",
    "GIT_BRANCH": "main"
  },
  "argumentOverrides": { "SESSION_SECRET": "${fn.randomSecret(64)}" },
  "spec": {
    "kind": "Workflow",
    "spec": {
      "type": "sequential",
      "steps": [
        {
          "kind": "Project", "ref": "project",
          "spec": { "name": "${args.PROJECT_NAME}", "region": "${args.REGION}", "networking": { "allowedIngressProjects": [] } }
        },
        {
          "kind": "Workflow",
          "spec": {
            "type": "sequential",
            "context": { "projectId": "${refs.project.id}" },
            "steps": [
              {
                "kind": "Addon", "ref": "db",
                "spec": {
                  "name": "db", "type": "postgresql", "version": "latest",
                  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 }
                }
              },
              {
                "kind": "CombinedService", "ref": "backend",
                "spec": {
                  "name": "backend",
                  "billing": { "deploymentPlan": "nf-compute-20" },
                  "deployment": { "instances": 1, "docker": { "configType": "default" } },
                  "vcsData": { "projectType": "github", "projectUrl": "${args.GIT_REPO}", "projectBranch": "${args.GIT_BRANCH}" },
                  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/apps/api/Dockerfile", "dockerWorkDir": "/apps/api" } },
                  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
                  "runtimeEnvironment": { "PORT": "8080" },
                  "healthChecks": [ { "protocol": "HTTP", "type": "readinessProbe", "path": "/health", "port": 8080,
                    "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 3, "successThreshold": 1 } ]
                }
              },
              {
                "kind": "SecretGroup", "ref": "backendenv",
                "spec": {
                  "name": "backend-env", "type": "secret", "secretType": "environment", "priority": 10,
                  "restrictions": { "restricted": true, "nfObjects": [ { "id": "${refs.backend.id}", "type": "service" } ] },
                  "secrets": { "variables": { "NODE_ENV": "production", "SESSION_SECRET": "${args.SESSION_SECRET}" }, "files": {} },
                  "addonDependencies": [
                    { "addonId": "${refs.db.id}", "keys": [ { "keyName": "POSTGRES_URI", "aliases": ["DATABASE_URL"] } ] }
                  ]
                }
              },
              {
                "kind": "CombinedService", "ref": "frontend",
                "spec": {
                  "name": "frontend",
                  "billing": { "deploymentPlan": "nf-compute-20" },
                  "deployment": { "instances": 1, "docker": { "configType": "default" } },
                  "vcsData": { "projectType": "github", "projectUrl": "${args.GIT_REPO}", "projectBranch": "${args.GIT_BRANCH}" },
                  "buildSettings": { "dockerfile": { "buildEngine": "buildkit", "dockerFilePath": "/apps/web/Dockerfile", "dockerWorkDir": "/apps/web" } },
                  "ports": [ { "name": "http", "internalPort": 3000, "public": true, "protocol": "HTTP" } ],
                  "buildArguments": { "API_URL": "https://${refs.backend.ports.0.dns}" },
                  "runtimeEnvironment": { "PORT": "3000", "API_URL": "https://${refs.backend.ports.0.dns}" }
                }
              },
              {
                "kind": "Workflow",
                "spec": {
                  "type": "parallel",
                  "steps": [
                    { "kind": "Condition", "spec": { "kind": "Addon", "spec": { "data": { "addonId": "${refs.db.id}" }, "type": "running" } } },
                    { "kind": "Condition", "spec": { "kind": "Service", "spec": { "data": { "serviceId": "${refs.backend.id}" }, "type": "running" } } },
                    { "kind": "Condition", "spec": { "kind": "Service", "spec": { "data": { "serviceId": "${refs.frontend.id}" }, "type": "running" } } }
                  ]
                }
              },
              {
                "kind": "Action", "ref": "restartbackend",
                "spec": { "kind": "Service", "spec": { "data": { "serviceId": "${refs.backend.id}" }, "type": "restart" } }
              }
            ]
          }
        }
      ]
    }
  }
}
```

Why this order:
- The secret group comes after `backend` because it references `${refs.backend.id}`; a node can only reference nodes that already ran. Using `restrictions.nfObjects` with a `${refs…}` value inside a `SecretGroup` node hasn't been confirmed: the field is documented for secret groups, but the doc example uses `restricted: false`.
- `${refs.backend.ports.0.dns}` is the backend's generated public hostname (first port). The frontend gets it as a build argument (for bundlers that inline env at build time) and as runtime env.
- The backend may start before `DATABASE_URL` exists; the conditions wait until the DB and both services are running, then the `Action` restarts the backend so it picks up the secret.
- `SESSION_SECRET` is generated once by `${fn.randomSecret(64)}` in `argumentOverrides` and stored outside the template; later runs reuse it. Put real secrets only in `argumentOverrides`, never in `arguments`.
- To deploy into an existing project instead, drop the `Project` node and set `"context": { "projectId": "<PROJECT_ID>" }` on the inner workflow.

Use the same repository URL, provider (`projectType`) and branch as in step 4, and your own Dockerfile paths and ports. `billing.buildPlan` is optional; leave it out to use the default build plan, and only set it if builds run out of memory, using a build plan your team has access to (an unavailable one fails with `Build plan not found`). Validate the file locally: `jq empty template.json`.

Northflank builds what's in your repository, not the files on your machine, so commit and push any Dockerfile or code changes from step 2 before the first run. `template.json` itself stays on your machine (it can be committed too; keep real secrets out of it):

```bash
git add apps/api apps/web
git commit -m "Prepare apps for Northflank"
git push
```

## Step 6: Create the template

```bash
export TEMPLATE_ID=$(northflank create template --noDefaults -o json --file template.json | jq -r '(.data // .).template.id')
echo "$TEMPLATE_ID"
northflank list templates -o json          # cross-check the ID, or find it here if the template already exists
```

The create response returns the new template under `template`, so read the ID from `template.id` instead of guessing it from the name.

REST: `POST https://api.northflank.com/v1/templates` with the file as body; the ID is at `.data.template.id`. If the template already exists, update it instead:

```bash
northflank update template --templateId "$TEMPLATE_ID" --noDefaults -o json --file template.json
# REST: POST https://api.northflank.com/v1/templates/{templateId}
```

## Step 7: Run it and wait

```bash
RUN_ID=$(northflank run template --templateId "$TEMPLATE_ID" --noDefaults -o json --input '{}' | jq -r '(.data // .).id')
OK=0
for i in $(seq 1 120); do
  R=$(northflank get template-run --templateId "$TEMPLATE_ID" --templateRunId "$RUN_ID" --noDefaults -o json)
  ST=$(echo "$R" | jq -r '(.data // .).status'); C=$(echo "$R" | jq -r '(.data // .).concluded')
  echo "run=$ST concluded=$C"
  [ "$C" = "true" ] && [ "$ST" = "success" ] && { OK=1; break; }
  [ "$C" = "true" ] && { echo RUN_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "RUN_NOT_COMPLETE: stop here and read the template run and the build or deployment logs"; false; }
if [ "$OK" = 1 ]; then
  for s in backend frontend; do
    H=$(northflank get service --projectId "$PROJECT_ID" --serviceId $s --noDefaults -o json \
      | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
    # Older API responses only list the host on the deprecated ports endpoint
    [ -n "$H" ] || H=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId $s --noDefaults -o json \
      | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
    [ $s = backend ] && BACK=$H || FRONT=$H
  done
  echo "backend:  https://$BACK"; echo "frontend: https://$FRONT"
fi
```

REST: `POST https://api.northflank.com/v1/templates/{templateId}/runs` (body `{"arguments": {...}}` to override arguments for this run), then `GET /v1/templates/{templateId}/runs/{templateRunId}`; fields are under `data`. If the guard prints `RUN_NOT_COMPLETE`, don't continue to the checks below: read `northflank get service build-logs --projectId "$PROJECT_ID" --serviceId backend` (or `frontend`). The first run can take 10+ minutes (addon provisioning plus two builds); poll no faster than every 10 seconds.

## Step 8: Re-run safely (idempotent updates)

Edit `template.json`, run `update template`, then step 7 again. Nodes create or update resources matched by name; existing resources are never deleted by a run. `concurrencyPolicy: "forbid"` stops two runs from overlapping.

## How to know it worked

You're done when all of these hold.

1. Both live builds are the commit you pushed. Each line should show the same SHA as `git rev-parse HEAD`:
   ```bash
   git rev-parse HEAD
   for s in backend frontend; do
     northflank get service builds --projectId "$PROJECT_ID" --serviceId $s --noDefaults -o json \
       | jq -r --arg s $s '(.data // .).builds | sort_by(.createdAt) | last | "\($s) \(.sha) \(.status)"'
   done
   ```
2. The run succeeded, the database is `running`, and both services show `build=SUCCESS deploy=COMPLETED` (`COMPLETED` means serving):
   ```bash
   northflank get template-run --templateId "$TEMPLATE_ID" --templateRunId "$RUN_ID" --noDefaults -o json | jq -r '(.data // .).status'
   northflank get addon --projectId "$PROJECT_ID" --addonId db --noDefaults -o json | jq -r '(.data // .).status'
   for s in backend frontend; do
     northflank get service --projectId "$PROJECT_ID" --serviceId $s --noDefaults -o json \
       | jq -r '(.data // .) | "\(.id) build=\(.status.build.status) deploy=\(.status.deployment.status)"'
   done
   ```
3. The backend answers, and reaches the database. `/health` should return 200. Then call one endpoint of your API that reads from the database (for example a list route); a 200 with data, not a 500, shows `DATABASE_URL` is wired:
   ```bash
   curl -s -o /dev/null -w 'backend /health %{http_code}\n' "https://$BACK/health"
   curl -s -w '\n%{http_code}\n' "https://$BACK/<DB_ROUTE>"
   ```
   If your `/health` already queries the database, the first line is enough.
4. The frontend loads:
   ```bash
   curl -s -o /dev/null -w 'frontend / %{http_code}\n' "https://$FRONT/"
   ```
5. CORS allows the frontend origin. This must print the frontend origin (or `*`); otherwise the browser blocks frontend → API calls:
   ```bash
   curl -s -D - -o /dev/null -H "Origin: https://$FRONT" "https://$BACK/health" | grep -i '^access-control-allow-origin'
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Run fails before any node executes | Dry run could not resolve a `${refs…}`/`${args…}` or JSON invalid | Check ref names and order; a ref must point to an earlier node |
| Node fails with project context error | Project resource node outside the workflow with `context` | Keep resource nodes inside the inner workflow |
| Step 4 check returns `accessible: false`, `CombinedService` node fails on repo, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the `CombinedService` node still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to both `vcsData` blocks as `"vcsLinkId": "<ID>"` |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |
| Condition node times out | Build failed or app crashes | `northflank get service build-logs …` / `logs …`; see [build with a Dockerfile](https://northflank.com/docs/v1/application/build/build-with-a-dockerfile) and [view logs](https://northflank.com/docs/v1/application/observe/view-logs) |
| Frontend calls `undefined/api` | Framework reads a differently named build-time var | Rename `API_URL` in `buildArguments` to the framework's prefix |
| Duplicate resources after a run | A `name` was changed | Restore old name or ask the user before deleting the duplicate |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't run the template until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows both live builds are the commit you pushed.
- If the step 7 loop prints `RUN_NOT_COMPLETE`, read the template run and the build logs before changing anything.
- Template runs never delete resources; deleting the old copies after a rename is destructive and needs explicit user confirmation per resource.
- Before changing the addon node on a database with data, back it up (see [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data)). Addon storage can only be increased, never reduced.
- Don't commit real secrets in `template.json`; use `argumentOverrides`.

## FAQ

### Where should I deploy my full stack app?
Ideally in one place that runs the frontend, the API and the database together, so the wiring between them stays private and in one file. A container platform with managed databases does that from your Dockerfiles. Splitting the frontend onto a static host also works, but you then manage the API URL and CORS across two providers.

### What happens if I run the template twice?
Nodes create or update resources matched by name, so a second run updates the existing stack. Runs never delete resources, and `concurrencyPolicy: "forbid"` stops two runs from overlapping.

### Why did I get duplicate resources?
A resource `name` was changed. Renaming creates a new resource alongside the old one. Restore the old name, or ask the user before deleting the duplicate.

### How does the frontend know the API URL?
The template passes `https://${refs.backend.ports.0.dns}`, the backend's generated public hostname, to the frontend as both a build argument and a runtime variable named `API_URL`. Rename it to match your framework's prefix if needed.

### Can I deploy into a project that already exists?
Yes. Drop the `Project` node and set `"context": { "projectId": "<PROJECT_ID>" }` on the inner workflow.

### Where do secrets go?
In `argumentOverrides`, never in `arguments`. `SESSION_SECRET` is generated once with `${fn.randomSecret(64)}` and reused by later runs.

## Conclusion

You now have a frontend, a backend API and a PostgreSQL database defined in one file, wired together with `DATABASE_URL` and the API URL, and deployable with a single idempotent run.

**Next:** create per-PR environments from the same stack with [set up a preview environment](https://northflank.com/docs/v1/application/release/pipeline/set-up-a-preview-environment). For wiring details, see [connect database secrets to workloads](https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads). For a single service without templates, see [how to deploy a Node.js API with PostgreSQL](deploy-nodejs-application.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy a Node.js application](deploy-nodejs-application.md)
- [How to deploy a Next.js app](deploy-nextjs-app.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
