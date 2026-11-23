<!--
agent-guide-meta:
  id: NF-STK-PROMETHEUS
  description: "How to deploy Prometheus with the official Docker image, prometheus.yml mounted as a file, a persistent TSDB volume and private scraping of your services."
  sources:
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://prometheus.io/docs/prometheus/latest/installation/
    - https://prometheus.io/docs/prometheus/latest/configuration/configuration/
    - https://prometheus.io/docs/prometheus/latest/storage/
    - https://prometheus.io/docs/prometheus/latest/management_api/
    - https://prometheus.io/docs/prometheus/latest/querying/api/
    - https://github.com/prometheus/prometheus/blob/main/Dockerfile
    - https://hub.docker.com/r/prom/prometheus/tags
    - https://northflank.com/docs/v1/api/project/services/put-deployment-service
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
    - https://northflank.com/docs/v1/api/project/volumes/create-volume
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/application/secure/upload-secret-files
    - https://northflank.com/docs/v1/application/network/configure-ports
    - https://northflank.com/docs/v1/application/network/add-security-policies-for-ports
    - https://northflank.com/docs/v1/application/network/enable-multi-project-networking
    - https://northflank.com/docs/v1/application/run/override-command-entrypoint
  related: [NF-STK-GRAFANA, NF-STK-KEYCLOAK, NF-DEP-002, NF-DB-001]
-->

# How to deploy Prometheus

To deploy Prometheus in production, run the official `prom/prometheus` image with your own `prometheus.yml` mounted at `/etc/prometheus/prometheus.yml`, keep the TSDB in `/prometheus` on a persistent volume with retention limits that fit it, scrape your services over a private network, and keep the UI and API off the public internet. Prometheus has no login of its own, so a public port exposes every metric.

This guide does it on Northflank with the CLI: a deployment service running the official image with `prometheus.yml` passed as a runtime file, a volume for the TSDB and a private port that only workloads in the project can reach. You end with Prometheus scraping your services by their internal names, verified through `/-/ready` and the targets API. Every step is a command a developer or a coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to deploy Prometheus

1. Install the CLI with `npm install -g @northflank/cli` and run `nf login`.
2. Confirm the team and project, list the services to scrape and pin a Prometheus version before changing anything.
3. Write `prometheus.yml` with scrape jobs that use internal addresses such as `api:8080`.
4. Run `prom/prometheus:<version>` on a private port 9090 with `northflank put service deployment`, with the config as a runtime file, and attach a volume at `/prometheus`.
5. Wait for the rollout, then check that the config is valid, `/-/ready` answers and every target in `/api/v1/targets` is `up`.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why this setup?

The Prometheus image is built for this: its Dockerfile runs `/bin/prometheus` as `nobody` with `--config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/prometheus` and declares `/prometheus` as a volume. The installation docs recommend a volume for production; without one, the data in `/prometheus` is lost whenever the container is replaced. Three choices follow:

- **The config as a mounted file.** No custom image to build; change the file and restart.
- **A volume with retention.** The storage docs say retention defaults to `15d` and that `--storage.tsdb.retention.size` drops the oldest blocks once a size cap is hit. Setting the cap below the volume size stops the disk filling up. NFS-style filesystems aren't supported, and a block volume is what's used here.
- **Private by default.** Prometheus pulls metrics, so it needs to reach your services, but nobody needs to reach it from the internet. Grafana in the same project queries it at `http://prometheus:9090`.

So where to deploy Prometheus? Next to what it scrapes. It needs a long-running process, a disk that survives restarts and a private network path to every target. Serverless hosts fail all three. A VM works if your services are on the same network. Kubernetes with the Prometheus Operator is the heavy option. A container platform that gives each service a private DNS name lets one container scrape everything in the project.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI) and **`jq`** (to read the CLI's JSON output).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **`openssl` and `base64`**, to encode the config and, optionally, generate a password.
- **Services that serve Prometheus metrics**, usually on `/metrics`, in the project Prometheus will run in.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Confirm where it will deploy

Prometheus scrapes services in its own project over the private network, so deploy it into the project that runs the services you want to monitor. Replace `monitoring` with that project's ID if it's different:

```bash
export PROJECT_ID=monitoring
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
northflank get project --projectId "$PROJECT_ID" -o json >/dev/null 2>&1 \
  && { echo "project: $PROJECT_ID exists and will be reused"; \
       northflank list services --projectId "$PROJECT_ID" -o json | jq -r '"services: " + ([(.data // .).services[].id] | join(" "))'; } \
  || echo "project: $PROJECT_ID will be created"
PROM_VERSION=$(curl -s "https://hub.docker.com/v2/repositories/prom/prometheus/tags?page_size=100&ordering=last_updated" \
  | jq -r '.results[].name' | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$' | sort -V | tail -n1); echo "prometheus: $PROM_VERSION"
```

- Each scrape target is `<service-id>:<port>`, using the IDs in the `services:` line. If the project will be created, it has no services yet; Prometheus still scrapes itself.
- The version line picks the newest plain version tag from Docker Hub (3.x at the time of writing). If it prints nothing, set `PROM_VERSION` by hand.

Names used below: service `prometheus`, volume `prometheus-data`.

**If you're an agent:** show the user the team, the project, the region (`europe-west` below, unless they choose another), whether the project is new or reused, the scrape targets and the Prometheus version. Wait for them to confirm before you create anything.

## Step 3: Create the project

A project holds your services. Pick the region closest to your users (`northflank list regions` lists them); it can't be changed later. Skip the `create` line if step 2 said the project will be reused. If you chose another project name in step 2, use it here. `create` fails if the name is taken, so it never changes a project you didn't mean to touch.

```bash
northflank create project -o json --input '{ "name": "monitoring", "region": "europe-west" }'
export PROJECT_ID=monitoring
```

## Step 4: Write `prometheus.yml`

Save this as `prometheus.yml`. Replace the `app` job with your services. Don't write `${…}` anywhere in this file, because Northflank substitutes those templates in secret files.

```yaml
global:
  scrape_interval: 30s
  evaluation_interval: 30s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  # Grafana serves /metrics on its HTTP port by default
  - job_name: grafana
    static_configs:
      - targets: ["grafana:3000"]

  # Your own service: internal name and port, no public URL needed
  - job_name: app
    metrics_path: /metrics
    static_configs:
      - targets: ["api:8080"]
```

Notes on targets:

- `api:8080` is the private address of the service `api` on port 8080. The [configure ports](https://northflank.com/docs/v1/application/network/configure-ports) docs give the format `[service-name]:[port-number]`, for services in the same project.
- Internal traffic to a service is load-balanced round-robin across its replicas, so with several instances each scrape hits a random one. Scrape services that run one instance, or expose metrics in a form that's correct per request (for example, through a shared exporter).
- Services in other projects need [multi-project networking](https://northflank.com/docs/v1/application/network/enable-multi-project-networking).

## Step 5: Deploy the Prometheus service

The custom command replaces the image's CMD and keeps its `/bin/prometheus` entrypoint. The installation docs warn that adding flags overrides the defaults, so the config and storage flags are repeated:

```bash
CFG_B64=$(base64 < prometheus.yml | tr -d '\n')
jq -n --arg img "prom/prometheus:$PROM_VERSION" --arg cfg "$CFG_B64" '{
  name: "prometheus",
  billing: { deploymentPlan: "nf-compute-50" },
  deployment: {
    instances: 1,
    external: { imagePath: $img },
    docker: { configType: "customCommand",
      customCommand: "--config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/prometheus --storage.tsdb.retention.time=15d --storage.tsdb.retention.size=8GB" }
  },
  ports: [ { name: "http", internalPort: 9090, public: false, protocol: "HTTP" } ],
  runtimeFiles: { "/etc/prometheus/prometheus.yml": { data: $cfg, encoding: "utf-8" } },
  healthChecks: [
    { protocol: "HTTP", type: "readinessProbe", path: "/-/ready", port: 9090,
      initialDelaySeconds: 5, periodSeconds: 10, timeoutSeconds: 3, failureThreshold: 30, successThreshold: 1 },
    { protocol: "HTTP", type: "livenessProbe", path: "/-/healthy", port: 9090,
      initialDelaySeconds: 30, periodSeconds: 20, timeoutSeconds: 3, failureThreshold: 3 }
  ]
}' > prometheus.json
northflank put service deployment --projectId "$PROJECT_ID" --noDefaults -o json --file prometheus.json
```

`public: false` makes the port reachable only from inside the project. `/-/ready` returns 200 once Prometheus can serve queries, which after a restart includes replaying the write-ahead log, hence the generous failure threshold. `nf-compute-50` (0.5 vCPU, 1 GB) suits a few dozen targets; memory grows with the number of active series.

## Step 6: Attach a volume for the TSDB

```bash
northflank get volume --projectId "$PROJECT_ID" --volumeId prometheus-data --noDefaults -o json >/dev/null 2>&1 || \
northflank create volume --projectId "$PROJECT_ID" --noDefaults -o json --input '{
  "name": "prometheus-data",
  "mounts": [ { "containerMountPath": "/prometheus" } ],
  "spec": { "accessMode": "ReadWriteOnce", "storageSize": 10240 },
  "attachedObjects": [ { "id": "prometheus", "type": "service" } ]
}'
```

10 GB with an 8 GB retention cap leaves room for the WAL and compaction. Volumes can grow but never shrink. Attaching redeploys the service, and a single read/write volume limits it to one instance.

## Step 7: Wait for the rollout

```bash
OK=0
for i in $(seq 1 60); do
  D=$(northflank get service --projectId "$PROJECT_ID" --serviceId prometheus --noDefaults -o json | jq -r '(.data // .).status.deployment.status // "NONE"')
  echo "deployment=$D"
  [ "$D" = "COMPLETED" ] && { OK=1; break; }
  [ "$D" = "FAILED" ] && { echo DEPLOY_FAILED; break; }
  sleep 10
done
[ "$OK" = 1 ] || { echo "ROLLOUT_NOT_COMPLETE: stop here and read the build or deployment logs"; false; }
```

`COMPLETED` means rolled out and passing readiness. If it prints `ROLLOUT_NOT_COMPLETE`, stop and read the logs: `northflank get service logs --projectId "$PROJECT_ID" --serviceId prometheus`.

## Step 8: Reach the UI privately

For a person, port-forward from their machine. This runs in the foreground until stopped, and prints the local address to use:

```bash
northflank forward service --projectId "$PROJECT_ID" --serviceId prometheus
```

With hostnames configured, open `http://prometheus:9090`. With `--skipHostnames`, use the loopback address it prints.

### Optional: expose it with basic auth

If something outside the project must query Prometheus (a hosted Grafana, for example), make the port public with a Northflank basic-auth rule rather than leaving it open. The patch sends the full `ports` list:

```bash
PROM_UI_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=')
northflank patch service deployment --projectId "$PROJECT_ID" --serviceId prometheus --noDefaults -o json --input "$(jq -n --arg pw "$PROM_UI_PASSWORD" '{
  ports: [ { name: "http", internalPort: 9090, public: true, protocol: "HTTP",
    security: { credentials: [ { type: "basic-auth", username: "prometheus", password: $pw } ] } } ]
}')"
```

**Tell the user to store `PROM_UI_PASSWORD`.** It isn't documented whether the rule also applies to internal traffic; after this change, check that anything in the project that queries `prometheus:9090` (Grafana's data source health check, for example) still works, and if it gets 401, add basic auth to that client. IP allow-lists are also available; see [add security policies for ports](https://northflank.com/docs/v1/application/network/add-security-policies-for-ports).

## How to know it worked

You're done when all of these hold. `exec` needs `script` without a TTY (Linux: `script -qefc '<cmd>' /dev/null`); the image is busybox-based, so it has `wget`, not `curl`.

1. The config is valid:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId prometheus \
     --shell-cmd 'sh -c' --cmd '"promtool check config /etc/prometheus/prometheus.yml"'
   ```
   Expect `SUCCESS`.
2. Prometheus is ready:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId prometheus \
     --shell-cmd 'sh -c' --cmd '"wget -qO- http://localhost:9090/-/ready"'
   ```
   Expect `Prometheus Server is Ready.`
3. Every target is up:
   ```bash
   script -q /dev/null northflank exec service --projectId "$PROJECT_ID" --serviceId prometheus \
     --shell-cmd 'sh -c' --cmd '"wget -qO- http://localhost:9090/api/v1/targets"' \
     | tr -d '\r' | sed -n '/^{/,$p' | jq -r '.data.activeTargets[] | "\(.labels.job) \(.scrapeUrl) \(.health) \(.lastError)"'
   ```
   Each line should end in `up` with an empty error.
4. Data lands on the volume and survives a restart: run `northflank restart service --projectId "$PROJECT_ID" --serviceId prometheus`, wait for `COMPLETED`, then query `up` and check older samples are still there with `wget -qO- 'http://localhost:9090/api/v1/query?query=count_over_time(up[1h])'`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `opening storage failed … permission denied` on `/prometheus` | Volume not writable by `nobody` | Report it to the user; the fallback is a small image `FROM prom/prometheus` built from Git that sets the user and group, as the [add a volume](https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume) docs suggest |
| Logs show the sample config's jobs only | Runtime file not mounted at the config path | Check `runtimeFiles` with `northflank get service`; redo step 5 |
| `parsing YAML file … failed` | YAML error | Run `promtool check config` locally before deploying |
| Target `down` with `no such host` | Wrong service ID, or the service is in another project | Use the ID from `list services`; enable multi-project networking |
| Target `down` with `server returned HTTP status 404` | Wrong `metrics_path` or port | Point at the port that serves `/metrics` |
| Disk full | Retention size above volume size | Lower `--storage.tsdb.retention.size` or grow the volume |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

If you're an agent running these steps for a user:

- Confirm the target from step 2 with the user before any write.
- Don't report success until checks 1 to 3 pass: a running container doesn't prove the targets are scraped.
- If the step 7 loop prints `ROLLOUT_NOT_COMPLETE`, read the logs before changing anything.
- Don't make the port public without a basic-auth or IP rule. Prometheus has no login, and metrics labels often reveal internal hostnames and tenant names.
- Don't put credentials in `prometheus.yml` as plain text in the repo; if a target needs a token, ask the user how they want it stored.
- To change the config, re-run step 5 with the new file, then `northflank restart service`. There's no need for `--web.enable-lifecycle`.
- Get explicit confirmation before deleting the volume; that's the whole metrics history.

## FAQ

### Where should I deploy Prometheus?
In the same private network as the services it scrapes, on a long-running container with a persistent disk. A container platform where each service has a private DNS name lets Prometheus scrape them without any public endpoints. A VM works too if it can reach every target.

### Should Prometheus be public?
No. Keep it private and reach it through Grafana or a port-forward. If something external must query it, put basic auth or an IP allow-list on the port.

### How do I add Grafana?
Follow [how to deploy Grafana](deploy-grafana.md) in the same project; its provisioned data source points at `http://prometheus:9090`.

### Can I scale Prometheus to more instances?
Not with one read/write volume. For high availability, run a second, independent Prometheus with its own volume and the same config, or send data to long-term storage with `remote_write`.

## Conclusion

You now have Prometheus on the official image with a mounted config, a persistent TSDB with retention limits, private scraping of your services by their internal names, and no public exposure.

**Next:** add dashboards with [how to deploy Grafana](deploy-grafana.md), and add alerting rules to `prometheus.yml`. Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to deploy Grafana](deploy-grafana.md)
- [How to deploy Keycloak](deploy-keycloak.md)
- [How to deploy a Docker image](deploy-docker-image.md)
- [How to deploy a PostgreSQL database](deploy-postgresql-database.md)
