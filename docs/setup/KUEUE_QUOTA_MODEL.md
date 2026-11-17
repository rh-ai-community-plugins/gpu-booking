# Kueue & Quota Model

The plugin turns GPU reservations into native Kueue objects. This page explains the quota model —
how a booking becomes ClusterQueues, LocalQueues and HardwareProfiles, and why the shared pool is
shaped as a Kueue **Cohort**.

For what this means for day-to-day booking (auto-bookings, overrides, preemption banners), see
the *Kueue & Auto-Bookings* topic in the plugin's in-app help.

---

## The `unreserved` cohort design

All GPU quota lives in a single Cohort named `unreserved`. The design has two rules:

1. **The cohort initially owns all resources.** Every unreserved workload *borrows* — its
   ClusterQueue has `nominalQuota: 0` and pulls from the cohort.
2. **A reservation carves quota out of the cohort.** When a user books, the cohort's quota is
   reduced by exactly the reserved amount, and a per-user ClusterQueue is created with that
   quota as protected `nominalQuota`.

```
Cohort: unreserved                      ← owns the whole pool, shrinks as users reserve
├── ClusterQueue: unreserved            (nominalQuota: 0 — borrows everything)
├── ClusterQueue: unreserved-priority   (borrows, may preempt other borrowers)
├── ClusterQueue: user-alice            (nominalQuota: her reservation — protected)
└── ClusterQueue: user-bob              (nominalQuota: his reservation — protected)
```

### Borrowing vs reclaiming

- **Borrowing** is *"can I use your unused quota?"* — an unreserved workload runs on capacity
  nobody has reserved.
- **Reclaiming** is *"I need my quota back"* — a reserved workload evicts borrowers.

User ClusterQueues are configured with `reclaimWithinCohort: Any` and
`borrowWithinCohort: Never`: they can preempt **any** borrowing workload to reclaim their quota,
but never borrow from the cohort themselves. This guarantees a reservation is honoured exactly,
while unreserved workloads still enjoy the idle pool. Unreserved workloads are not lost when
preempted — Kueue re-queues them and they resume when capacity frees up.

### Quotas are additive

Cohort and ClusterQueue quotas are **additive** in Kueue, so the plugin must reduce the Cohort by
exactly the reserved amount. If the shared `unreserved` ClusterQueue also declared a non-zero
`nominalQuota`, that would be added on top and silently inflate the total. Keep it at `0`.

Worked example — 8 total GPUs, alice reserves 1:

| Object | `nvidia.com/gpu` quota |
|---|---|
| Cohort `unreserved` | **7** (8 − 1) |
| ClusterQueue `unreserved` | **0** |
| ClusterQueue `user-alice` | **1** (protected) |
| **Effective total** | **8** ✔ |

---

## Building blocks

### ResourceFlavors

A ResourceFlavor tells Kueue *which nodes* a quota line applies to. It matches the GPU pool's
node label and tolerates its taint:

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: h200
spec:
  nodeLabels:
    node.kubernetes.io/instance-type: gx3d-160x1792x8h200
  tolerations:
    - key: nvidia.com/gpu
      operator: Equal
      value: h200
      effect: NoSchedule
```

The flavor name is configurable in the plugin's GPU configuration (`flavorName`, default
**`h200`**) and is used on every ClusterQueue quota line the plugin creates. A plain
`default-flavor` (no node labels) is typically reserved for CPU-only queues.

### ClusterQueues

**`unreserved`** — the shared pool. Zero quota on every flavor; borrows from the cohort:

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: unreserved
spec:
  cohort: unreserved
  namespaceSelector: {}          # any namespace may submit
  preemption:
    reclaimWithinCohort: Never   # never evicts a reserved workload
  queueingStrategy: BestEffortFIFO
  resourceGroups:
    - coveredResources: [cpu, memory, nvidia.com/gpu, nvidia.com/mig-1g.18gb]
      flavors:
        - name: h200
          resources:
            - name: nvidia.com/gpu
              nominalQuota: "0"
```

**`unreserved-priority`** — an optional queue for high-priority unreserved work (e.g. platform
services). Restricted to namespaces with a selector label and allowed to preempt *other
borrowers* below a priority threshold:

```yaml
spec:
  cohort: unreserved
  namespaceSelector:
    matchLabels:
      rhai-tmm.dev/clusterqueues: h200-priority
  preemption:
    borrowWithinCohort:
      policy: LowerPriority
      maxPriorityThreshold: 100
    reclaimWithinCohort: LowerPriority
```

Paired with a WorkloadPriorityClass:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 10000
```

**(Per-user / per-namespace) reservation queues** — created by the plugin, see below.

### LocalQueues

LocalQueues are the namespace-scoped entry point workloads submit to. A typical namespace has:

| LocalQueue | ClusterQueue | Purpose |
|---|---|---|
| `unreserved` | `unreserved` | Best-effort work, borrows idle capacity |
| `default` | `default` | CPU-only work (optional) |
| `reserved` | `user-<username>` / `system-<namespace>` | Created by the plugin for a booking |

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: unreserved
  namespace: user-alice
spec:
  clusterQueue: unreserved
```

### HardwareProfiles

HardwareProfiles (`infrastructure.opendatahub.io/v1`, namespace `redhat-ods-applications` for
shared ones) expose the queues in the RHOAI/ODH dashboard workbench launcher:

```yaml
apiVersion: infrastructure.opendatahub.io/v1
kind: HardwareProfile
metadata:
  name: unreserved-gpu
  namespace: redhat-ods-applications
  annotations:
    opendatahub.io/managed: "false"
    opendatahub.io/dashboard-feature-visibility: "[]"
spec:
  identifiers:
    - identifier: cpu
      resourceType: CPU
      maxCount: 19
    - identifier: memory
      resourceType: Memory
      maxCount: 216Gi
    - identifier: nvidia.com/gpu
      resourceType: Accelerator
      maxCount: 8
  scheduling:
    type: Queue
    kueue:
      localQueueName: unreserved
```

---

## What the plugin creates for a booking

When a reservation becomes active (today's date within the booking window), the reservation sync
applies three objects. For a user `alice` booking 2 full GPUs for the day:

**1. ClusterQueue `user-alice`** (namespace `user-alice`):

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: user-alice
  labels:
    rhai-tmm.dev/until: "1764729600"   # booking end, unix seconds — drives expiry
spec:
  cohort: unreserved
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: user-alice
  preemption:
    borrowWithinCohort: {policy: Never}   # protected quota: never borrowed away
    reclaimWithinCohort: Any              # evicts borrowers to reclaim quota
    withinClusterQueue: Never
  flavorFungibility:
    whenCanBorrow: Borrow
    whenCanPreempt: TryNextFlavor
  queueingStrategy: BestEffortFIFO
  resourceGroups:
    - coveredResources: [cpu, memory, nvidia.com/gpu, ...]
      flavors:
        - name: h200
          resources:
            - name: nvidia.com/gpu
              nominalQuota: "2"
            - name: cpu
              nominalQuota: "76"
            - name: memory
              nominalQuota: "760Gi"
```

CPU/memory follow the GPU config's *share* model: each resource type has a `gpuEquivalent`
share (e.g. a MIG `1g.18gb` slice ≈ 0.125 GPU), and the quota is `share × totalCPU/memory`,
with floors of 2 CPU / 8Gi. The user's namespace is created if it does not exist yet.

**2. LocalQueue `reserved`** in `user-alice`, pointing at `user-alice`.

**3. HardwareProfile `reserved-gpu`** (one per reserved resource type) in `user-alice`, with
`scheduling.kueue.localQueueName: reserved` — this is the profile users pick in the workbench
launcher to land on their protected quota.

**System bookings** (admin-created for an existing namespace such as a shared model-serving
project) follow the same pattern but target the namespace directly: ClusterQueue
`system-<namespace>`, a `reserved` LocalQueue and `reserved-<resource>` HardwareProfiles inside
the target namespace, plus a `rhai-tmm.dev/namespace` label on the ClusterQueue for cleanup.

### Expiry and cleanup

Every plugin-created object carries a `rhai-tmm.dev/until` label (unix timestamp of the last
booked slot). A cleaner loop every 2 minutes:

1. Drops reservations whose active window has passed and recomputes the `unreserved` Cohort with
   the freed quota
2. Drains ClusterQueues with no remaining bookings (graceful two-phase drain; workloads finish
   or re-queue, force-deleted after 30 minutes)
3. Deletes the ClusterQueue, `reserved` LocalQueue and `reserved-*` HardwareProfiles once empty

Cancelling a booking triggers the same sync immediately, so the Kueue footprint tracks the
calendar.

---

## Static (GitOps) vs dynamic (plugin) provisioning

Two supported operating models:

| | Static (GitOps) | Dynamic (plugin-managed) |
|---|---|---|
| ResourceFlavors, ClusterQueues, LocalQueues, HardwareProfiles | Declared in Git, rendered by a Helm chart, reconciled by ArgoCD | Created/updated/deleted by the plugin's reservation sync |
| Reservations | Still need the plugin (or manual edits) to carve quota from the cohort | Fully automatic from bookings |
| Best for | Regulated environments where every object must be reviewed | Fast-moving teams; the default |

The two compose: many deployments manage the *baseline* (flavor, `unreserved` cohort and
queues, `default` profiles) in Git while letting the plugin own the per-reservation objects —
the plugin only touches objects it names itself (`user-*`, `system-*`, `reserved*`). If you use
ArgoCD over the shared objects, add an `ignoreDifferences` on `/spec/resourceGroups` for
`ClusterQueue`/`Cohort` so syncs don't revert the quota the plugin subtracts for reservations.

On clusters where Kueue quota (rather than live node discovery) is the source of truth, set
`gpuDiscovery.enabled=false` and provide the resource list via the GPU config — this also fixes
the flavor name.

---

## Preemption policies at a glance

| Queue | `reclaimWithinCohort` | `borrowWithinCohort` | Effect |
|---|---|---|---|
| `unreserved` | `Never` | (default) | Can run on spare capacity, never evicts anyone |
| `unreserved-priority` | `LowerPriority` | `LowerPriority` (threshold 100) | May preempt lower-priority borrowers |
| `user-*` / `system-*` | `Any` | `Never` | Reclaims its full reservation from any borrower |

Preempted workloads surface in the plugin's booking page (preempted-workloads banner) and in
Kueue as `Evicted`/`Preempted` conditions on the Workload.

## Related pages

- [Cluster Prerequisites](PREREQUISITES.md) — installing Kueue and the GPU stack
- [RBAC & Access Control](RBAC.md) — the permissions behind these API calls
- [Deploying the Plugin on OpenShift](../deployment/OPENSHIFT_DEPLOY.md) — chart values
  (`kueue.syncEnabled`, `kueue.syncIntervalSeconds`, `gpuDiscovery.enabled`)
