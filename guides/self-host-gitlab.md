<!--
agent-guide-meta:
  id: NF-STK-GITLAB
  description: "How to self-host GitLab CE with the official Omnibus Docker image, a persistent volume for config, logs and data, an HTTPS external_url and optional SSH."
  sources:
    - https://northflank.com/stacks/deploy-gitlab-ce
    - https://docs.gitlab.com/install/docker/installation/
    - https://docs.gitlab.com/install/docker/configuration/
    - https://docs.gitlab.com/install/requirements/
    - https://docs.gitlab.com/administration/monitoring/health_check/
    - https://docs.gitlab.com/omnibus/settings/ssl/
    - https://docs.gitlab.com/omnibus/settings/nginx/
    - https://docs.gitlab.com/omnibus/settings/memory_constrained_envs/
    - https://gitlab.com/gitlab-org/omnibus-gitlab/-/blob/master/docker/Dockerfile
    - https://hub.docker.com/r/gitlab/gitlab-ce/tags
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/services/get-service
    - https://northflank.com/docs/v1/api/project/services/get-service-ports
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/api/team/load-balancers/put-load-balancer
    - https://northflank.com/docs/v1/api/team/load-balancers/get-load-balancer
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/configure-load-balancers
    - https://northflank.com/docs/v1/application/observe/configure-health-checks
  related: [NF-DEP-002, NF-STK-N8N, NF-DB-001]
-->

# How to self-host GitLab

To self-host GitLab, run the official Omnibus Docker image (`gitlab/gitlab-ce`) on a machine with at least 8 GB of memory and a 256 MB `/dev/shm`, persist `/etc/gitlab`, `/var/log/gitlab` and `/var/opt/gitlab`, and set `external_url` through `GITLAB_OMNIBUS_CONFIG`. Behind a TLS-terminating proxy, turn off the bundled NGINX's HTTPS listener, and remember that Git over SSH on port 22 needs its own TCP route.

This guide does it on Northflank with the CLI: one `gitlab/gitlab-ce` service on an 8 vCPU, 16 GB plan, a single volume mounted at the three GitLab paths, and `external_url` set from the HTTPS hostname Northflank assigns. You end with GitLab on a public HTTPS URL with HTTPS clones, verified with `curl` and GitLab's readiness check. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run; a human signs in as `root` once.

## TL;DR: how to self-host GitLab

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Pin a `gitlab/gitlab-ce:<version>-ce.0` tag, check the plan, and confirm the team and project before changing anything.
3. Create a project, then a `gitlab/gitlab-ce` service at 0 instances with `shmSize: 256` and one volume mounted at `/etc/gitlab`, `/var/log/gitlab` and `/var/opt/gitlab`.
4. Read the public host, then set `GITLAB_OMNIBUS_CONFIG` (`external_url 'https://<host>'`, NGINX on port 80 without HTTPS) and start 1 instance.
5. Wait for the first boot; a human signs in as `root`. Optionally route SSH through a TCP load balancer.
6. Verify that `/users/sign_in` returns 200 and `/-/readiness?all=1` reports `ok`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

The GitLab Docker image is the Omnibus Linux package in a container: PostgreSQL, Redis, Gitaly, Puma, Sidekiq and NGINX all run inside it. GitLab lists 8 vCPU and 16 GB as the single-node baseline (8 GB minimum when memory-constrained). It needs three persistent paths:

- **`/etc/gitlab`**: `gitlab.rb`, secrets and SSH host keys. Lose it and encrypted data becomes unreadable.
- **`/var/opt/gitlab`**: the bundled PostgreSQL, Redis and every Git repository.
- **`/var/log/gitlab`**: logs.

Behind a platform proxy, `external_url` must be the public `https://` address (or clone URLs and redirects break), and because TLS ends at the proxy, the bundled NGINX must listen on plain port 80 and skip Let's Encrypt.

Be clear about scope: this is one container with no high availability. GitLab's Docker docs advise against running this image on Kubernetes because it's a single point of failure, and point to the Helm chart for HA. Northflank runs on Kubernetes, so treat this as a single-node GitLab backed by volume backups.

Where to host GitLab comes down to real CPU and memory, fast persistent disk, a stable HTTPS URL and, for SSH clones, a TCP port. Serverless and static hosts can't run it. A VM works, but you patch the OS, manage TLS and back up the disk yourself; a container platform with volumes and HTTPS ingress runs the same image with less to operate.

If you'd rather click through it, Northflank has a [GitLab CE stack template](https://northflank.com/stacks/deploy-gitlab-ce) that deploys it from the UI.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **A plan with at least 8 GB of memory.** `nf-compute-800-16` (8 vCPU, 16 GB) matches GitLab's baseline; check that your team can use it in step 2.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Pick a version and confirm where it will deploy

Pin a GitLab version (tags look like `19.6.1-ce.0`), check that the plan exists, and print the target:

```bash
GITLAB_TAG=$(curl -s "https://hub.docker.com/v2/repositories/gitlab/gitlab-ce/tags?page_size=100&ordering=last_updated" \
  | jq -r '.results[].name' | grep -E '^[0-9]+\.[0-9]+\.[0-9]+-ce\.0$' | sort -V | tail -n1)
echo "gitlab:  ${GITLAB_TAG:-NOT FOUND}"
curl -s https://api.northflank.com/v1/plans | jq -r '.data.plans[] | select(.id == "nf-compute-800-16" or .id == "nf-compute-400") | "plan:    \(.id) cpu=\(.cpuResource) mem=\(.ramResource)"'
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId gitlab -o json >/dev/null 2>&1 \
  && echo "project: gitlab exists and will be reused" \
  || echo "project: gitlab will be created"
```

- If the version prints `NOT FOUND`, pick a tag from [Docker Hub](https://hub.docker.com/r/gitlab/gitlab-ce/tags). For Enterprise Edition, use `gitlab/gitlab-ee` and `-ee.0` tags instead.
- `nf-compute-800-16` (8 vCPU, 16 GB) matches GitLab's baseline. `nf-compute-400` (4 vCPU, 8 GB) can work for a small team with the memory settings in the FAQ.
- Names used below: project `gitlab`, service `gitlab`, volume `gitlab-data`, load balancer `gitlab-ssh`.

**If you're an agent:** show the user the team, the GitLab tag, the plan, the region (`europe-west` below, unless they choose another) and whether the project is new or reused. GitLab needs a large plan, so confirm the cost is expected. Wait for them to confirm before you create anything.

## Step 3: Create a project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "gitlab", "region": "europe-west" }'
export PROJECT_ID=gitlab
```

## Step 4: Create the GitLab service at 0 instances

The service starts at 0 instances so GitLab's first boot happens once, with the volume attached and the right `external_url`. Write `gitlab.json`, replacing `<GITLAB_TAG>`:

```json
{
  "name": "gitlab",
  "billing": { "deploymentPlan": "nf-compute-800-16" },
  "deployment": {
    "instances": 0,
    "external": { "imagePath": "gitlab/gitlab-ce:<GITLAB_TAG>" },
    "docker": { "configType": "default" },
    "storage": { "shmSize": 256 }
  },
  "ports": [
    { "name": "http", "internalPort": 80, "public": true, "protocol": "HTTP" },
    { "name": "ssh", "internalPort": 22, "public": false, "protocol": "TCP" }
  ],
  "healthChecks": [
    { "protocol": "CMD", "type": "startupProbe", "cmd": "/opt/gitlab/bin/gitlab-healthcheck --fail --max-time 10",
      "initialDelaySeconds": 60, "periodSeconds": 30, "timeoutSeconds": 30, "failureThreshold": 40, "successThreshold": 1 },
    { "protocol": "CMD", "type": "readinessProbe", "cmd": "/opt/gitlab/bin/gitlab-healthcheck --fail --max-time 10",
      "initialDelaySeconds": 10, "periodSeconds": 30, "timeoutSeconds": 30, "failureThreshold": 3, "successThreshold": 1 },
    { "protocol": "CMD", "type": "livenessProbe", "cmd": "/opt/gitlab/bin/gitlab-healthcheck --fail --max-time 10",
      "initialDelaySeconds": 10, "periodSeconds": 60, "timeoutSeconds": 30, "failureThreshold": 5, "successThreshold": 1 }
  ]
}
```

```bash
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file gitlab.json
```

`shmSize: 256` matches `shm_size: '256m'` in GitLab's Compose example. The probes run `gitlab-healthcheck`, the image's own Docker `HEALTHCHECK`; HTTP probes on `/-/readiness` would fail because GitLab only answers monitoring endpoints from its IP allowlist (localhost by default). The startup probe allows 20 minutes for the first `reconfigure`. Port 22 stays private: public service ports are HTTP only, so raw TCP needs a load balancer (step 8).

## Step 5: Attach one volume at three paths

A volume can mount different subdirectories at different paths, so one volume holds all three GitLab directories:

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId gitlab-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "gitlab-data",
  "mounts": [
    { "containerMountPath": "/etc/gitlab",     "volumeMountPath": "config" },
    { "containerMountPath": "/var/log/gitlab", "volumeMountPath": "logs" },
    { "containerMountPath": "/var/opt/gitlab", "volumeMountPath": "data" }
  ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 51200 },
  "attachedObjects": [ { "id": "gitlab", "type": "service" } ]
}'
```

GitLab asks for 40 GB for the application plus all your repositories, so 50 GB is a floor. Volume storage can grow but never shrink. `create volume` returns 409 if the volume exists, hence the `get` guard. A read/write-once volume limits the service to one instance, which is all the Omnibus image supports anyway.

## Step 6: Set `external_url` and start GitLab

Read the hostname Northflank assigned to the public port:

```bash
for i in $(seq 1 30); do
  HOST=$(northflank get service --projectId "$PROJECT_ID" --serviceId gitlab --noDefaults -o json \
    | jq -r '[(.data // .).ports[]? | select(.public == true) | .dns // empty][0] // empty')
  # Older API responses only list the host on the deprecated ports endpoint
  [ -n "$HOST" ] || HOST=$(northflank get service ports --projectId "$PROJECT_ID" --serviceId gitlab --noDefaults -o json \
    | jq -r '(.data // .).ports[] | select(.public == true) | .dns // empty' | head -n1)
  [ -n "$HOST" ] && break; sleep 10
done; echo "https://$HOST"
```

If `HOST` is still empty, the hostname may not be assigned until an instance runs. Run the patch below with an `external_url` of `http://localhost` first, then repeat this loop and the patch with the real host.

Now set the Omnibus configuration and start one instance. Settings in `GITLAB_OMNIBUS_CONFIG` are separated by semicolons and evaluated on every start:

```bash
OMNIBUS="external_url 'https://$HOST'; gitlab_rails['nginx']['listen_port'] = 80; gitlab_rails['nginx']['listen_https'] = false; letsencrypt['enable'] = false"
northflank patch service deployment --projectId "$PROJECT_ID" --serviceId gitlab --noDefaults -o json --input "$(jq -n --arg c "$OMNIBUS" '{
  deployment: { instances: 1 },
  runtimeEnvironment: { GITLAB_OMNIBUS_CONFIG: $c }
}')"
```

The `gitlab_rails['nginx']` keys are the GitLab 19.2+ names; older images use `nginx['listen_port']` and `nginx['listen_https']`. With an `https://` external URL, GitLab sets `X-Forwarded-Proto: https` itself. The patch replaces the whole `runtimeEnvironment` map, so always send every variable.

## Step 7: Wait for the first boot

```bash
OK=0
for i in $(seq 1 150); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId gitlab --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the deployment logs"; false; }
```

`COMPLETED` means rolled out and serving. Expect the first boot to take several minutes. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId gitlab`.

**This step needs a human:** GitLab writes a generated `root` password to `/etc/gitlab/initial_root_password`. Ask the user to run this themselves so the password never passes through the agent's output (Linux: `script -qefc '<cmd>' /dev/null`):

```bash
script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId gitlab \
  --shell-cmd 'sh -c' --cmd '"grep Password: /etc/gitlab/initial_root_password"'
```

Then sign in at `https://$HOST` as `root` and change the password right away. GitLab deletes the file on the first restart after 24 hours.

## Step 8 (optional): Git over SSH through a TCP load balancer

HTTPS clones work without this step. For `git@` SSH clones you need a team-level Layer 4 load balancer, which gives a public IP and forwards TCP to the private port 22. An API token needs the `Account > Networking > LoadBalancers` create and read permissions (check with `northflank list load-balancers -o json >/dev/null && echo LB_OK`). Write `gitlab-ssh.json` with the project's region, `europe-west` unless you chose another (backends' weights must sum to 100):

```json
{
  "name": "gitlab-ssh",
  "spec": {
    "type": "tcp", "visibility": "public",
    "target": { "type": "region", "targetId": "europe-west" },
    "ports": [ { "id": "port-1", "port": "22",
      "backends": [ { "id": "gitlab/gitlab", "type": "service", "port": 22, "weight": 100 } ] } ]
  }
}
```

```bash
northflank put load-balancer --noDefaults -o json --file gitlab-ssh.json
OK=0
for i in $(seq 1 60); do
  LB=$(northflank get load-balancer --loadBalancerId gitlab-ssh --noDefaults -o json)
  S=$(echo "$LB" | jq -r '(.data // .).state.status // "NONE"'); echo "lb=$S"
  [ "$S" = "created" ] && { OK=1; break; }
  [ "$S" = "error" ] && echo "LB_ERROR (the platform retries automatically)"
  sleep 10
done
[ "$OK" = 1 ] || { echo "LOAD_BALANCER_NOT_CREATED: stop here and check the load balancer in the UI"; false; }
[ "$OK" = 1 ] && SSH_HOST=$(echo "$LB" | jq -r '(.data // .).state.endpoints[0]') && echo "$SSH_HOST"
```

Provisioning takes a few minutes; ports 30000–32767 are reserved, and updating a load balancer re-provisions it with a short interruption. Then add `; gitlab_rails['gitlab_ssh_host'] = '<SSH_HOST>'` to `OMNIBUS` and re-run the patch in step 6 so SSH clone URLs use the load balancer address. SSH host keys live in `/etc/gitlab` on the volume, so they survive restarts.

## How to know it worked

You're done when all of these hold:

1. The sign-in page loads over HTTPS:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' "https://$HOST/users/sign_in"   # 200
   ```
2. GitLab's links use the public HTTPS address, not the container hostname:
   ```bash
   curl -s "https://$HOST/users/sign_in" | grep -o "https://$HOST" | head -n1
   ```
3. Readiness passes for the database, Redis and Gitaly. It's called from inside the container, where the allowlist permits it:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId gitlab \
     --shell-cmd 'sh -c' --cmd '"/opt/gitlab/embedded/bin/curl -s http://localhost/-/readiness?all=1"'
   ```
   Every check should report `"status":"ok"`.
4. The volume is attached:
   ```bash
   northflank get volume --projectId "$PROJECT_ID" --volumeId gitlab-data --noDefaults -o json | jq '(.data // .) | {status, attachedObjects}'
   ```
5. After the human signs in, a new project's HTTPS clone URL starts with `https://<HOST>/`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Startup probe fails and the container restarts in a loop | First boot is slower than 20 minutes, or the plan is too small | Raise `failureThreshold` on the startup probe; use at least `nf-compute-400` |
| Killed with out-of-memory errors | Under 8 GB of memory | Move to `nf-compute-800-16`, or apply the memory settings in the FAQ |
| Redirect loop, or 502 from the proxy | NGINX still listening for HTTPS | Check `listen_https = false` and `listen_port = 80` in `GITLAB_OMNIBUS_CONFIG` |
| `422` or `Can't verify CSRF token authenticity` on sign-in | `external_url` isn't the `https://` host you browse to | Re-run step 6 with the right host |
| SSH clone times out | No load balancer, or `gitlab_ssh_host` not set | Do step 8 |
| `initial_root_password` missing | Deleted on a restart after 24 hours | Reset the password with `gitlab-rake "gitlab:password:reset[root]"` in an exec session |
| Creating the service is refused because of the plan | The team can't use `nf-compute-800-16` | Pick a plan from step 2's list with at least 8 GB, and apply the FAQ's memory settings |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write: team, GitLab tag, plan, region and whether the project is new or reused.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Don't report success until checks 1 to 3 pass.
- Never print the root password, `/etc/gitlab/gitlab-secrets.json` or SMTP credentials. Let the user read the initial password themselves.
- Hand the root sign-in to the user right after the first boot; until the password is changed, anyone who reads it can take over the instance.
- Never delete the volume or recreate the service without explicit confirmation. Back up the volume (see [backup and clone volumes](https://northflank.com/docs/v1/application/databases-and-persistence/backup-and-clone-volumes)) before upgrading.
- GitLab has no mail server. Ask for SMTP settings before promising email, and keep SMTP passwords in a secret group.

## FAQ

### How much CPU and memory does GitLab need?
GitLab's requirements page sets 8 vCPU and 16 GB as the single-node baseline, with 8 GB as the floor for memory-constrained installs. For a small team on 8 GB, add `puma['worker_processes'] = 0; sidekiq['concurrency'] = 10; prometheus_monitoring['enable'] = false` to `GITLAB_OMNIBUS_CONFIG`, following GitLab's memory-constrained guide.

### How do I upgrade GitLab?
Back up the volume, then move one supported upgrade step at a time (GitLab publishes required upgrade stops). Change the tag with `northflank patch service deployment --projectId "$PROJECT_ID" --serviceId gitlab --input '{"deployment":{"instances":1,"external":{"imagePath":"gitlab/gitlab-ce:<NEW_TAG>"}}}'` and wait for `COMPLETED`. GitLab reconfigures and migrates on start.

### Where should I host GitLab?
On a host with real CPU and memory, fast persistent disk, an HTTPS URL and, for SSH, a TCP port; serverless and static hosts can't run it. A VM works if you manage TLS, patches and backups yourself; a container platform with volumes and TCP load balancers, like this guide's setup, takes those off your hands. For HA, GitLab recommends its Helm chart.

### Should I choose GitLab or Gitea?
GitLab bundles CI/CD, a registry and planning tools, and needs the resources above. For Git hosting with pull requests only, Gitea runs in a fraction of the memory.

## Conclusion

You now have GitLab CE on the official Omnibus image, with config, logs and data on a persistent volume, an `external_url` that matches the public HTTPS address, and optional SSH through a TCP load balancer.

**Next:** [link a custom domain to the port](https://northflank.com/docs/v1/application/domains/link-a-domain-to-a-port) and repeat step 6 with that hostname, configure SMTP, and register a GitLab Runner for CI jobs. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup).

## Related articles

- [How to self-host Gitea](self-host-gitea.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to self-host Plane](self-host-plane.md)
- [How to deploy Mattermost](deploy-mattermost.md)
- [How to self-host n8n](self-host-n8n.md)
