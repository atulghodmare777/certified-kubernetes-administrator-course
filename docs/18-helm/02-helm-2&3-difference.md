# Helm 2 vs Helm 3 — Differences & Features

---

## What is Helm?
Helm is the **package manager for Kubernetes**. It lets you install, upgrade, and manage Kubernetes applications using pre-built templates called **Charts**.

---

## Architecture

### Helm 2 — Client + Server
Helm 2 had two components:
- **Helm CLI** — runs on your machine
- **Tiller** — a server-side Pod running inside the Kubernetes cluster

The CLI sent commands to Tiller via gRPC, and Tiller talked to the Kubernetes API on your behalf.

```
helm CLI  --gRPC-->  Tiller (Pod in cluster)  -->  Kubernetes API
```

### Helm 3 — Client Only
Helm 3 removed Tiller completely. The CLI talks directly to the Kubernetes API using your kubeconfig.

```
helm CLI  -->  Kubernetes API
```

---

## What Was Wrong in Helm 2 & What Helm 3 Fixed

---

### 1. Tiller — Security Risk

**Helm 2 Problem:**
Tiller ran inside the cluster with `cluster-admin` permissions by default. This meant it could create, modify, or delete *any* resource in *any* namespace. A low-privileged user who could reach Tiller could effectively do admin-level operations, bypassing Kubernetes security entirely.

**Helm 3 Fix:**
Tiller is gone. Helm 3 uses your own kubeconfig credentials to talk directly to the Kubernetes API. Whatever you can do in Kubernetes is what you can do with Helm — no more, no less.

---

### 2. RBAC Support

**Helm 2 Problem:**
Kubernetes RBAC (Role-Based Access Control) was completely bypassed because Tiller held the actual permissions. Configuring Tiller with fine-grained RBAC was complex and rarely done correctly, so most teams just gave it full admin access.

**Helm 3 Fix:**
Since Helm 3 uses your kubeconfig directly, RBAC works natively. A user with read-only permissions cannot install or delete releases. Permissions are properly scoped per user.

---

### 3. Release Storage — ConfigMaps vs Secrets

**Helm 2 Problem:**
Helm 2 stored all release information (chart name, values used, history) in **ConfigMaps** inside the `kube-system` namespace. ConfigMaps are plain text and not encrypted, which meant sensitive data like passwords or tokens passed as values could be read by anyone with access to that namespace.

**Helm 3 Fix:**
Helm 3 stores release data in **Kubernetes Secrets**, which are namespace-scoped and can be encrypted at rest. This is much more secure for storing release metadata.

---

### 4. Release Names — Global vs Namespace-Scoped

**Helm 2 Problem:**
Release names in Helm 2 were **global across the entire cluster**. You could not have two releases with the same name even if they were in completely different namespaces. This made multi-environment setups (dev, staging, prod) awkward.

**Helm 3 Fix:**
Release names in Helm 3 are **scoped to a namespace**. You can now have:
```
helm install myapp ./chart -n dev
helm install myapp ./chart -n prod
```
Both are independent releases with the same name in different namespaces.

---

### 5. Upgrade Strategy — Two-Way vs Three-Way Merge

**Helm 2 Problem:**
Helm 2 used a **two-way merge** when upgrading — it only compared the old chart and the new chart. It completely ignored the live state of resources in the cluster. So if someone manually changed a resource (e.g., scaled replicas via `kubectl`), a `helm upgrade` would overwrite that change without warning.

**Helm 3 Fix:**
Helm 3 uses a **three-way strategic merge** — it compares:
- The old chart manifest
- The new chart manifest
- The **current live state** in the cluster

This means manual changes are detected and respected during upgrades, making releases much safer.

---

### 6. Dependency File — `requirements.yaml` vs `Chart.yaml`

**Helm 2 Problem:**
Chart dependencies were defined in a **separate file** called `requirements.yaml`. You had to manage two files (`Chart.yaml` + `requirements.yaml`) and run `helm dependency update` to sync them. This was confusing and easy to get out of sync.

**Helm 3 Fix:**
Dependencies are now declared directly inside `Chart.yaml` under the `dependencies:` key. One file, one source of truth.

```yaml
# Chart.yaml in Helm 3
dependencies:
  - name: redis
    version: "14.x.x"
    repository: "https://charts.bitnami.com/bitnami"
```

---

### 7. Delete Behavior

**Helm 2 Problem:**
Running `helm delete <release>` did **not fully remove** a release. It only soft-deleted it, keeping the history. To completely remove a release you had to run:
```
helm delete --purge <release-name>
```
This caused confusion — developers assumed deletion was complete when it wasn't.

**Helm 3 Fix:**
`helm uninstall <release>` completely removes the release by default. If you want to keep history, you explicitly pass `--keep-history`. Clean and predictable.

---

### 8. CRD (Custom Resource Definition) Handling

**Helm 2 Problem:**
CRDs were installed using a special `crd-install` hook. This was unreliable — timing issues often caused CRDs to not be ready before the chart's resources tried to use them, leading to failed deployments.

**Helm 3 Fix:**
Helm 3 introduced a dedicated `crds/` directory inside a chart. Any CRDs placed here are **installed first**, before any other chart resources. This guarantees CRDs are available when needed.

---

### 9. Values Validation

**Helm 2 Problem:**
There was no way to validate user-supplied values before installation. If someone passed a wrong type or missed a required field, the error only appeared after Kubernetes rejected the rendered manifests — hard to debug.

**Helm 3 Fix:**
Helm 3 supports a `values.schema.json` file inside a chart. Chart authors can define the expected structure, types, and required fields. Helm validates values **before** doing anything, giving clear, early error messages.

---

### 10. Library Charts

**Helm 2 Problem:**
There was no way to share common template helpers across multiple charts without duplicating code. Every chart had to redefine the same helper templates.

**Helm 3 Fix:**
Helm 3 introduced **Library Charts** — a chart type that contains only reusable named templates and cannot be installed on its own. Other charts declare it as a dependency and reuse its templates, eliminating duplication.

---

### 11. Chart API Version

**Helm 2:** Charts used `apiVersion: v1` in `Chart.yaml`.

**Helm 3:** Charts use `apiVersion: v2`, which:
- Enables declaring chart type (`application` or `library`)
- Allows specifying `kubeVersion` constraints
- Helm 3 still supports `v1` charts for backward compatibility

---

## Summary Table

| Feature | Helm 2 | Helm 3 |
|---|---|---|
| Architecture | CLI + Tiller (server) | CLI only |
| Security | Tiller had cluster-admin | Uses user's kubeconfig |
| RBAC | Bypassed by Tiller | Fully native |
| Release storage | ConfigMaps (plain text) | Secrets (encrypted) |
| Release scope | Cluster-wide (global) | Namespace-scoped |
| Upgrade strategy | Two-way merge | Three-way strategic merge |
| Dependencies | `requirements.yaml` | `Chart.yaml` |
| Delete behavior | Soft delete, needed `--purge` | Full delete by default |
| CRD handling | `crd-install` hook (unreliable) | `crds/` directory (reliable) |
| Values validation | Not supported | JSON Schema validation |
| Library Charts | Not available | Supported |
| Chart API version | `v1` | `v2` |
