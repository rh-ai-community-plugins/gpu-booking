# Cluster Prerequisites

The GPU Booking plugin manages GPU quota through **Kueue** and discovers GPU resources from the
cluster. Before installing the plugin, the cluster needs a small set of building blocks in place.
This page is the full reference; for a quick summary see the
[deployment guide](../deployment/OPENSHIFT_DEPLOY.md#prerequisites).

## Install order

The components build on each other, so install them in this order (a GitOps deployment can
express this with ArgoCD sync-waves):

| Order | Component | Why it is needed |
|---|---|---|
| 1 | Red Hat OpenShift AI (or ODH) | Provides the dashboard that hosts the plugin and the `HardwareProfile` API |
| 2 | Kueue operator | Provides the job queueing and quota model the plugin reserves with |
| 3 | Node Feature Discovery | Labels nodes with hardware capabilities (GPU detection) |
| 4 | GPU capacity | NVIDIA GPU Operator + MIG configuration (or a fake-GPU operator in labs) |
| 5 | User namespaces + labels | Namespaces users book into and submit workloads from |
| 6 | Kueue queues (optional) | A `default` ClusterQueue/LocalQueue for unreserved work |
| 7 | **GPU Booking plugin** | Last — it consumes everything above |

---

## 1. OpenShift AI

Install the RHOAI operator and create the `DataScienceCluster` (DSC). The examples below are from
a production deployment running RHOAI **3.5** on OpenShift 4.x:

- Operator: `rhods-operator`, channel `stable-3.5`, namespace `redhat-ods-operator`
- `DSCInitialization` with `applicationsNamespace: redhat-ods-applications`
- `DataScienceCluster` named `default-dsc`

Two DSC settings matter for the plugin:

**Kueue management state.** Set `kueue.managementState: Unmanaged` so the standalone Kueue
operator (not the RHOAI operator) owns ClusterQueue/LocalQueue configuration. The plugin creates
and deletes queues dynamically, which is simplest when nothing else reconciles them:

```yaml
apiVersion: datasciencecluster.opendatahub.io/v1
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  kueue:
    defaultClusterQueueName: default
    defaultLocalQueueName: default
    managementState: Unmanaged
```

> **Managed vs Unmanaged:** with `Managed`, the RHOAI operator reconciles Kueue configuration and
> may fight the plugin's dynamic queues. With `Unmanaged`, the standalone Kueue operator owns the
> Kueue CR and everything else (GitOps, the plugin) manages queue objects directly. Both the
> production reference deployment and the GitOps automation example use `Unmanaged`.

**Dashboard Kueue integration.** Leave the dashboard's Kueue UI enabled so users can see
workloads and queues alongside the booking plugin:

```yaml
# odh-dashboard ConfigMap/CR — do NOT set disableKueue
disableKueue: false
```

> **ODH vs RHOAI:** on the upstream Open Data Hub distribution substitute the `opendatahub`
> namespace and `odh-dashboard` names throughout.

## 2. Kueue operator

Install the Kueue operator from the Red Hat operator catalog with OLM. The production reference
pins channel `stable-v1.4`:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kueue-operator
  namespace: openshift-kueue-operator
spec:
  channel: stable-v1.4
  installPlanApproval: Automatic
  name: kueue-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

The namespace needs an `OperatorGroup` with `global: true` (cluster-scoped watches). Once the
operator is running, create the `Kueue` instance to enable the workload framework integrations
the booking model relies on:

```yaml
apiVersion: kueue.openshift.io/v1
kind: Kueue
metadata:
  name: cluster
spec:
  managementState: Managed
  logLevel: Normal
  operatorLogLevel: Normal
  config:
    integrations:
      frameworks:
        - Pod
        - PyTorchJob
        - RayCluster
        - RayJob
        - StatefulSet
        - TrainJob
```

For distributed workloads the production deployment also installs
**leader-worker-set** (`stable-v1.0`) and **job-set** (`stable-v1.0`) operators, but they are
not required by the plugin itself.

## 3. Node Feature Discovery

NFD labels nodes with their hardware capabilities. Install the operator
(namespace `openshift-nfd`, channel `stable`) and create the instance:

```yaml
apiVersion: nfd.openshift.io/v1
kind: NodeFeatureDiscovery
metadata:
  name: nfd-instance
  namespace: openshift-nfd
spec:
  operand:
    image: registry.redhat.io/openshift4/ose-node-feature-discovery:v4.19
  workerConfig:
    configData: |
      sources:
        pci:
          deviceClassWhitelist:
            - "0200"
            - "03"
            - "12"
```

This produces the `nvidia.com/gpu.present=true` label the GPU operator and the plugin's
discovery rely on.

## 4. GPU capacity

### Real GPUs (NVIDIA GPU Operator)

Install the NVIDIA GPU operator from `certified-operators` (the production reference runs
channel `v26.7`, namespace `nvidia-gpu-operator`) and a `ClusterPolicy` with MIG enabled:

```yaml
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  migManager:
    enabled: true
    config:
      name: default-mig-parted-config
      default: all-disabled   # per-node label switches the MIG config
  mig:
    strategy: mixed
  driver:
    enabled: true
    kernelModuleType: auto
  toolkit:
    enabled: true
  dcgmExporter:
    enabled: true
  gfd:
    enabled: true
```

GPU nodes are typically a dedicated machine set with a GPU taint and a role label:

```yaml
# MachineSet excerpt
spec:
  replicas: 2
  template:
    spec:
      metadata:
        labels:
          node-role.kubernetes.io/gpu: ""
      taints:
        - key: nvidia.com/gpu
          value: h200
          effect: PreferNoSchedule
```

Two properties of the node matter downstream:

- **Node labels** are how Kueue `ResourceFlavors` target the pool (e.g.
  `node.kubernetes.io/instance-type: gx3d-160x1792x8h200`). Pick a label scheme and stay
  consistent — see [Kueue & Quota Model](KUEUE_QUOTA_MODEL.md#resourceflavors).
- **MIG slices** appear as allocatable resources (`nvidia.com/mig-1g.18gb`,
  `nvidia.com/mig-2g.35gb`, `nvidia.com/mig-3g.71gb`, ...). The plugin discovers whatever the
  nodes advertise and turns each type into a bookable resource.

### Lab / development clusters (fake GPU operator)

For demos without physical GPUs, the
[fake-gpu-operator](https://github.com/run-ai/fake-gpu-operator) simulates the GPU stack: a
device plugin registers `nvidia.com/gpu` and MIG resources with the kubelet, and a node-labeling
job stamps every worker with the pool label the flavors use (e.g.
`run.ai/simulated-gpu-node-pool=default`). Everything else in this guide — Kueue, queues,
HardwareProfiles, the plugin — works identically against simulated GPUs.

## 5. User namespaces and labels

The plugin books quota **for namespaces**. Users get a `user-<username>` namespace (created by
the plugin on first booking, or pre-provisioned by GitOps), and shared namespaces can hold
admin-created *system bookings*. Label every managed namespace so the dashboard and Kueue treat
it correctly:

| Label | Value | Purpose |
|---|---|---|
| `opendatahub.io/dashboard` | `"true"` | Namespace appears in the RHOAI/ODH dashboard |
| `kueue-managed` | `"true"` | Marks the namespace as Kueue-managed |
| `kueue.openshift.io/managed` | `"true"` | Same, for tooling that uses the fully-qualified form |
| `rhai-tmm.dev/owner` | `<owner name>` | Optional — shown as the booking owner in the UI |

```bash
oc label namespace user-alice \
  opendatahub.io/dashboard=true \
  kueue-managed=true \
  kueue.openshift.io/managed=true
```

Namespaces also need a Kueue `LocalQueue` for **unreserved** work so users can submit jobs
without a booking (see [Kueue & Quota Model](KUEUE_QUOTA_MODEL.md#localqueues)).

## 6. Baseline queues (optional)

If you want unreserved workloads to run alongside reservations, create a `default` ClusterQueue
plus a shared `unreserved` Cohort before installing the plugin — or let the plugin create the
cohort on its first reservation. Both models are described in
[Kueue & Quota Model](KUEUE_QUOTA_MODEL.md#static-gitops-vs-dynamic-plugin-provisioning).

---

## Verification checklist

Once the components above are in place, verify each layer:

```bash
# 1. Kueue operator is running
oc get pods -n openshift-kueue-operator

# 2. ClusterQueues exist (a `default` queue at minimum)
oc get clusterqueues

# 3. User namespaces have their LocalQueues
oc -n user-alice get localqueues

# 4. ResourceFlavors are registered (created by GitOps or the plugin)
oc get resourceflavors

# 5. GPU nodes advertise GPU/MIG allocatable
oc get nodes -o json | jq '.items[].status.allocatable | with_entries(select(.key | test("nvidia")))'

# 6. HardwareProfiles exist for the dashboard
oc get hardwareprofiles -n redhat-ods-applications
```

If all six checks pass, continue with
[Deploying the Plugin on OpenShift](../deployment/OPENSHIFT_DEPLOY.md) and then configure access
with [RBAC & Access Control](RBAC.md).

## Related pages

- [Kueue & Quota Model](KUEUE_QUOTA_MODEL.md) — how the plugin models GPU quota in Kueue
- [RBAC & Access Control](RBAC.md) — what the plugin's ServiceAccount needs and how to grant
  users/admins access
- [Deploying the Plugin on OpenShift](../deployment/OPENSHIFT_DEPLOY.md) — Helm install and
  dashboard registration
