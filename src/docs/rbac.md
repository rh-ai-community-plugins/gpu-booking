# RBAC & Access Control

<span class="badge">Topics: RBAC, TokenReview, SubjectAccessReview, ClusterRoles</span>

---

The plugin uses Kubernetes RBAC in two places: the **backend's ServiceAccount** (created by the Helm chart, no action needed) and **your users' identities** (you control who is an admin by binding ClusterRoles to groups).

## How your identity is checked

Every API call travels through the RHOAI dashboard's proxy with your OpenShift Bearer token. The backend then:

1. **TokenReview** -- exchanges the token for your username and groups
2. **SubjectAccessReview** -- asks the API server whether you have the **`admin` verb** on `gpubooking.openshift.io/bookings`
3. Caches the result for 5 minutes; requests without a valid token get `401`

<div class="alert alert-info">
  Any authenticated dashboard user can view the calendar and create bookings — the <code>admin</code> verb only gates privileged features: the Admin Dashboard, system bookings, and deleting other users' bookings. Access to the plugin route itself follows the dashboard's <code>allowedGroups</code> configuration.
</div>

## Backend ServiceAccount permissions

The chart creates a `gpu-booking-bff` ClusterRole + ClusterRoleBinding automatically:

| API group | Resources | Verbs |
|-----------|-----------|-------|
| `authentication.k8s.io` | `tokenreviews` | `create` |
| `authorization.k8s.io` | `subjectaccessreviews` | `create` |
| `kueue.x-k8s.io` | `clusterqueues`, `localqueues` | `get, list, create, patch, delete` |
| `kueue.x-k8s.io` | `cohorts` | `get, create, patch` |
| `kueue.x-k8s.io` | `workloads`, `resourceflavors` | `list` |
| `infrastructure.opendatahub.io` | `hardwareprofiles` | `get, list, patch, delete` |
| core | `nodes` | `list` |
| core | `namespaces` | `get` |

Cluster-scoped is inherent here: ClusterQueues, Cohorts and ResourceFlavors are cluster-scoped objects, and GPU discovery reads all nodes. `create` (not just patch) is required because the reservation sync uses server-side apply.

## Granting users and admins access

The chart ships two ClusterRoles for people — bind them to your OpenShift groups:

```bash
# Admins: full admin dashboard, system bookings, delete-any
oc create clusterrolebinding gpu-booking-admins \
  --clusterrole=gpu-booking-admin \
  --group=my-gpu-cluster-admins

# Regular users
oc create clusterrolebinding gpu-booking-users \
  --clusterrole=gpu-booking-user \
  --group=my-gpu-cluster-users
```

- **`gpu-booking-user`** grants `bookings: use` on `gpubooking.openshift.io`
- **`gpu-booking-admin`** grants `bookings: use, admin`

<div class="alert alert-warning">
  Without an <code>admin</code> binding, <strong>nobody</strong> can open the Admin Dashboard — the SubjectAccessReview fails for every user. Bind the admin role to at least one group.
</div>

## Recommended: Kueue visibility for users

Users who submit workloads benefit from read access to queue state. The production reference deployment ships a `kueue-cluster-visibility` ClusterRole — `get, list, watch` on `clusterqueues`, `localqueues`, `cohorts`, `workloads` and `resourceflavors` (plus status `get`), bound to the user group — and a namespace-scoped `kueue-user-ns` variant for LocalQueue/Workload detail. Full YAML is in the repository: `docs/setup/RBAC.md`.

Do **not** give end users write access to ClusterQueues or Cohorts — the backend's ServiceAccount is the only writer, keeping the cohort quota math consistent.

## Optional guardrail

If users hold broad edit rights in their namespaces, a `ValidatingAdmissionPolicy` can deny non-admin mutations of `clusterqueues` (and `cohorts`/`resourceflavors`), so manual changes can't drift the quota model. See `docs/setup/RBAC.md` for a ready-made policy.

---

## Next Steps

- [Cluster Prerequisites](cluster-prerequisites) -- what must exist before install
- [Kueue & Quota Model](kueue-quota-model) -- what the ServiceAccount creates
- [Admin Dashboard](admin) -- what admins can do once bound
