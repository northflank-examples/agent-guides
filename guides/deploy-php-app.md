<!--
agent-guide-meta:
  id: NF-STK-PHP
  description: "How to deploy a PHP app with Docker: the official php Apache image with composer install, production php.ini, an optional managed MySQL database and HTTPS."
  sources:
    - https://hub.docker.com/_/php
    - https://hub.docker.com/_/composer
    - https://github.com/docker-library/php/issues/1631
    - https://frankenphp.dev/docs/docker/
    - https://github.com/php/frankenphp/blob/main/caddy/frankenphp/Caddyfile
    - https://www.php.net/manual/en/install.fpm.configuration.php
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/services/check-repository-access-for-services
    - https://northflank.com/docs/v1/api/project/services/put-combined-service
    - https://northflank.com/docs/v1/application/getting-started/link-your-git-account
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/project/addons/put-addon
    - https://northflank.com/docs/v1/api/project/addons/get-addon-credentials
    - https://northflank.com/docs/v1/api/project/secrets/put-project-secret
    - https://northflank.com/docs/v1/application/databases-and-persistence/deploy-databases-on-northflank/deploy-mysql-on-northflank
    - https://northflank.com/docs/v1/application/databases-and-persistence/connect-database-secrets-to-workloads
    - https://northflank.com/docs/v1/application/build/build-with-a-dockerfile
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DEP-009, NF-STK-MYSQL, NF-STK-WORDPRESS, NF-DEP-001]
-->

# How to deploy a PHP app

To deploy a PHP app, build a Docker image from the official `php` Apache image (or php-fpm with nginx, or FrankenPHP), run `composer install --no-dev` in a build stage, enable the production `php.ini` and the extensions you use, point the document root at your public folder, and read database credentials from environment variables. FTP uploads to shared hosting skip these steps, which causes most "works on my machine" PHP bugs.

This guide does it on Northflank with the CLI: a multi-stage `Dockerfile` on `php:8.5-apache` built from Git, an optional managed MySQL database linked as `DB_*` variables, and a health check. You end with a public HTTPS URL serving your app, a `/healthz.php` that confirms the database connection, and production settings verified inside the container. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is; a human links your Git account once.

## TL;DR: how to deploy a PHP app

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Check the app's layout and PHP version, confirm the team, repository, branch and project, and give Northflank access to the repository.
3. Push a `Dockerfile` (a `composer:2` stage, then `php:8.5-apache` with `php.ini-production`, `pdo_mysql` and the document root at `public/`) and a `public/healthz.php`.
4. Optionally create MySQL with `northflank put addon` and link `HOST`, `PORT`, `DATABASE`, `USERNAME` and `PASSWORD` as `DB_*` with `northflank put secret`.
5. Deploy with `northflank put service combined` (HTTP port 80, readiness probe on `/healthz.php`) and wait until `COMPLETED`.
6. Check that the live build is your pushed commit, `/healthz.php` returns `ok`, `display_errors` is off and `vendor/` isn't reachable.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why a container for PHP?

PHP itself is stateless per request, which makes it easy to run anywhere, but the runtime around it isn't: the PHP version, extensions, `php.ini`, the web server config and the `vendor/` folder all have to match what the code expects. The official `php` image packages PHP with Apache (`-apache`), with FastCGI for a separate web server (`-fpm`) or as a CLI. Building the image from Git pins all of that in one file.

When choosing where to deploy a PHP app, the options are shared PHP hosting (cheap, but little control over versions and extensions, and usually no long-running workers), a VM with Apache or nginx you patch yourself, serverless platforms that need a PHP runtime layer, or a container platform that runs your image. On Northflank, a **combined service** builds the `Dockerfile` from Git on every push and serves it over HTTPS, and a MySQL **addon** gives you a managed database. If you're on Laravel, follow [how to deploy a Laravel project](deploy-laravel-project.md) instead: it adds the queue worker, scheduler and migrations Laravel needs.

## Prerequisites

- **A PHP app whose entry point passes a syntax check** (use `index.php` if there's no `public/` folder):
  ```bash
  php -l public/index.php && echo BUILD_OK
  ```
- **The code is in a GitHub, GitLab or Bitbucket repository** that you can push to.
- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.

Names used below: project `php-app`, addon `php-db`, secret group `php-env`, service `web`.

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
jq -r '"php: \(.require.php // "unset")  extensions: \([.require | keys[] | select(startswith("ext-"))] | join(" "))"' composer.json 2>/dev/null || echo "composer.json: missing"
ls public/index.php index.php composer.lock Dockerfile 2>/dev/null

echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
echo "repo:    $(git remote get-url origin)"
echo "branch:  $(git branch --show-current)"
northflank get project --projectId php-app -o json >/dev/null 2>&1 \
  && echo "project: php-app exists and will be reused" \
  || echo "project: php-app will be created"
```

- If there's neither `public/index.php` nor `index.php`, this isn't a plain PHP web app. Stop and use a guide for your framework (for Laravel, use a Laravel guide).
- This guide assumes `public/index.php` as the web root and `composer.json` plus `composer.lock` at the repo root. If `index.php` is at the root, set `APACHE_DOCUMENT_ROOT=/var/www/html` in step 5. If there's no `composer.json`, delete the `vendor` stage.
- The `php` constraint sets the image tag (`8.5`, `8.4`…), and each `ext-*` is an extension to add to `docker-php-ext-install`.
- If a `Dockerfile` already exists, compare it with step 5 before replacing it.
- If the remote is an SSH address (`git@github.com:owner/repo.git`), use its HTTPS form (`https://github.com/owner/repo`) in the steps below.

**If you're an agent:** show the user the team, repository, branch, region (`europe-west` below, unless they choose another) and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Give Northflank access to your repository

**This step needs a human, once per repository.** Open the [Git integrations page](https://app.northflank.com/s/account/integrations/vcs) and click **Link GitHub** (or GitLab or Bitbucket). When GitHub asks which repositories to share, include this one. If the account is already linked, select it and click **Edit installation** to add the repository. Public repositories need no access. See [link your Git account](https://northflank.com/docs/v1/application/getting-started/link-your-git-account).

## Step 4: Create a project and check repository access

A project holds your services and databases. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "php-app", "region": "europe-west" }'
export PROJECT_ID=php-app

northflank get service repository-access --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "projectUrl": "https://github.com/<OWNER>/<REPO>", "projectType": "github", "projectBranch": "main"
}' | jq '(.data // .) | {accessible, reason}'
```

Use your repository's URL and branch, and `gitlab` or `bitbucket` as the `projectType` if that's where it lives. Continue when `accessible` is `true`. If it's `false`, Northflank can't see the repository yet: go back to step 3.

## Step 5: Write the `Dockerfile` and a health check

```dockerfile
FROM composer:2 AS vendor
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --prefer-dist --no-interaction --optimize-autoloader --ignore-platform-reqs

FROM php:8.5-apache
RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini" \
 && docker-php-ext-install pdo_mysql \
 && a2enmod rewrite
ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf \
 && sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf
WORKDIR /var/www/html
COPY . .
COPY --from=vendor /app/vendor ./vendor
RUN chown -R www-data:www-data /var/www/html
EXPOSE 80
```

- **`php.ini-production`**: the image docs strongly recommend it for production. It turns `display_errors` off, so errors go to the log instead of the page.
- **`docker-php-ext-install`** is the image's helper for core extensions. Add the ones your app needs (`gd`, `intl`, `zip`…); some need `-dev` system packages first, as the image docs show for `gd`. OPcache is built into PHP 8.5 and can't be installed this way (`docker-php-ext-install opcache` fails on 8.5 images); on PHP 8.4 or earlier, add `opcache` to the list.
- **`--ignore-platform-reqs`** lets the `composer` image resolve packages even though its PHP extensions differ from the runtime image. Keep it only if Composer complains about a missing `ext-*`; the runtime stage must still install those extensions.
- The `sed` lines are the image docs' way to change Apache's document root. The default command starts Apache on port 80.
- **`--no-scripts`** skips Composer scripts, because the `vendor` stage only has `composer.json`. If your app relies on them, run `composer run-script post-install-cmd` in the runtime stage after copying `composer` in.

Add a health check at `public/healthz.php`. It confirms PHP runs and, when `DB_HOST` is set, that the database answers. It never prints credentials:

```php
<?php
header('Content-Type: text/plain');
if (getenv('DB_HOST')) {
    try {
        $dsn = sprintf('mysql:host=%s;port=%s;dbname=%s', getenv('DB_HOST'), getenv('DB_PORT') ?: '3306', getenv('DB_NAME'));
        new PDO($dsn, getenv('DB_USER'), getenv('DB_PASSWORD'), [PDO::ATTR_TIMEOUT => 2]);
    } catch (Throwable $e) {
        http_response_code(503);
        exit("db error\n");
    }
}
echo "ok\n";
```

Add a `.dockerignore` with `vendor`, `.env`, `.git` and `node_modules`.

**Alternatives.**

- **php-fpm + nginx:** use `php:8.5-fpm` with nginx in a second container or process. php-fpm's `clear_env` setting defaults to `yes`, which hides container environment variables from PHP; set `clear_env = no` in the pool config or `getenv('DB_HOST')` returns `false`.
- **FrankenPHP:** `dunglas/frankenphp:1-php8.5` serves `/app/public` through Caddy and includes `install-php-extensions`. Copy the app to `/app` and set `SERVER_NAME=:8080` so it serves plain HTTP on an unprivileged port (TLS ends at the platform's load balancer), then use 8080 as the service port below.

Northflank builds what's on the remote branch, not your local working tree, so commit and push these files before creating the service:

```bash
git add Dockerfile .dockerignore public/healthz.php
git commit -m "Add Northflank deployment files"
git push
```

## Step 6 (optional): Create the MySQL database

Skip steps 6 and 7 if the app has no database.

```bash
northflank put addon --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "php-db", "type": "mysql", "version": "8-latest",
  "billing": { "deploymentPlan": "nf-compute-20", "storage": 6144, "replicas": 1 },
  "customCredentials": { "dbName": "app" }
}'

OK=0
for i in $(seq 1 90); do
  A=$(northflank get addon --projectId "$PROJECT_ID" --addonId php-db --noDefaults -o json | jq -r '(.data // .).status')
  echo "addon=$A"
  case "$A" in running) OK=1; break;; failed|error|errorAllocating|deleting|deleted) echo ADDON_FAILED; break;; esac
  sleep 10
done
[ "$OK" = 1 ] || { echo "ADDON_NOT_READY: stop here and check the addon in the UI"; false; }
```

Storage is in MB and can only be increased. `customCredentials.dbName` can't be changed after creation. For versions and options, see [how to deploy a MySQL database](deploy-mysql-database.md).

## Step 7 (optional): Link the database into a secret group

Confirm the credential key names first. Print names only, never values:

```bash
northflank get addon credentials --projectId "$PROJECT_ID" --addonId php-db --noDefaults -o json \
  | jq '(.data // .) | {secrets: (.secrets | keys), envs: (.envs | keys)}'
```

If the output uses different names, change the `keyName` values below to match:

```bash
northflank put secret --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "php-env", "type": "secret", "secretType": "environment", "priority": 10,
  "restrictions": { "restricted": false },
  "addonDependencies": [ { "addonId": "php-db", "keys": [
    { "keyName": "HOST",     "aliases": ["DB_HOST"] },
    { "keyName": "PORT",     "aliases": ["DB_PORT"] },
    { "keyName": "DATABASE", "aliases": ["DB_NAME"] },
    { "keyName": "USERNAME", "aliases": ["DB_USER"] },
    { "keyName": "PASSWORD", "aliases": ["DB_PASSWORD"] }
  ] } ],
  "secrets": { "variables": { "APP_ENV": "production" } }
}'
```

The password never passes through your terminal. Variables are injected at runtime, so read them with `getenv()` in the app, not at build time.

## Step 8: Create the service

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
    { "protocol": "HTTP", "type": "readinessProbe", "path": "/healthz.php", "port": 80,
      "initialDelaySeconds": 5, "periodSeconds": 10, "timeoutSeconds": 3, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "TCP", "type": "livenessProbe", "port": 80,
      "initialDelaySeconds": 15, "periodSeconds": 20, "timeoutSeconds": 3, "failureThreshold": 3 }
  ]
}
```

```bash
northflank put service combined --projectId "$PROJECT_ID" --noDefaults -o json --file web.json
```

Use the same repository URL, provider and branch as in step 4. REST: `PUT https://api.northflank.com/v1/projects/{projectId}/services/combined`. If no build appears within a minute, run `northflank start service build --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json --input '{}'`.

Readiness checks the database through `healthz.php`, so a database outage takes the app out of rotation instead of serving errors. Liveness only checks the port, so the same outage doesn't restart Apache in a loop.

**Sessions and uploads.** PHP's default file sessions and anything written to `uploads/` live in the container and disappear on each deploy. Store sessions in the database or Redis, and files in object storage, before raising `instances`.

## Step 9: Wait for the rollout and get the URL

```bash
OK=0
for i in $(seq 1 90); do
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

`COMPLETED` means rolled out and serving. If the secret group changes later, run `northflank restart service --projectId "$PROJECT_ID" --serviceId web`.

## How to know it worked

You're done when all of these hold. `exec` drops output without a TTY, so wrap it in `script` (Linux: `script -qefc '<cmd>' /dev/null`):

1. The live build is the commit you pushed. The two values should match:
   ```bash
   git rev-parse HEAD
   northflank get service builds --projectId "$PROJECT_ID" --serviceId web --noDefaults -o json \
     | jq -r '(.data // .).builds | sort_by(.createdAt) | last | "\(.sha) \(.status)"'
   ```
2. The health check passes, including the database when one is linked:
   ```bash
   curl -s -w ' %{http_code}\n' "https://$HOST/healthz.php"     # ok 200
   ```
3. The app's home page answers:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/"      # 200 (or your app's redirect)
   ```
4. Production settings and extensions are active, and the database variables reach the container (names only):
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId web --shell-cmd 'sh -c' \
     --cmd '"php -i | grep ^display_errors; php -m | grep -x pdo_mysql; env | cut -d= -f1 | grep -x DB_HOST"'
   ```
   This prints `display_errors => Off => Off`, `pdo_mysql` and `DB_HOST`. Check 2 already proved that Apache's PHP can read them.
5. Composer's `vendor/` folder isn't reachable from the web, because the document root is `public/`:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/vendor/autoload.php"   # 404 (or your router's not-found page), never PHP output
   ```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Step 4 check returns `accessible: false`, or `Repository "…" not found for GitHub installation` | The repository isn't shared with Northflank | Step 3: on the Git integrations page, select the account, click **Edit installation** and add the repository. Then re-run the step 4 check |
| Check passes but the service still can't find the repo | Several Git accounts are linked and the wrong one is used | Add the check's `vcsLinkId` (in its full output) to `vcsData` as `"vcsLinkId": "<ID>"` |
| `Class "…" not found` / `vendor/autoload.php` missing | Composer stage skipped or its `COPY --from=vendor` line removed | Keep both; rebuild |
| `could not find driver` | `pdo_mysql` not installed | Add it to `docker-php-ext-install` |
| `SQLSTATE[HY000] [2002]` on `localhost` | `DB_HOST` not injected, or php-fpm `clear_env` | Check step 7's key names; set `clear_env = no` for fpm |
| 404 on every route except `/` | `mod_rewrite` off or no `.htaccess` front controller | Keep `a2enmod rewrite`; add the framework's `public/.htaccess` |
| 403 Forbidden on `/` | Document root has no `index.php` | Fix `APACHE_DOCUMENT_ROOT` |
| Users logged out after every deploy | File sessions in the container | Use database or Redis sessions |
| Build killed or out of memory | Default build plan too small | Set `billing.buildPlan` to a larger build plan your team has access to (an unavailable one fails with `Build plan not found`) |
| `Build plan not found` | `billing.buildPlan` names a plan your team can't use | Remove `buildPlan` to use the default |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write, and don't create the service until the step 4 check returns `accessible: true`. If it doesn't, ask the user to share the repository (step 3) instead of retrying.
- Don't report success until check 1 shows the live build is the commit you pushed.
- If the step 9 loop prints `ROLLOUT_NOT_COMPLETE`, read the build or deployment logs before changing anything.
- Never print `DB_PASSWORD` or the credentials response; print key names only.
- Never switch to `php.ini-development` or turn `display_errors` on in production to debug. Read `northflank get service logs` instead.
- Back up the database before running schema changes against it (see [backup, restore and import data](https://northflank.com/docs/v1/application/databases-and-persistence/backup-restore-and-import-data)).
- Don't delete the addon, service or project without explicit, per-resource confirmation.

## FAQ

### Where should I deploy my PHP app?
On a host that runs your PHP version, extensions and web server config exactly as you built them. A container platform with a managed MySQL database covers that from one `Dockerfile`. Shared PHP hosting can be enough for a small site with no workers and standard extensions.

### Apache, php-fpm with nginx, or FrankenPHP?
Apache with mod_php is the simplest single container and supports `.htaccess`. php-fpm with nginx splits the web server from PHP and is common in larger setups. FrankenPHP is one binary with Caddy and adds worker mode for frameworks that support it. All three deploy the same way here.

### Can I deploy a PHP app on Vercel or other serverless hosts?
Only with a community PHP runtime, and long-running work, file sessions and local uploads don't fit that model. A container keeps standard PHP behaviour.

### How do I deploy Laravel or WordPress?
Use [how to deploy a Laravel project](deploy-laravel-project.md) for Laravel (queue worker, scheduler, migrations) and [how to deploy a WordPress website](deploy-wordpress-website.md) for WordPress.

## Conclusion

You now have a PHP app built from Git into the official Apache image with Composer dependencies, production `php.ini` and the extensions it needs, serving over HTTPS, with an optional managed MySQL database linked through a secret group and a health check that tests the connection.

**Next:** add your domain with [domains on Northflank](https://northflank.com/docs/v1/application/domains/domains-on-northflank), or Redis for sessions with [how to deploy Redis](deploy-redis.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy a Laravel project](deploy-laravel-project.md)
- [How to deploy a MySQL database](deploy-mysql-database.md)
- [How to deploy a WordPress website](deploy-wordpress-website.md)
- [How to deploy Redis](deploy-redis.md)
- [How to deploy a Docker container from a Git repository](deploy-docker-container-from-git.md)
