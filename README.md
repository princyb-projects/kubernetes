# Kubernetes Multi-Tenant RBAC

Repository for Kubernetes RBAC resources designed to support a **multi-tenant cluster** architecture.

---

## Overview

This repository provides a complete Role-Based Access Control (RBAC) design for a multi-tenant Kubernetes cluster. Each tenant is isolated in its own namespace. Three permission tiers are defined per tenant (`admin`, `developer`, `viewer`), plus cluster-wide roles for the platform team.

### Tenants

| Tenant     | Namespace  | Description                  |
|------------|------------|------------------------------|
| `tenant-a` | `tenant-a` | Example tenant A             |
| `tenant-b` | `tenant-b` | Example tenant B             |

Add more tenants by following the patterns in the existing files.

---

## RBAC Architecture

```
Cluster
├── ClusterRole: platform-admin         → ClusterRoleBinding: platform-admin-binding  (Group: platform-admins)
├── ClusterRole: cluster-viewer         → ClusterRoleBinding: cluster-viewer-binding   (Group: cluster-viewers)
│
├── ClusterRole: tenant-admin    ─┐
├── ClusterRole: tenant-developer ├─ (shared definitions, scoped per namespace via RoleBindings below)
├── ClusterRole: tenant-viewer   ─┘
│
├── Namespace: tenant-a
│   ├── RoleBinding: tenant-a-admin-binding      → ClusterRole: tenant-admin      (SA: tenant-a-admin)
│   ├── RoleBinding: tenant-a-developer-binding  → ClusterRole: tenant-developer  (SA: tenant-a-developer)
│   └── RoleBinding: tenant-a-viewer-binding     → ClusterRole: tenant-viewer     (SA: tenant-a-viewer)
│
└── Namespace: tenant-b
    ├── RoleBinding: tenant-b-admin-binding      → ClusterRole: tenant-admin      (SA: tenant-b-admin)
    ├── RoleBinding: tenant-b-developer-binding  → ClusterRole: tenant-developer  (SA: tenant-b-developer)
    └── RoleBinding: tenant-b-viewer-binding     → ClusterRole: tenant-viewer     (SA: tenant-b-viewer)
```

---

## Role Definitions

### Tenant ClusterRoles (scoped to a namespace via RoleBindings)

These are defined as `ClusterRole` resources so the definitions are shared and not duplicated per namespace. They are always applied via namespace-scoped `RoleBinding` objects, so the effective permissions are **confined to the bound namespace only**.

| ClusterRole        | Description                                                                 |
|--------------------|-----------------------------------------------------------------------------|
| `tenant-admin`     | Full control over workload resources, ingress, and network policy within the namespace. `pods/exec` and namespace-level RBAC management are intentionally excluded. |
| `tenant-developer` | Deploy and manage workloads (Deployments, Jobs, Services, ConfigMaps). Read-only access to Secrets. No `exec` access to pods. |
| `tenant-viewer`    | Read-only (`get`, `list`, `watch`) access to all standard resources in the namespace. |

### Cluster-Scoped Roles

| ClusterRole       | Description                                                                 |
|-------------------|-----------------------------------------------------------------------------|
| `platform-admin`  | Cluster-wide administrative access: manage namespaces, storage, RBAC, quotas, and network policies. Excludes node deletion. |
| `cluster-viewer`  | Cluster-wide read-only access across all namespaces. Suitable for monitoring tools and auditors. |

---

## Repository Structure

```
rbac/
├── namespaces/
│   ├── tenant-a.yaml                   # Namespace for tenant-a
│   └── tenant-b.yaml                   # Namespace for tenant-b
├── service-accounts/
│   ├── tenant-a.yaml                   # ServiceAccounts: tenant-a-{admin,developer,viewer}
│   └── tenant-b.yaml                   # ServiceAccounts: tenant-b-{admin,developer,viewer}
├── roles/
│   ├── tenant-admin.yaml               # ClusterRole: tenant-admin (applied via RoleBindings)
│   ├── tenant-developer.yaml           # ClusterRole: tenant-developer (applied via RoleBindings)
│   └── tenant-viewer.yaml              # ClusterRole: tenant-viewer (applied via RoleBindings)
├── role-bindings/
│   ├── tenant-a.yaml                   # RoleBindings for tenant-a
│   └── tenant-b.yaml                   # RoleBindings for tenant-b
├── cluster-roles/
│   └── cluster-roles.yaml              # ClusterRoles: platform-admin, cluster-viewer
└── cluster-role-bindings/
    └── cluster-role-bindings.yaml      # ClusterRoleBindings for platform-admin and cluster-viewer
```

---

## Usage

### Apply all RBAC resources in order

```bash
# 1. Create namespaces
kubectl apply -f rbac/namespaces/

# 2. Create ServiceAccounts
kubectl apply -f rbac/service-accounts/

# 3. Create shared tenant ClusterRoles (permissions are namespace-scoped via RoleBindings)
kubectl apply -f rbac/roles/

# 4. Create RoleBindings (scopes each ClusterRole to its tenant namespace)
kubectl apply -f rbac/role-bindings/

# 5. Create platform ClusterRoles and ClusterRoleBindings
kubectl apply -f rbac/cluster-roles/
kubectl apply -f rbac/cluster-role-bindings/
```

### Verify permissions

```bash
# Check if tenant-a-developer can list pods in tenant-a
kubectl auth can-i list pods --namespace tenant-a \
  --as system:serviceaccount:tenant-a:tenant-a-developer

# Check that tenant-a-developer cannot access tenant-b namespace
kubectl auth can-i list pods --namespace tenant-b \
  --as system:serviceaccount:tenant-a:tenant-a-developer

# Check platform-admin can list namespaces
kubectl auth can-i list namespaces \
  --as-group platform-admins --as fake-admin-user
```

---

## Adding a New Tenant

1. Copy `rbac/namespaces/tenant-a.yaml` → `rbac/namespaces/tenant-c.yaml` and update the `name` and `tenant` label.
2. Copy `rbac/service-accounts/tenant-a.yaml` → `rbac/service-accounts/tenant-c.yaml` and update names/namespaces.
3. Copy `rbac/role-bindings/tenant-a.yaml` → `rbac/role-bindings/tenant-c.yaml` and update names/namespaces.
4. The shared `roles/` definitions are reused — no changes needed there.
5. Apply the new files:

```bash
kubectl apply -f rbac/namespaces/tenant-c.yaml
kubectl apply -f rbac/service-accounts/tenant-c.yaml
kubectl apply -f rbac/role-bindings/tenant-c.yaml
```

---

## Security Considerations

- **Namespace isolation**: Each tenant is confined to their own namespace. Cross-namespace access is not granted by any RoleBinding in this repository. Tenant ClusterRoles are bound via namespace-scoped `RoleBinding` objects (never `ClusterRoleBinding`), so their effective permissions are restricted to the bound namespace.
- **Least privilege**: The `tenant-developer` role cannot exec into pods or create/delete Secrets. The `tenant-viewer` role has no write access. `pods/exec` is excluded from all tenant roles to prevent arbitrary command execution inside containers.
- **No privilege escalation via RBAC**: `tenant-admin` cannot manage Roles or RoleBindings within its namespace. Namespace-level RBAC is reserved for the platform team, preventing a tenant from granting themselves elevated permissions.
- **No wildcard verbs or resources**: All rules explicitly enumerate resources and verbs to avoid accidental privilege escalation.
- **ClusterRoleBindings are reserved** for the platform team (`platform-admins` group) and monitoring/audit tools (`cluster-viewers` group). Tenants never receive ClusterRoleBindings.
- **Secrets access**: Only `tenant-admin` can create and delete Secrets. Developers have read-only Secret access (required for referencing existing secrets in workloads). Consider using an external secrets manager (e.g., Vault, External Secrets Operator) for stricter control.
- **Human users and groups**: RoleBindings include commented-out examples for binding to `User` and `Group` subjects (e.g., from an OIDC/SSO provider). Uncomment and populate these as appropriate for your environment.
