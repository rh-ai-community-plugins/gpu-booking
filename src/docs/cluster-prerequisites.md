# Cluster Prerequisites

<span class="badge">Topics: Setup, Kueue, GPUs, Namespaces</span>

---

This page is for **cluster administrators**. The GPU Booking plugin manages GPU quota through [Kueue](kueue-quota-model) and discovers GPU resources from the cluster, so a few building blocks must exist before installing it. The full reference (with copy-paste YAML) lives in the repository: `docs/setup/PREREQUISITES.md`.

## Install order

| Order | Component | Why |
|-------|-----------|-----|
| 1 | OpenShift AI (or ODH) | Hosts the dashboard, provides the `HardwareProfile` API |
| 2 | Kueue operator | Job queueing and quota model the plugin reserves with |
| 3 | Node Feature Discovery | Labels nodes with hardware capabilities |
| 4 | GPU capacity | NVIDIA GPU Operator + MIG config (or a fake-GPU operator in labs) |
| 5 | User namespaces + labels | Namespaces users book into |
| 6 | Baseline queues (optional) | A `default` ClusterQueue/LocalQueue for unreserved work |
| 7 | **GPU Booking plugin** | Last — it consumes everything above |

---

## OpenShift AI

Install the RHOAI operator (namespace `redhat-ods-operator`, channel `stable-3.5` in the production reference) and a `DataScienceCluster`. Two settings matter:

- **`kueue.managementState: Unmanaged`** on the DSC, so the standalone Kueue operator owns queue configuration and the plugin's dynamic queues are not reconciled away. Keep `defaultClusterQueueName: default` and `defaultLocalQueueName: default`.
- **Kueue UI enabled** on the dashboard (`disableKueue: false`) so users can see workloads alongside bookings.

<div class="alert alert-info">
  On the upstream Open Data Hub distribution, substitute the <code>opendatahub</code> namespace and <code>odh-dashboard</code> names.
</div>

## Kueue operator

Install via OLM from `redhat-operators`:

- Name `kueue-operator`, channel **`stable-v1.4`**, namespace `openshift-kueue-operator` with a global OperatorGroup
- Create the `Kueue` instance with the workload framework integrations you need (Pod, PyTorchJob, RayCluster/Job, StatefulSet, TrainJob)
- Optional for distributed workloads: **leader-worker-set** and **job-set** operators (`stable-v1.0`)

## GPU capacity

- **Node Feature Discovery** (`openshift-nfd`, channel `stable`) produces the `nvidia.com/gpu.present=true` node label
- **NVIDIA GPU Operator** (certified-operators) with a `ClusterPolicy` enabling `migManager` and `mig.strategy: mixed`
- GPU nodes typically sit in a dedicated machine set with a role label (`node-role.kubernetes.io/gpu`) and a GPU taint (`nvidia.com/gpu=h200:PreferNoSchedule`); the Kueue ResourceFlavor tolerates the taint and matches a node label
- MIG slices advertised by the nodes (e.g. `nvidia.com/mig-1g.18gb`, `nvidia.com/mig-3g.71gb`) each become a bookable resource type

<div class="alert alert-info">
  <strong>No physical GPUs?</strong> The fake-gpu-operator simulates the GPU stack for labs and demos — device plugin, MIG resources and pool labels. Everything in the booking app works identically against simulated GPUs.
</div>

## User namespaces and labels

Bookings reserve quota **for namespaces**. Label every managed namespace:

```bash
oc label namespace user-alice \
  opendatahub.io/dashboard=true \
  kueue-managed=true \
  kueue.openshift.io/managed=true
```

| Label | Purpose |
|-------|---------|
| `opendatahub.io/dashboard: "true"` | Namespace appears in the RHOAI/ODH dashboard |
| `kueue-managed` / `kueue.openshift.io/managed: "true"` | Marks the namespace as Kueue-managed |
| `rhai-tmm.dev/owner` | Optional — booking owner display name |

The plugin creates a `user-<username>` namespace automatically on a user's first booking if it doesn't exist.

## Verify before installing

```bash
oc get pods -n openshift-kueue-operator   # Kueue controller running
oc get clusterqueues                      # default queue exists
oc get resourceflavors                    # GPU flavor registered
oc get hardwareprofiles -n redhat-ods-applications
```

---

## Next Steps

- [Deploying the plugin](https://github.com/rh-ai-community-plugins/gpu-booking/blob/main/docs/deployment/OPENSHIFT_DEPLOY.md) -- Helm install and dashboard registration
- [Kueue & Quota Model](kueue-quota-model) -- how bookings become Kueue objects
- [RBAC & Access Control](rbac) -- granting users and admins access
