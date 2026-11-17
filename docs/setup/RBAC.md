# RBAC & Access Control

The plugin touches the Kubernetes API on two fronts, and each has its own RBAC story:

1. **The backend's ServiceAccount** — performs all Kueue/HardwareProfile/node operations on
   behalf of the app. Created automatically by the Helm chart.
2. **The end user's identity** — forwarded as a Bearer token and verified per-request to decide
   *who* the user is and *whether they are an admin*. Controlled by ClusterRoles you bind to
   your users and groups.

---

## How user authentication works

Every `/api/*` request arrives through the RHOAI dashboard's proxy with the user's OpenShift
Bearer token (`authorize: True` in the `proxyService` entry). The backend's auth middleware
then:

1. **TokenReview** — exchanges the Bearer token for the authenticated user's name and groups
   (`authentication.k8s.io/tokenreviews`, verb `create`)
2. **SubjectAccessReview** — asks the API server whether that user has the **`admin` verb** on
   `gpubooking.openshift.io/bookings` (`authorization.k8s.io/subjectaccessreviews`, verb
   `create`). This is what the `gpu-booking-admin` ClusterRole grants.
3. **Cache** — the resulting identity is cached for 5 minutes (keyed by a hash of the token);
   `/health` bypasses auth entirely.

The result (`username`, `groups`, `is_admin`) is attached to the request context. Requests
without a valid token get `401`.

> **Note:** the middleware does not check a `use` verb — any authenticated dashboard user can
> view the calendar and create bookings. Access to the plugin's *route* is governed by the
> dashboard's `allowedGroups` configuration; the admin verb only gates privileged operations
> (admin dashboard, system bookings, deleting other users' bookings).

## Backend ServiceAccount permissions

The Helm chart creates a `gpu-booking-bff` ClusterRole + ClusterRoleBinding
([`chart/templates/bff-rbac.yaml`](../../chart/templates/bff-rbac.yaml)):

| API group | Resources | Verbs | Why |
|---|---|---|---|
| `authentication.k8s.io` | `tokenreviews` | `create` | Verify user Bearer tokens |
| `authorization.k8s.io` | `subjectaccessreviews` | `create` | Determine admin users |
| `kueue.x-k8s.io` | `clusterqueues` | `get, list, create, patch, delete` | Reservation queues + shared pool |
| `kueue.x-k8s.io` | `localqueues` | `get, list, create, patch, delete` | `reserved` LocalQueue per namespace |
| `kueue.x-k8s.io` | `cohorts` | `get, create, patch` | Shrink/restore the `unreserved` cohort |
| `kueue.x-k8s.io` | `workloads` | `list` | Auto-bookings from live usage; preemption banner |
| `kueue.x-k8s.io` | `resourceflavors` | `list` | Resolve the flavor used in quotas |
| `infrastructure.opendatahub.io` | `hardwareprofiles` | `get, list, patch, delete` | `reserved-*` profiles for workbench launches |
| `""` (core) | `nodes` | `list` | GPU/MIG discovery from node allocatable |
| `""` (core) | `namespaces` | `get` | Verify/create `user-<username>` namespaces |

Notes:

- **`create` on ClusterQueues/LocalQueues/Cohorts is required for server-side apply** — the
  sync uses `PATCH apply-patch+yaml`, which needs create rights for objects that do not exist
  yet.
- **Cluster-scoped is inherent**: ClusterQueues, Cohorts and ResourceFlavors are cluster-scoped
  API objects, and GPU discovery needs to read all nodes. The chart cannot scope these to a
  namespace. The only namespaced objects the backend manages (LocalQueues, HardwareProfiles)
  are still reached through the ClusterRole for simplicity.
- If you *must* restrict further, the HardwareProfile rules can be converted to a namespaced
  `Role` in `redhat-ods-applications` (where dashboard-visible profiles live) plus per-user
  namespace `Role`s — at the cost of more bindings to maintain.

## Granting users and admins access

The chart also ships two ClusterRoles for *people*:

```yaml
# gpu-booking-user — marks regular users of the plugin
rules:
  - apiGroups: ["gpubooking.openshift.io"]
    resources: ["bookings"]
    verbs: ["use"]

# gpu-booking-admin — enables admin features
rules:
  - apiGroups: ["gpubooking.openshift.io"]
    resources: ["bookings"]
    verbs: ["use", "admin"]
```

Bind them to your OpenShift groups (the backend's SubjectAccessReview sees group memberships
from the TokenReview, so group bindings are the idiomatic path):

```bash
# Admins: full admin dashboard, system bookings, delete-any
oc create clusterrolebinding gpu-booking-admins \
  --clusterrole=gpu-booking-admin \
  --group=rhdp-ai-bu-gpu-cluster-admins

# Regular users
oc create clusterrolebinding gpu-booking-users \
  --clusterrole=gpu-booking-user \
  --group=rhdp-ai-bu-gpu-cluster-users
```

Without an `admin` binding nobody sees the Admin Dashboard — the SubjectAccessReview returns
`false` for every user.

## Recommended supporting roles

Users interacting with workloads and queues benefit from read access to Kueue state. These are
the roles the production reference deployment ships (independent of the plugin, useful for any
Kueue cluster):

**`kueue-cluster-visibility`** — cluster-wide read-only view of queues, bound to the user group:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kueue-cluster-visibility
rules:
  - apiGroups: ["kueue.x-k8s.io"]
    resources: [cohorts, clusterqueues, localqueues, workloads, resourceflavors, workloadpriorityclasses]
    verbs: [get, list, watch]
  - apiGroups: ["kueue.x-k8s.io"]
    resources: ["clusterqueues/status", "localqueues/status", "workloads/status"]
    verbs: [get]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kueue-cluster-visibility
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kueue-cluster-visibility
subjects:
  - kind: Group
    name: rhdp-ai-bu-gpu-cluster-users
    apiGroup: rbac.authorization.k8s.io
```

**`kueue-user-ns`** — namespace-scoped variant (LocalQueues/Workloads + pending-workload
visibility), bound via `RoleBinding` in each user namespace. The production deployment binds it
automatically in every `user-<username>` namespace.

Do **not** grant end users write access to ClusterQueues or Cohorts — the plugin's ServiceAccount
is the only writer, which keeps the cohort math consistent.

## Optional guardrail: block manual queue mutation

If users have broad edit rights in their namespaces (e.g. `admin`), a
ValidatingAdmissionPolicy can stop anyone but cluster-admins from mutating the shared Kueue
objects — this is what the production reference deployment enforces:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: blastwall-kueue-admin-only
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["kueue.x-k8s.io"]
        apiVersions: ["*"]
        operations: ["CREATE", "UPDATE", "DELETE"]
        resources: ["clusterqueues"]
  validations:
    - expression: |
        !request.userInfo.groups.exists(g, g == 'rhdp-ai-bu-gpu-cluster-users')
      message: "ClusterQueues are managed by the GPU Booking plugin (GitOps)."
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: blastwall-kueue-admin-only
spec:
  policyName: blastwall-kueue-admin-only
  validationActions: [Deny]
```

Adjust the group and resource list (add `cohorts`, `resourceflavors`) to your environment.

## Related pages

- [Cluster Prerequisites](PREREQUISITES.md) — the components these permissions operate on
- [Kueue & Quota Model](KUEUE_QUOTA_MODEL.md) — what the ServiceAccount does with them
- [Deploying the Plugin on OpenShift](../deployment/OPENSHIFT_DEPLOY.md) — `bff.rbac.create`
  chart value and the registration step
