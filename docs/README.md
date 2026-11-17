# Documentation

Project documentation:

```text
docs/setup/       — cluster prerequisites, Kueue quota model, RBAC
docs/deployment/  — OpenShift deployment with Helm and dashboard registration
```

## Setup

- [Cluster Prerequisites](setup/PREREQUISITES.md) — OpenShift AI, Kueue operator, GPU capacity
  (NFD/GPU operator/MIG), namespace labels, install order, verification checklist
- [Kueue & Quota Model](setup/KUEUE_QUOTA_MODEL.md) — the `unreserved` cohort design, what the
  plugin creates per booking, expiry/drain lifecycle, static vs dynamic provisioning
- [RBAC & Access Control](setup/RBAC.md) — auth flow (TokenReview/SubjectAccessReview), backend
  ServiceAccount permissions, user/admin role bindings, supporting visibility roles

## Deployment

- [Deploying the Plugin on OpenShift](deployment/OPENSHIFT_DEPLOY.md) — Helm install,
  `MODULE_FEDERATION_CONFIG` registration, granting users access, verification, uninstall.
