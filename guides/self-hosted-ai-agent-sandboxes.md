<!--
agent-guide-meta:
  id: NF-SBX-BYOC
  description: "How to run self-hosted AI agent sandboxes in your own cloud on Kubernetes with microVM or gVisor isolation, custom plans, your storage classes and egress rules."
  sources:
    - https://northflank.com/docs/v1/application/sandboxes/sandboxes-in-your-own-cloud
    - https://northflank.com/docs/v1/application/sandboxes/quickstart
    - https://northflank.com/docs/v1/application/sandboxes/configure-and-manage-sandboxes
    - https://northflank.com/docs/v1/application/sandboxes/examples
    - https://northflank.com/docs/v1/application/bring-your-own-cloud/use-other-cloud-providers-with-northflank
    - https://northflank.com/docs/v1/application/bring-your-own-cloud/byoc-and-byok-requirements
    - https://northflank.com/docs/v1/application/bring-your-own-cloud/aws-launch-templates
    - https://northflank.com/docs/v1/application/bring-your-own-cloud/configure-your-cluster
    - https://northflank.com/docs/v1/application/bring-your-own-cloud/deploy-and-scale-node-pools
    - https://northflank.com/docs/v1/application/bring-your-own-cloud/deploy-workloads-to-your-cluster
    - https://northflank.com/docs/v1/application/bring-your-own-cloud/create-custom-resource-plans
    - https://northflank.com/docs/v1/application/release/tag-workloads-and-resources
    - https://northflank.com/docs/v1/application/network/configure-network-policies
    - https://northflank.com/docs/v1/application/databases-and-persistence/add-a-volume
    - https://northflank.com/docs/v1/api/org/cloud-providers/get-cluster
    - https://northflank.com/docs/v1/api/team/projects/create-project
    - https://northflank.com/docs/v1/api/team/tags/put-tag
    - https://northflank.com/docs/v1/api/team/tags/list-tags
    - https://northflank.com/docs/v1/api/project/services/patch-deployment-service
  related: [NF-SBX-CODEEXEC, NF-SBX-MCP, NF-SBX-CLAUDECODE, NF-AI-003]
-->

# How to run self-hosted AI agent sandboxes in your own cloud

To run self-hosted AI agent sandboxes in your own cloud, use a Kubernetes cluster in your AWS, GCP or Azure account with node pools that support microVMs (bare metal or nested virtualization) or gVisor, install those runtimes, select a runtime class for every sandbox, and use volumes from your own storage classes. Installing the runtimes alone isolates nothing: without a runtime class, workloads run as ordinary containers.

This guide does it with Northflank's control plane on your cluster, using the CLI and Python SDK: sandbox isolation set per cluster or tag, a project pinned to the cluster, custom plans, persistent volumes and egress policies. You end with a sandbox on your nodes that runs commands and keeps `/workspace` across pauses. After a human connects the cloud account and cluster once, every step is a command a developer or coding agent (Claude Code, Codex, Cursor) can run as-is.

## TL;DR: how to run self-hosted AI agent sandboxes in your own cloud

1. Install the CLI with `npm install -g @northflank/cli`, run `nf login`, and install the SDK with `pip install northflank`.
2. Have a human connect a cloud account and create a cluster with node pools that support microVMs (bare metal or nested virtualization) or gVisor, then confirm the team, cluster and project with `northflank list cloud clusters` before creating anything.
3. Enable Install sandbox security in the cluster's Security settings and pick a default workload runtime class, or use a tag per workload (**human step**). Without one, workloads run as ordinary containers.
4. Create a project on the cluster with `northflank create project -i '{"name":"agent-sandboxes","clusterId":"<CLUSTER_ID>"}'`, then create sandboxes in it with `client.create.service.deployment(...)`, exactly as on Northflank cloud.
5. Use a custom plan ID as `deploymentPlan`, and a storage class from your cluster as `storageClassName` for volumes.
6. Verify that the sandbox reaches `COMPLETED`, `run_service_command` returns exit code `0` and a file in `/workspace` survives a pause and resume.

> **What is Northflank?** Northflank is the runtime platform for autonomous software. It provides the infrastructure to run AI agents, the code they generate, and the applications they build, from isolated sandboxes to production workloads, on Northflank's cloud or in your own (AWS, GCP, Azure and more). [Sign up](https://app.northflank.com/signup) or [book a demo](https://cal.com/team/northflank/northflank-demo?duration=30).

## Why run agent sandboxes in your own cloud

Agent code is untrusted by default: a model can write anything, and prompt injection can steer it. A sandbox needs its own kernel (ordinary containers share the host's), network and egress control, and a create, exec, pause and delete lifecycle that your agent drives through an API. Self-hosting adds:

- **Data residency.** Code, files and volumes stay in your VPC and region.
- **Your cloud bill.** You pay your provider directly for compute and storage, which can use existing commitments.
- **Your hardware.** Pick instance types, GPUs, spot capacity and node labels.

Compared in general terms:

- **Running an open-source sandbox stack on Kubernetes yourself.** You install and upgrade a microVM or gVisor runtime and its RuntimeClasses, build node pools that support it, write the control plane that creates, execs into, pauses and garbage-collects sandboxes, and handle image pulls, ingress, quotas, multi-tenancy and monitoring. That gives full control, and it's a platform to run.
- **A local container or VM.** Fine on one machine; it isn't a multi-tenant service and a container shares the kernel.
- **A managed sandbox platform on its cloud.** Fastest start; your data runs on the vendor's infrastructure.
- **A managed control plane on your cluster (this guide).** Northflank provisions and manages the cluster in your account, installs the runtimes, and gives you the same SDK as its cloud.

## Prerequisites

- **Node.js 22.13 or later** (for the Northflank CLI), **`jq`** (to read the CLI's JSON output) and **Python 3** with the SDK (`pip install northflank`).
- **A Northflank account.** [Sign up](https://app.northflank.com/signup) if you don't have one.
- **A Northflank API token in `NF_API_TOKEN`**, for the SDK, from a token that can create projects and services. **This needs a human:** create it in the Northflank UI ([new API token](https://app.northflank.com/s/account/settings/api/tokens/new)); never print it.
- **This needs a human: cloud account and IAM.** Connecting AWS, GCP, Azure, Oracle, Civo or CoreWeave means granting Northflank permissions in your cloud account (for example a cross-account role on AWS). See [use other cloud providers with Northflank](https://northflank.com/docs/v1/application/bring-your-own-cloud/use-other-cloud-providers-with-northflank).
- **This needs a human: the cluster.** Create the cluster and its node pools in the Northflank UI, sized to at least the [BYOC minimum](https://northflank.com/docs/v1/application/bring-your-own-cloud/byoc-and-byok-requirements): 1 node, 4 vCPU and 8 GB per node, 12 vCPU and 24 GB per cluster, with about 100 GB ephemeral storage per node recommended.

## Step 1: Install the Northflank CLI and log in

```bash
npm install -g @northflank/cli@latest
nf login
set -o pipefail
northflank get auth -o json | jq -r '(.data // .).entityId'
```

**This step needs a human:** `nf login` opens a browser, where you sign in and pick the team to deploy into. The last command prints that team's ID, so you know the CLI is logged in to the right place. `set -o pipefail` makes a failed `northflank` command show up as a failure even when its output is piped into `jq`; keep it on for the rest of this guide. On a machine without a browser, such as CI, create a team API token in the Northflank UI and run `nf login --name ci --token "$NF_API_TOKEN"` instead.

If you're a coding agent (Claude Code, Codex, Cursor or OpenCode), also run `nf agent setup --agent claude --yes`, replacing `claude` with your agent. It installs the Northflank skill and MCP server, which load in your next session. This guide works without them.

## Step 2: Confirm the cluster and where the sandboxes will run

Before creating anything, check that the SDK token is set (without printing it), and print the team, your clusters and whether the project exists:

```bash
echo "team:    $(northflank get auth -o json | jq -r '(.data // .).entityId')"
[ -n "${NF_API_TOKEN:-}" ] && echo "NF_API_TOKEN: set" || echo "NF_API_TOKEN: missing"
northflank list cloud clusters -o json | jq -r '(.data // .).clusters[] | "cluster: \(.id)\t\(.provider)\t\(.region)"'
if P=$(northflank get project --projectId agent-sandboxes -o json 2>/dev/null); then
  echo "project: agent-sandboxes exists on cluster $(echo "$P" | jq -r '(.data // .).cluster.id // "unknown"') and will be reused"
else
  echo "project: agent-sandboxes will be created"
fi
```

- If no cluster is listed, stop: a human connects the cloud account and creates the cluster first (see the prerequisites).
- If `NF_API_TOKEN` says `missing`, ask the user to export it in this shell. The Python SDK authenticates with it, not with your `nf login` session, so it must belong to the team printed above.
- If the project exists on a different cluster, or on Northflank cloud, it can't be moved: use a new project name throughout.

**If you're an agent:** show the user the team, the cluster ID you'll use and whether the project is new or reused. Wait for them to confirm before you create anything.

## Step 3: Choose node pools that can run the runtime

From [sandboxes in your own cloud](https://northflank.com/docs/v1/application/sandboxes/sandboxes-in-your-own-cloud):

- **MicroVM (Kata Containers)** needs bare-metal nodes or nodes with nested virtualization. On AWS that means bare-metal instances or eligible non-metal instances with nested virtualization enabled through a [launch template](https://northflank.com/docs/v1/application/bring-your-own-cloud/aws-launch-templates). Availability varies by provider and region.
- **gVisor** runs on nodes where microVMs aren't supported, so a cluster can mix both.

To keep sandboxes on dedicated nodes, add a label such as `pool: sandbox` when you create the node pool (labels can't be changed later), then use a tag with a node affinity rule (step 5). See [deploy and scale node pools](https://northflank.com/docs/v1/application/bring-your-own-cloud/deploy-and-scale-node-pools).

Check the state of the cluster you picked in step 2:

```bash
northflank get cloud cluster --clusterId <CLUSTER_ID> -o json | jq '(.data // .) | {state: .status.state.state, sandboxing: .settings.infrastructure}'
```

Wait for `running` before going on.

## Step 4: Turn on sandbox isolation for the cluster

**This step needs a human** (or an agent with the user's explicit go-ahead in the UI): open the cluster, go to Security, then Sandboxing, and enable **Install sandbox security**. That installs both the microVM (Kata Containers) and gVisor runtimes. Then choose:

- **Workload runtime class**: applies to services, jobs and addons, so every sandbox in projects on this cluster gets it.
- **Build runtime class**: applies to builds.

The API accepts the runtime classes `none`, `gvisor`, `kata-clh` and `kata-qemu`. Read the result back with the `get cloud cluster` command from step 3.

**Isolation isn't automatic.** Installing the runtimes sandboxes nothing by itself. A workload is sandboxed only when a runtime class selects one, from the cluster default or from a tag. With the runtime class set to `none`, workloads run as ordinary containers on sandbox-capable nodes. If other, trusted workloads share the cluster, leave the default at `none` and use a tag (step 5) for sandboxes only.

## Step 5 (optional): Select the runtime per workload with a tag

Tags carry a `sandboxing` block per workload type. Save as `tag.json`:

```json
{
  "name": "agent-sandbox",
  "description": "Run in a microVM",
  "useAsInfrastructureLabel": true,
  "sandboxing": { "services": { "enabled": true, "runtimeClass": "kata-clh" } }
}
```

```bash
northflank put tag --file tag.json -o json >/dev/null
northflank list tags -o json | jq -r '(.data // .).tags[] | select(.name == "agent-sandbox") | .id'
```

Use `gvisor` instead of `kata-clh` on nodes without virtualization support. `useAsInfrastructureLabel` lets [network policies](https://northflank.com/docs/v1/application/network/configure-network-policies) select tagged workloads. Add node affinity rules matching your `pool: sandbox` label to the tag, either in the UI or as `nodeAffinities` in `tag.json` ([tag workloads and resources](https://northflank.com/docs/v1/application/release/tag-workloads-and-resources)). A mandatory rule that matches no node pool means the workload never schedules.

## Step 6: Create a project on your cluster

```bash
northflank create project -i '{"name":"agent-sandboxes","clusterId":"<CLUSTER_ID>"}' -o json | jq -r '(.data // .).id'
```

Skip this step if step 2 said the project will be reused. `create` fails if the name is taken, so it never changes a project you didn't mean to touch. A BYOC project takes `clusterId` instead of `region`, and its cluster can't be changed after creation. Every service, job and volume in it runs on your cluster.

## Step 7: Create a sandbox and run a command

The create call is the same as on Northflank cloud. The payload has no isolation setting: the runtime comes from the cluster default or the tag.

```python
import os, uuid
from northflank import ApiClient

client = ApiClient(api_token=os.environ["NF_API_TOKEN"])
project_id = "agent-sandboxes"  # the project from step 6
sandbox_id = f"sandbox-{uuid.uuid4().hex[:8]}"

client.create.service.deployment(
    project_id=project_id,
    data={
        "name": sandbox_id,
        "tags": ["agent-sandbox"],  # omit if the cluster default selects the runtime
        "billing": {"deploymentPlan": "nf-compute-200"},
        "deployment": {
            "instances": 1,
            "docker": {
                "configType": "customEntrypointCustomCommand",
                "customEntrypoint": "/bin/bash",
                "customCommand": "-c 'sleep infinity'",
            },
            "external": {"imagePath": "ubuntu:22.04"},
            "storage": {"ephemeralStorage": {"storageSize": 2048}},
        },
    },
)

client.helpers.wait_for_service_ready(project_id=project_id, service_id=sandbox_id, timeout_s=300, poll_interval_s=1)

result = client.exec.run_service_command(
    project_id=project_id, service_id=sandbox_id,
    command="uname -r && cat /etc/os-release | head -2", shell="/bin/bash -c",
)
print(result.exit_code, result.stdout, result.stderr)
```

`wait_for_service_ready` raises `RuntimeError` if the deployment reports `FAILED`, which on your own cluster usually means no node pool satisfies the runtime or affinity rules. Pause, resume, logs, metrics, file copy and delete work as on Northflank cloud; see [configure and manage sandboxes](https://northflank.com/docs/v1/application/sandboxes/configure-and-manage-sandboxes). GPU sandboxes use the same payload as the [GPU example](https://northflank.com/docs/v1/application/sandboxes/examples) on clusters with GPU node pools.

## Step 8: Size sandboxes with a custom resource plan

On your own cloud, you aren't limited to the standard `nf-compute-*` plans. **This step needs a human:** create a [custom resource plan](https://northflank.com/docs/v1/application/bring-your-own-cloud/create-custom-resource-plans) in the UI, choosing:

- **Northflank configuration**: set CPU and memory limits; requests follow the cluster's [request modifiers](https://northflank.com/docs/v1/application/bring-your-own-cloud/configure-your-cluster).
- **Kubernetes configuration**: set requests and limits yourself. Avoid unbounded CPU for agent sandboxes; many busy sandboxes could exhaust a node pool.

Size plans to fit your node types so you don't strand capacity. Use the plan's ID as `billing.deploymentPlan` when creating a sandbox, or move an existing one with `client.patch.service.deployment(project_id=..., service_id=..., data={"billing": {"deploymentPlan": "<CUSTOM_PLAN_ID>"}})`, which restarts the sandbox and loses ephemeral files.

## Step 9: Add a persistent volume from your storage class

Volumes on your cluster use your cluster's storage classes, not Northflank cloud's. List them with their access modes:

```bash
northflank get cloud cluster --clusterId <CLUSTER_ID> -o json \
  | jq '(.data // .).storage.storageClasses[] | {name, kubernetesName, type, accessModes: .options.accessModes}'
```

Create the volume, then attach it when you create the sandbox:

```python
volume_name = f"data-{sandbox_id}"
volume = client.create.volume(
    project_id=project_id,
    data={
        "name": volume_name,
        "mounts": [{"containerMountPath": "/workspace"}],
        "spec": {
            # ReadWriteMany needs a storage class that supports shared access.
            "accessMode": "ReadWriteOnce",
            "storageClassName": "your-storage-class",
            "storageSize": 10240,
        },
    },
)
volume_id = (volume.data or {}).get("id") or volume_name
# then add to the create.service.deployment payload:
#   "createOptions": {"volumesToAttach": [volume_id]}
```

Files in `/workspace` then survive pause and resume. Your cloud provider bills the storage, including while the sandbox is paused. A volume outlives its sandbox: detach it, delete the service, then delete the volume ([manage sandboxes](https://northflank.com/docs/v1/application/sandboxes/configure-and-manage-sandboxes)). By default, deleting the cluster also deletes its volumes.

## Step 10: Restrict sandbox egress

On your own cluster you can add [network policies](https://northflank.com/docs/v1/application/network/configure-network-policies) scoped to the sandbox project or the `agent-sandbox` tag. An egress rule set to "Deny all traffic", or an allow-list of hostnames, CIDRs or projects, keeps agent code from reaching your internal services. Once a policy has an allow-list, anything not listed is blocked, so test it on a staging project first.

## How to know it worked

1. `get cloud cluster` shows state `running` and sandboxing settings with a runtime class other than `none` (or your tag exists).
2. The project points at your cluster: `northflank get project --projectId agent-sandboxes -o json | jq '(.data // .).cluster'`.
3. The sandbox reaches `COMPLETED` and `run_service_command` returns exit code `0`.
4. As a quick check, `uname -r` in the sandbox differs from the node's kernel version.
5. A file written to `/workspace` survives a pause and resume.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Sandbox stuck in `PENDING` or `FAILED` | No node pool supports the runtime, or affinity rules match nothing | Add bare-metal / nested-virtualization nodes, or use `gvisor`; check tag rules |
| Sandbox runs, but as a normal container | Runtime class is `none` and no tag selects one | Set the workload runtime class or apply the tag |
| Volume never binds | Storage class doesn't exist or has no running provisioner | Use a class from `storage.storageClasses` |
| `ReadWriteMany` rejected | Class doesn't support shared access | Use `ReadWriteOnce` |
| Nodes run out of capacity | Plans larger than nodes, or unbounded CPU | Resize the custom plan or scale the node pool |
| An SDK call returns 401 or 403 | `NF_API_TOKEN` missing, expired, or from another team | Ask a human for a team API token for the team from step 2, export it and retry |
| `northflank get auth` fails, or a command returns 401 | Not logged in, or the session expired | Run `nf login` again |
| A command is refused while `get auth` works | The API token's role lacks that permission (each command's `--help` lists it) | Use a token with that permission, or log in with `nf login` |

## A note for AI coding agents

- Confirm the target from step 2 with the user before any write: team, cluster, and whether the project is new or reused.
- If `wait_for_service_ready` raises, read the sandbox logs (`northflank get service logs --projectId agent-sandboxes --serviceId <SANDBOX_ID>`) before changing anything.
- Don't create cloud integrations, IAM roles or clusters. Stop and ask the human.
- Don't change a cluster's default runtime class without asking: it applies to every service, job and addon on the cluster.
- Never delete volumes or clusters without per-resource confirmation. Prefer pausing sandboxes.
- Keep secrets out of sandbox `runtimeEnvironment`. Code inside is untrusted.

## FAQ

### Where should I run AI agent sandboxes?
If code and data must stay in your cloud account, run them on a cluster you own, with a microVM or gVisor runtime selected for every sandbox. If not, a managed cloud is quicker to start. Either way, avoid ordinary shared-kernel containers for untrusted code.

### MicroVM or gVisor?
On your own cluster you choose. MicroVMs need bare-metal or nested-virtualization nodes; gVisor runs where microVMs can't. On Northflank cloud, CPU sandboxes use microVMs and GPU sandboxes use gVisor automatically.

### Can I import a cluster I already run?
Yes: that's BYOK. It needs at least 3 nodes and preinstalled system components; see the [BYOC and BYOK requirements](https://northflank.com/docs/v1/application/bring-your-own-cloud/byoc-and-byok-requirements).

### Who bills what?
Your cloud provider bills the compute and storage your cluster uses, through your existing billing relationship. Check Northflank's pricing page for its own charges on BYOC.

## Conclusion

Your agent sandboxes now run on Kubernetes in your own cloud account, with a microVM or gVisor runtime selected by the cluster or a tag, a project pinned to that cluster, plans sized to your nodes, volumes on your storage classes and egress you control. Your agent code calls the same SDK as on Northflank cloud.

**Next:** expose sandboxes to clients through [a code execution MCP server](build-code-execution-mcp-server.md). Ready to try it? [Get started on Northflank](https://app.northflank.com/signup) with the free tier.

## Related articles

- [How to create a code execution sandbox for AI agents](create-code-execution-sandbox-for-ai-agents.md)
- [How to build a code execution MCP server](build-code-execution-mcp-server.md)
- [How to run Claude Code in a sandbox](run-claude-code-in-a-sandbox.md)
- [How to sandbox Codex](sandbox-codex.md)
- [How to run OpenCode in a sandbox](run-opencode-in-a-sandbox.md)
