# Kueue & Quota Model

<span class="badge">Topics: Kueue, Cohort, ClusterQueue, HardwareProfile</span>

---

Every reservation in the booking app is backed by real Kubernetes objects. This page explains the model so administrators understand what the plugin creates, why the shared pool is shaped as a Kueue **Cohort**, and how expiry works. For day-to-day effects (auto-bookings, overrides, preemption) see [Kueue & Auto-Bookings](kueue).

## The `unreserved` cohort

All GPU quota lives in a single Cohort named `unreserved` with two rules:

1. **The cohort owns the whole pool initially.** Unreserved workloads *borrow* — their ClusterQueue has `nominalQuota: 0`.
2. **A reservation carves quota out of the cohort.** The cohort shrinks by exactly the reserved amount and a per-user ClusterQueue is created with that quota, protected.

```
Cohort: unreserved
├── ClusterQueue: unreserved            (borrows everything, quota 0)
├── ClusterQueue: unreserved-priority   (borrows, may preempt other borrowers)
├── ClusterQueue: user-alice            (her reservation -- protected)
└── ClusterQueue: user-bob              (his reservation -- protected)
```

**Borrowing** is "can I use your unused quota?"; **reclaiming** is "I need my quota back". User queues get `reclaimWithinCohort: Any` and `borrowWithinCohort: Never` — they can preempt any borrower to take their reservation back, but never borrow themselves. That is why a reserved slot is a guarantee, and why preempted workloads simply re-queue.

<div class="alert alert-warning">
  <strong>Quotas are additive.</strong> The shared <code>unreserved</code> ClusterQueue must keep <code>nominalQuota: 0</code>. A non-zero quota there would be added on top of the cohort and silently inflate the total.
</div>

Worked example with 8 total GPUs, alice reserves 1:

| Object | `nvidia.com/gpu` quota |
|--------|------------------------|
| Cohort `unreserved` | **7** |
| ClusterQueue `unreserved` | **0** |
| ClusterQueue `user-alice` | **1** |
| Effective total | **8** |

## What a booking creates

When a reservation becomes active, the sync applies three objects. For user `alice` booking 2 full GPUs:

| Object | Name | Details |
|--------|------|---------|
| ClusterQueue | `user-alice` | Cohort `unreserved`, protected quota (2 GPU + share-based CPU/memory), label `rhai-tmm.dev/until` |
| LocalQueue | `reserved` (in `user-alice`) | Points at `user-alice` |
| HardwareProfiles | `reserved-gpu`, ... | One per reserved resource type, wired to the `reserved` LocalQueue |

- The namespace is `user-<username>` (created on demand) — or the target namespace for admin-created [system bookings](making-bookings), which use a `system-<namespace>` ClusterQueue instead.
- CPU/memory for the quota follow each resource type's GPU-equivalent **share** (e.g. a MIG `1g.18gb` slice ≈ 0.125 GPU), floored at 2 CPU / 8Gi.
- All three objects carry the `rhai-tmm.dev/until` label (unix timestamp of the booking end).

## Expiry and cleanup

A cleaner loop runs every 2 minutes:

1. Drops reservations whose end time has passed and returns the quota to the cohort
2. Drains emptied ClusterQueues gracefully (two-phase: stop admission, wait for workloads, force-delete after 30 minutes)
3. Deletes the ClusterQueue, `reserved` LocalQueue and `reserved-*` HardwareProfiles

Cancelling a booking triggers the same sync immediately — the Kueue footprint always tracks the calendar.

## ResourceFlavors

A ResourceFlavor tells Kueue which nodes a quota line applies to — the GPU pool's node label plus a toleration for its taint:

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

The flavor name comes from the app's GPU configuration (default **`h200`**).

## Static (GitOps) vs dynamic provisioning

- **Dynamic (default):** the plugin creates and deletes all reservation objects itself. Simplest to operate.
- **Static (GitOps):** flavors, the `unreserved` cohort and baseline queues are declared in Git and reconciled by ArgoCD, while the plugin still owns the per-reservation objects (`user-*`, `system-*`, `reserved*`).

The two compose cleanly — a common setup manages the baseline in Git and lets the plugin handle reservations. If ArgoCD manages the shared objects, add an `ignoreDifferences` on `/spec/resourceGroups` for ClusterQueue/Cohort so syncs don't revert reservation math. On clusters where Kueue quota (not live node discovery) is the source of truth, set `gpuDiscovery.enabled=false` and supply the resource list in the GPU config.

## Preemption policies

| Queue | Reclaims from cohort | May preempt |
|-------|----------------------|-------------|
| `unreserved` | Never | Nobody |
| `unreserved-priority` | Lower priority | Lower-priority borrowers |
| `user-*` / `system-*` | Any borrower | Whoever borrows its quota |

Preempted workloads appear in the booking page's [preempted workloads banner](kueue) and re-queue automatically.

---

## Next Steps

- [Cluster Prerequisites](cluster-prerequisites) -- installing Kueue and the GPU stack
- [RBAC & Access Control](rbac) -- the permissions behind these operations
- [Kueue & Auto-Bookings](kueue) -- what this means when booking
