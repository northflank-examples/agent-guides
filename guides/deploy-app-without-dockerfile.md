<!--
agent-guide-meta:
  id: NF-DEP-003
  description: "How to deploy an app without a Dockerfile using buildpacks that detect Node.js, Python, Go or Ruby, with a Procfile start process and a verified public URL."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/application/build/build-with-buildpacks
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
    - https://northflank.com/docs/v1/application/build/inject-build-arguments
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/api/project/services/create-combined-service
    - https://northflank.com/docs/v1/api/project/services/patch-combined-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
  related: [NF-DEP-001, NF-DEP-002, NF-DB-001, NF-JOB-002]
-->

# How to deploy an app without a Dockerfile

To deploy an app without a Dockerfile, build it from source with a buildpack: the builder detects the language from its manifest (`package.json`, `requirements.txt`, `go.mod`, `Gemfile`), installs dependencies and produces a container image, a `Procfile` `web:` line or the language default sets the start command, and the app listens on `0.0.0.0:$PORT`. A missing default process is the most common failure.

This guide does it on Northflank with the CLI: a combined service with `buildSettings.buildpack`, a pinned builder stack and a start process from a `Procfile` or a runtime override. You end with a public HTTPS URL that answers `curl`, with the build log showing which buildpack and process were detected. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run. A human only has to log in once and share the repository with Northflank.

## TL;DR: how to deploy an app without a Dockerfile

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check that the repo has a language manifest (`package.json`, `requirements.txt`/`pyproject.toml`, `go.mod`, `Gemfile`…) and no `Dockerfile`, and confirm the team, repository, branch and project before changing anything.
3. Give Northflank access to your repository on the Git integrations page (once), then create a project and check access.
4. Make sure the app listens on `0.0.0.0:$PORT` and has a start command (a `Procfile` `web:` line or the language default), then push.
5. Create the service with `"buildSettings": { "buildpack": { "builder": "HEROKU_24", "buildContext": "/" } }` using `northflank put service combined`.
6. Wait until the build is `SUCCESS` and the deployment is `COMPLETED`, then check that the live build is your pushed commit and that the URL answers.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Where to deploy an app without a Dockerfile

An app without a `Dockerfile` still needs to become something a host can run: either a platform that builds it from source for you, or an image you build yourself. Your options are roughly a **platform with source builds** (buildpacks or a similar detector) that turns the repo into a container and runs it with HTTPS; a **static or serverless host**, which fits frontends and functions but not every long-running server; or **Kubernetes or a VM**, where you'd have to produce the image or install the runtime yourself. Buildpacks give you the first option without changing the repo.

Buildpacks inspect the source, pick the language runtime, install dependencies and set a start command. That saves you writing and maintaining a `Dockerfile` for a conventional app, and it avoids committing a `Dockerfile` to a repo whose owners didn't ask for one.

Choose a `Dockerfile` instead when the app needs system packages the builder doesn't have, a non-standard build step, or a multi-language build that detection gets wrong. Then follow [how to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md).

## Prerequisites

- **The app starts locally** with its usual command (for example `npm start` or `gunicorn app:app`) and answers on `http://localhost:$PORT`.
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

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

Before changing anything, confirm that this guide fits the app and print the deployment target. Run this from the build context root (the repository root, or the app's folder in a monorepo):

```bash
ls package.json requirements.txt pyproject.toml Pipfile go.mod Gemfile composer.json pom.xml build.gradle Procfile 2>/dev/null
ls package-lock.json pnpm-lock.yaml yarn.lock poetry.lock uv.lock Pipfile.lock go.sum Gemfile.lock 2>/dev/null
ls Dockerfile 2>/dev/null && echo "Dockerfile: present" || echo "Dockerfile: none (buildpacks fit)"

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId buildpack-app -o json >/dev/null 2>&1 \
  && echo "project: buildpack-app exists and will be reused" \
  || echo "project: buildpack-app will be created"
```

- If the first line prints no manifest, the builder has nothing to detect. Stop: this guide needs a Node.js, Python, Go, Ruby, PHP or Java app, so use a guide for your framework or write a `Dockerfile`.
- The manifest and lockfile tell you the language and package manager the buildpack will use. A `Procfile` means the start command is already defined.
- If a `Dockerfile` already exists, you probably want [how to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md) instead. Building with buildpacks ignores it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "buildpack-app", "region": "europe-west" }'
export PROJECT_ID=buildpack-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Pick a builder

`buildSettings.buildpack.builder` accepts these values in the API spec: `HEROKU_26`, `HEROKU_24`, `HEROKU_22`, `HEROKU_22_CLASSIC`, `HEROKU_20`, `HEROKU_18`, `GOOGLE_22`, `GOOGLE_V1`, `CNB_ALPINE`, `CNB_BIONIC`, `PAKETO_RESOLUTE`, `PAKETO_NOBLE`, `PAKETO_JAMMY_TINY`, `PAKETO_JAMMY_BASE`, `PAKETO_JAMMY_FULL`, `PAKETO_TINY`, `PAKETO_BASE` and `PAKETO_FULL`. If you leave it out, the spec default is `HEROKU_26`.

The buildpacks docs list these supported stacks and languages:

| Builder (docs) | Likely enum | Languages |
|---|---|---|
| `heroku/builder:24` (Ubuntu 24.04) | `HEROKU_24` | Go, Java, Node.js, PHP, Python, Ruby, Scala, .NET |
| `heroku/builder:22` (Ubuntu 22.04) | `HEROKU_22` | Go, Java, Node.js, PHP, Python, Ruby, Scala, .NET |
| `paketobuildpacks/builder-jammy-full` | `PAKETO_JAMMY_FULL` | PHP, Java, Node.js, Go, Python, .NET Core, Ruby, NGINX, HTTPD |
| `paketobuildpacks/builder-jammy-base` | `PAKETO_JAMMY_BASE` | Java, Node.js, Go, Python, .NET Core, Ruby, NGINX |
| `gcr.io/buildpacks/builder:google-22` | `GOOGLE_22` | Node.js, Python, Go, Java, .NET, PHP, Ruby |

The docs mark `heroku/builder-classic:22`, `gcr.io/buildpacks/builder:v1` and the Ubuntu 18.04 Paketo stacks as deprecated, and the `cnbs/sample-builder` stacks as not for production. The enum-to-image mapping in the middle column is inferred from the names, not stated in the docs. This guide pins `HEROKU_24` because it appears in both the spec and the docs.

## Step 6: Make the app listen on `$PORT` and define the start process

The app must listen on `0.0.0.0` and the port in `$PORT`, for example `app.listen(process.env.PORT, '0.0.0.0')` or `gunicorn app:app --bind 0.0.0.0:$PORT`. Pin the language version too, so builds are reproducible: `engines.node` in `package.json`, `.python-version`, the `go` line in `go.mod`, `ruby` in `Gemfile`.

The builder sets a default `web` process when it can. If the start command isn't obvious, add a `Procfile` at the build context root:

```
web: gunicorn app:app --bind 0.0.0.0:$PORT
worker: python worker.py
```

Northflank runs `web` by default. Other process types (`worker` here) can run in separate services with `customProcess`. If the image ends up with **no** default process, the container won't start until you set a process or command, according to the override docs. That's the most common buildpack deployment failure.

Northflank builds what's on the remote branch, not your local working tree. If you added a `Procfile` (or changed anything else), commit and push it before step 7:

```bash
git add Procfile
git commit -m "Add Northflank deployment files"
git push
```

If you changed nothing, there's nothing to push.

## Step 7: Create the service

Write `app.json`:

```json
{
  "name": "app",
  "billing": { "deploymentPlan": "nf-compute-20" },
  "vcsData": { "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main" },
  "buildSettings": {
    "buildpack": {
      "builder": "HEROKU_24",
      "buildContext": "/",
      "buildpackLocators": [],
      "useCache": true
    }
  },
  "deployment": { "instances": 1, "buildpack": { "configType": "default" } },
  "ports": [ { "name": "http", "internalPort": 8080, "public": true, "protocol": "HTTP" } ],
  "runtimeEnvironment": { "PORT": "8080" },
  "healthChecks": [
    { "protocol": "TCP", "type": "readinessProbe", "port": 8080,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 2, "failureThreshold": 3, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file app.json
```

Use the same repository URL, provider and branch as in step 4. `billing.buildPlan` is optional; leave it out to use the default build plan, and only set it if builds run out of memory, using a build plan your team has access to (an unavailable one fails with `Build plan not found`).

REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined`. The first build starts automatically.

What the fields do, from the schema and docs:

- `buildContext` is the directory to build in (default `/`). For a monorepo, set it to the app's folder, for example `/apps/api`.
- `buildpackLocators` is an optional list of custom buildpacks, run as a group. Accepted formats include a URL to a `.tar.gz`, a GitHub URL with an optional `#tag`, `docker://` image references and `urn:cnb:registry:<id>`. Leave it empty to let the builder detect.
- `useCache` is the schema's flag for caching build dependencies (default `false`). The buildpacks docs say Northflank provides no built-in caching for buildpack builds, so don't count on faster rebuilds; use a Dockerfile with BuildKit caching if build time matters.
- `deployment.buildpack.configType` accepts:
  - `default`: the builder's default process, usually `web`.
  - `customProcess`: a named Procfile process, given in `customProcess`.
  - `customCommand`: an arbitrary command run through the buildpack launcher, which sets up the environment and runs the command in bash, so `$PORT` expands.
  - `customEntrypointCustomCommand` and `originalEntrypointCustomCommand`: set both entrypoint and command.
- A TCP readiness probe works for any app. Switch to an HTTP probe on a real health route when the app has one. See [configure health checks](https://northflank.com/docs/v1/application/observe/configure-health-checks).

To run the `worker` process from the same repo as a second service, copy `app.json`, rename it, remove `ports` and `healthChecks`, and set:

```json
"deployment": { "instances": 1, "buildpack": { "configType": "customProcess", "customProcess": "worker" } }
```

## Step 8: Wait for the deploy and get your URL

The first build takes a few minutes. This loop checks every 10 seconds and prints your URL when the app is live. `COMPLETED` means rolled out and serving.

```bash
OK=0
for i in $(seq 1 90); do
  S=$(northflank get service --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json)
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
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns' | head -n1)
  echo "https://$HOST"
fi
```

If it prints `ROLLOUT_NOT_COMPLETE`, don't continue; read the build logs in step 9 first. The troubleshooting table covers the common causes.

## Step 9: Read what the builder detected

The build log shows which buildpacks passed detection and which process became the default. Check it on the first deploy, even if everything worked:

```bash
BUILD_ID=$(northflank get service builds --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
           | jq -r '(.data // .).builds | sort_by(.createdAt) | last | .id')
northflank get service build-logs --projectId "$PROJECT_ID" --serviceId app --buildId "$BUILD_ID" --lineLimit 300
```

For more log options, see [view logs](https://northflank.com/docs/v1/application/observe/view-logs).

## How to know it worked

You're done when all of these hold:

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId app --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The URL is answered by your app. Use a route your app serves (`/` here, or its health route); it should print `200` (or the status that route normally returns). A `502` or `503` means nothing is listening on port 8080:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"
   ```
3. The step 9 build log names the expected language buildpack (for example Node.js or Python) and a `web` process.
4. The service logs show the app listening on `0.0.0.0` and the `PORT` you set (8080):
   ```bash
   northflank get service logs --projectId "$PROJECT_ID" --serviceId app --lineLimit 100
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| Build fails: no buildpack groups passed detection | No manifest at `buildContext`, or wrong builder for the language | Fix `buildContext`; check the stack table; or write a `Dockerfile` |
| Build `SUCCESS`, container exits at once or never starts | Image has no default process | Add a `Procfile` `web:` line, or set `configType: customProcess` / `customCommand` |
| Health check fails, logs show listening on `127.0.0.1` or port 5000/3000 | App ignores `$PORT` or binds to localhost | Bind `0.0.0.0:$PORT`; keep `PORT` and `internalPort` equal |
| Wrong language version installed | Version not pinned | Pin it in the manifest (`engines`, `.python-version`, `go.mod`) |
| Build fails installing a native dependency | Missing system library on the stack | Try a fuller stack (`PAKETO_JAMMY_FULL`) or switch to a `Dockerfile` |
| Build argument seems ignored | Buildpack overrides it (the docs warn about this) | Set the value through the buildpack's own config file, or use a `Dockerfile` |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

For deeper build debugging, see [view builds](https://northflank.com/docs/v1/application/observe/see-builds).

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 8 loop prints `ROLLOUT_NOT_COMPLETE`, read the build logs (step 9) before changing anything.
- Prefer buildpacks when the repo has no `Dockerfile` and the user didn't ask for one. Don't commit a new `Dockerfile` without asking.
- Adding a `Procfile` or pinning a language version changes the repo. Tell the user and commit it on a branch.
- Don't put secrets in `buildArguments`. Runtime secrets belong in a secret group with `secretType: "environment"`. See [manage secret groups](https://northflank.com/docs/v1/application/secure/manage-secret-groups).
- Don't switch an existing service from Dockerfile to buildpack (or back) without confirmation. It changes how the next build behaves.

## FAQ

### Where should I deploy my app if it has no Dockerfile?
Pick a host that can build from source, so you don't have to write and maintain a `Dockerfile` just to deploy. A container platform with buildpacks covers conventional Node.js, Python, Go and Ruby servers and keeps them running behind HTTPS; a static or serverless host suits frontends and functions. If the app needs unusual system packages or build steps, write a `Dockerfile` and deploy it as a container instead.

### Can I switch an existing service from buildpacks to a Dockerfile?
Yes. Patch `buildSettings` with a `dockerfile` object instead of `buildpack` (`northflank patch service combined`), set `deployment.docker`, and start a new build.

### Which builder should I choose?
Start with `HEROKU_24` for mainstream languages. Use the Paketo Jammy full stack (`PAKETO_JAMMY_FULL`) when you need extra system libraries or NGINX/HTTPD. Avoid the deprecated Ubuntu 18.04 stacks.

### Can I use multiple buildpacks, for example Node.js for assets and Python for the app?
Yes. List them in order in `buildpackLocators`. The builder runs through the group to find the ones that apply.

### Do buildpack apps work with jobs?
Yes. Jobs accept the same `buildSettings.buildpack` object and `deployment.buildpack` runtime modes. See [run an image once or on a schedule](https://northflank.com/docs/v1/application/run/run-an-image-once-or-on-a-schedule).

## Conclusion

You've deployed a repository with no `Dockerfile` by letting a buildpack detect and build it, chosen a supported builder explicitly, controlled the start process, and confirmed the build, the rollout and the URL.

**Next:** add a database with [how to deploy a PostgreSQL database](deploy-postgresql-database.md), or a worker with [how to deploy Celery in production](deploy-celery-in-production.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
- [How to deploy Celery in production](deploy-celery-in-production.md)
