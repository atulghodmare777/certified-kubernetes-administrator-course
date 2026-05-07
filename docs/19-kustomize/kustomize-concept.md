# Kustomize — Complete Notes for Beginners

> **Prerequisites:** You should already know Kubernetes basics — Pods, Deployments, Services, ConfigMaps, Namespaces, and how to write YAML manifests. Kustomize builds on top of that knowledge.

---

## Table of Contents
1. [What is Kustomize?](#1-what-is-kustomize)
2. [Why Use Kustomize?](#2-why-use-kustomize)
3. [Kustomize vs Helm](#3-kustomize-vs-helm)
4. [Core Concepts](#4-core-concepts)
5. [Kustomize File Structure](#5-kustomize-file-structure)
6. [Key Functionalities](#6-key-functionalities)
7. [Practical Examples](#7-practical-examples)
8. [Real-World Multi-Environment Setup](#8-real-world-multi-environment-setup)
9. [Useful Commands](#9-useful-commands)

---

## 1. What is Kustomize?

**Kustomize** is a tool that lets you customize Kubernetes YAML configuration files without modifying the original files.

It works by taking your **base** Kubernetes manifests and applying **patches/overrides** on top of them to produce the final configuration.

Think of it like this:

```
Base Kubernetes YAML
(original, untouched)
        +
Kustomization overrides
(your changes layered on top)
        =
Final YAML sent to Kubernetes
(merged result)
```

### Built into kubectl

Kustomize is **built directly into kubectl** since Kubernetes 1.14. You do not need to install anything extra.

```bash
kubectl apply -k ./my-kustomize-folder/
```

You can also use the standalone `kustomize` CLI for more control:
```bash
kustomize build ./my-kustomize-folder/ | kubectl apply -f -
```

---

## 2. Why Use Kustomize?

### The Problem Without Kustomize

Imagine you have a simple Nginx deployment YAML:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
```

Now you need to deploy this to **three environments** — dev, staging, and prod.

**Without Kustomize**, you end up doing one of these:
- Copy the YAML three times and change values manually → duplicate files, gets out of sync
- Use `sed` commands to find-and-replace values in scripts → fragile, hard to maintain
- Create one giant YAML with everything hardcoded → not reusable at all

Any change to the base configuration (e.g., adding a new label or updating the image) means updating all three copies.

### How Kustomize Solves This

With Kustomize, you write the base YAML **once** and define only the **differences** per environment:

```
base/                   ← write once, the common config
  deployment.yaml
  service.yaml

overlays/
  dev/                  ← only what's different for dev
  staging/              ← only what's different for staging
  prod/                 ← only what's different for prod
```

- Dev → replicas: 1, image: nginx:latest
- Staging → replicas: 2, image: nginx:1.21
- Prod → replicas: 5, image: nginx:1.21, resource limits added

All three share the **same base**. You only write and maintain the differences.

### Summary — Why Kustomize?
- No duplication of YAML files across environments
- No templating language to learn (it's just YAML)
- Original files stay clean and untouched
- Works natively with `kubectl` — no extra tools needed
- Easy to review changes — you can see exactly what differs per environment
- Great for GitOps — everything is plain YAML in Git

---

## 3. Kustomize vs Helm

| | Kustomize | Helm |
|---|---|---|
| Approach | Patches on top of plain YAML | Templates with a custom language |
| Learning curve | Low — just YAML | Medium — needs Go templating knowledge |
| Config format | Plain YAML | YAML + `{{ .Values.xxx }}` templating |
| Built into kubectl | ✅ Yes | ❌ No, separate install |
| Packaging & sharing | ❌ Not designed for it | ✅ Charts can be packaged and shared |
| Best for | Managing your own app configs across envs | Installing third-party apps (Prometheus, WordPress, etc.) |
| Secret management | Basic (use with Sealed Secrets / SOPS) | Basic (same limitation) |

> **When to use which:**
> Use **Helm** to install third-party software (databases, monitoring, ingress controllers).
> Use **Kustomize** to manage your own application deployments across environments.
> They can also be used **together** — Helm to install, Kustomize to patch on top.

---

## 4. Core Concepts

### 4.1 kustomization.yaml

This is the **heart of Kustomize**. Every Kustomize folder must have a file named `kustomization.yaml`. This file tells Kustomize:
- Which resources (YAML files) to include
- What transformations or patches to apply
- What generators to run

```yaml
# kustomization.yaml — minimum example
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

---

### 4.2 Base

A **Base** is a directory containing the original, common Kubernetes manifests plus a `kustomization.yaml` that references them. It is the starting point — the config that applies everywhere.

```
base/
├── kustomization.yaml
├── deployment.yaml
└── service.yaml
```

The base should have no environment-specific values. It should represent the neutral, shared config.

---

### 4.3 Overlay

An **Overlay** is a directory that points to a base and defines changes on top of it. Each environment (dev, staging, prod) gets its own overlay.

```
overlays/
├── dev/
│   └── kustomization.yaml     ← points to base, adds dev-specific changes
├── staging/
│   └── kustomization.yaml     ← points to base, adds staging-specific changes
└── prod/
    └── kustomization.yaml     ← points to base, adds prod-specific changes
```

An overlay's `kustomization.yaml` references the base using a relative path:

```yaml
# overlays/dev/kustomization.yaml
resources:
  - ../../base          # ← points to the base directory
```

---

### 4.4 Patch

A **Patch** is a YAML snippet that modifies a specific field in a resource. You don't rewrite the whole file — just the parts that need to change.

There are two types of patches:
- **Strategic Merge Patch** — looks like a partial YAML of the resource, Kustomize merges it
- **JSON 6902 Patch** — uses JSON patch operations (`add`, `replace`, `remove`) for precise control

---

### 4.5 Transformer

A **Transformer** is a built-in Kustomize feature that automatically applies a change across all resources. Examples:
- Add a label to every resource
- Add a name prefix to every resource
- Set the namespace for every resource

---

### 4.6 Generator

A **Generator** creates a new Kubernetes resource from non-YAML sources. Examples:
- Create a ConfigMap from a `.properties` file or `.env` file
- Create a Secret from literal values or files

---

## 5. Kustomize File Structure

### Minimal Structure (single environment)
```
my-app/
├── kustomization.yaml
├── deployment.yaml
└── service.yaml
```

### Standard Multi-Environment Structure
```
my-app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── prod/
        ├── kustomization.yaml
        ├── replica-patch.yaml
        └── resource-limits-patch.yaml
```

---

## 6. Key Functionalities

### 6.1 `resources` — Include YAML Files

Tells Kustomize which files or directories to include.

```yaml
resources:
  - deployment.yaml
  - service.yaml
  - ../../base            # reference another kustomize directory (base)
```

---

### 6.2 `namePrefix` and `nameSuffix` — Add Prefix/Suffix to All Names

Automatically prepends or appends a string to the `metadata.name` of every resource.

```yaml
namePrefix: dev-
nameSuffix: -v2
```

A resource named `nginx` becomes `dev-nginx-v2` automatically across all included files.

**Use case:** Prevent name collisions when deploying multiple versions to the same namespace.

---

### 6.3 `namespace` — Set Namespace for All Resources

Sets the namespace on every resource at once without editing each file.

```yaml
namespace: dev
```

**Use case:** Your base files may have no namespace set. Each overlay applies its own.

---

### 6.4 `commonLabels` — Add Labels to Everything

Adds labels to every resource (and to selector fields in Deployments/Services).

```yaml
commonLabels:
  app: nginx
  env: dev
  team: backend
```

**Use case:** Standardize labels across all resources for filtering and observability.

---

### 6.5 `commonAnnotations` — Add Annotations to Everything

Adds annotations to every resource.

```yaml
commonAnnotations:
  owner: "platform-team"
  managed-by: "kustomize"
```

---

### 6.6 `images` — Override Image Name or Tag

Change the container image or tag across all Deployments/StatefulSets without touching the YAML files.

```yaml
images:
  - name: nginx                  # match this image name in your manifests
    newTag: "1.25"               # change just the tag
  - name: myapp
    newName: myrepo/myapp        # change the image name
    newTag: "v2.1.0"             # and the tag
```

**Use case:** Each environment uses a different image tag. Dev uses `latest`, prod uses a pinned version.

---

### 6.7 `patches` — Strategic Merge Patch

Apply a partial YAML that gets merged into the matching resource.

```yaml
patches:
  - path: replica-patch.yaml
```

The patch file looks like a partial version of the resource you want to modify:

```yaml
# replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx           # must match the resource you're targeting
spec:
  replicas: 5           # only this field is changed — everything else stays
```

Kustomize finds the Deployment named `nginx` and merges this patch into it.

---

### 6.8 `patches` — JSON 6902 Patch

More surgical — you specify the exact path to modify using JSON Patch operations.

```yaml
patches:
  - target:
      kind: Deployment
      name: nginx
    patch: |
      - op: replace
        path: /spec/replicas
        value: 3
      - op: add
        path: /spec/template/spec/containers/0/env
        value:
          - name: ENV
            value: "production"
```

Operations available:
| Operation | What it Does |
|---|---|
| `replace` | Replace the value at the given path |
| `add` | Add a new field or array item |
| `remove` | Remove a field or array item |

---

### 6.9 `configMapGenerator` — Generate ConfigMaps

Generate a ConfigMap from files, `.env` files, or literal values.

```yaml
configMapGenerator:
  - name: app-config
    literals:
      - APP_ENV=production
      - LOG_LEVEL=info
    files:
      - config.properties
    envs:
      - app.env
```

Kustomize automatically **appends a hash** to the ConfigMap name (e.g., `app-config-7d8f9g2`). When the content changes, the hash changes, which forces a rolling update of all Pods using that ConfigMap.

To disable the hash:
```yaml
configMapGenerator:
  - name: app-config
    options:
      disableNameSuffixHash: true
    literals:
      - APP_ENV=production
```

---

### 6.10 `secretGenerator` — Generate Secrets

Generate a Kubernetes Secret from literals or files.

```yaml
secretGenerator:
  - name: db-credentials
    literals:
      - DB_USER=admin
      - DB_PASSWORD=supersecret
    type: Opaque
```

> ⚠️ **Important:** Never commit actual secrets in plain text to Git. Use this with tools like **Sealed Secrets** or **SOPS** to encrypt secret values before committing.

---

### 6.11 `vars` — Reference Field Values Across Resources

Allows you to reference a value from one resource and use it in another. For example, referencing a Service name in a Deployment's environment variable.

> Note: `vars` is deprecated in newer versions of Kustomize. Prefer using `replacements` for this in Kustomize v4.1+.

---

### 6.12 `components` — Reusable Kustomize Modules

A **Component** is like a reusable plugin — a chunk of Kustomize config that can be shared and included in multiple overlays.

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - servicemonitor.yaml

commonLabels:
  prometheus: "true"
```

Include it in an overlay:
```yaml
# overlays/prod/kustomization.yaml
components:
  - ../../components/monitoring
```

**Use case:** You want to add monitoring config to prod and staging but not dev. Define it once as a component, include it only where needed.

---

## 7. Practical Examples

---

### Example 1 — Simple Single App Setup

**Goal:** Deploy an Nginx app with Kustomize.

**File structure:**
```
nginx-app/
├── kustomization.yaml
├── deployment.yaml
└── service.yaml
```

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - containerPort: 80
```

**service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

**kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

namespace: my-app

commonLabels:
  app: nginx
  managed-by: kustomize
```

**Build and preview (does not apply, just shows output):**
```bash
kubectl kustomize ./nginx-app/
```

**Apply to cluster:**
```bash
kubectl apply -k ./nginx-app/
```

---

### Example 2 — Image Override Per Environment

**Goal:** Use the same deployment but different image tags for dev and prod.

**base/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myrepo/myapp:latest
```

**base/kustomization.yaml:**
```yaml
resources:
  - deployment.yaml
```

**overlays/dev/kustomization.yaml:**
```yaml
resources:
  - ../../base

namespace: dev

images:
  - name: myrepo/myapp
    newTag: "latest"
```

**overlays/prod/kustomization.yaml:**
```yaml
resources:
  - ../../base

namespace: prod

images:
  - name: myrepo/myapp
    newTag: "v1.5.2"       # pinned, stable version for prod
```

**Apply dev:**
```bash
kubectl apply -k ./overlays/dev/
```

**Apply prod:**
```bash
kubectl apply -k ./overlays/prod/
```

---

### Example 3 — Strategic Merge Patch (Change Replicas and Resources)

**Goal:** Prod needs more replicas and resource limits that dev doesn't need.

**base/kustomization.yaml:**
```yaml
resources:
  - deployment.yaml
  - service.yaml
```

**overlays/prod/kustomization.yaml:**
```yaml
resources:
  - ../../base

namespace: prod

patches:
  - path: prod-patch.yaml
```

**overlays/prod/prod-patch.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp         # must match the name in base
spec:
  replicas: 5         # override replicas
  template:
    spec:
      containers:
        - name: myapp
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

Kustomize **merges** this patch into the base Deployment. Only the fields in the patch change — everything else from the base remains.

---

### Example 4 — ConfigMap Generator

**Goal:** Create a ConfigMap from an env file and inject it into the app.

**app.env:**
```
APP_ENV=production
LOG_LEVEL=warn
DB_HOST=postgres.prod.svc.cluster.local
DB_PORT=5432
```

**kustomization.yaml:**
```yaml
resources:
  - deployment.yaml

configMapGenerator:
  - name: app-config
    envs:
      - app.env
```

**Using the ConfigMap in deployment.yaml:**
```yaml
spec:
  containers:
    - name: myapp
      image: myrepo/myapp:v1.0
      envFrom:
        - configMapRef:
            name: app-config      # Kustomize will resolve the hashed name automatically
```

When you change `app.env`, Kustomize generates a new ConfigMap with a new hash, which triggers a rolling restart of the Deployment automatically.

---

### Example 5 — JSON 6902 Patch (Add an Environment Variable)

**Goal:** Add an environment variable to the container only in production.

**overlays/prod/kustomization.yaml:**
```yaml
resources:
  - ../../base

patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |
      - op: add
        path: /spec/template/spec/containers/0/env
        value:
          - name: ENVIRONMENT
            value: "production"
          - name: FEATURE_FLAG
            value: "enabled"
```

---

## 8. Real-World Multi-Environment Setup

This is the most common pattern used in production GitOps workflows.

### Full Structure
```
my-app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── replica-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── replica-patch.yaml
    └── prod/
        ├── kustomization.yaml
        └── patches/
            ├── replica-patch.yaml
            └── resources-patch.yaml
```

---

### base/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myrepo/myapp:latest
          ports:
            - containerPort: 8080
```

### base/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### base/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

---

### overlays/dev/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: dev

namePrefix: dev-

images:
  - name: myrepo/myapp
    newTag: "latest"

commonLabels:
  env: dev

patches:
  - path: patches/replica-patch.yaml
```

### overlays/dev/patches/replica-patch.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
```

---

### overlays/staging/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: staging

namePrefix: staging-

images:
  - name: myrepo/myapp
    newTag: "v1.5.0-rc1"

commonLabels:
  env: staging

patches:
  - path: patches/replica-patch.yaml
```

### overlays/staging/patches/replica-patch.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
```

---

### overlays/prod/kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: prod

namePrefix: prod-

images:
  - name: myrepo/myapp
    newTag: "v1.5.0"

commonLabels:
  env: prod

commonAnnotations:
  owner: "platform-team"

patches:
  - path: patches/replica-patch.yaml
  - path: patches/resources-patch.yaml
```

### overlays/prod/patches/replica-patch.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
```

### overlays/prod/patches/resources-patch.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

---

### What Each Environment Gets

| Setting | Dev | Staging | Prod |
|---|---|---|---|
| Namespace | `dev` | `staging` | `prod` |
| Name prefix | `dev-` | `staging-` | `prod-` |
| Replicas | 1 | 2 | 5 |
| Image tag | `latest` | `v1.5.0-rc1` | `v1.5.0` |
| Resource limits | ❌ No | ❌ No | ✅ Yes |

All from the **same base** — zero duplication.

---

### Apply Each Environment

```bash
# Preview without applying
kubectl kustomize ./overlays/dev/
kubectl kustomize ./overlays/staging/
kubectl kustomize ./overlays/prod/

# Apply to cluster
kubectl apply -k ./overlays/dev/
kubectl apply -k ./overlays/staging/
kubectl apply -k ./overlays/prod/

# Delete resources
kubectl delete -k ./overlays/dev/
```

---

## 9. Useful Commands

| Command | What it Does |
|---|---|
| `kubectl kustomize <dir>` | Build and print the final YAML (does NOT apply) |
| `kubectl apply -k <dir>` | Build and apply to the cluster |
| `kubectl delete -k <dir>` | Delete all resources defined by the kustomization |
| `kubectl diff -k <dir>` | Show diff between current cluster state and kustomize output |
| `kustomize build <dir>` | Same as kubectl kustomize, using standalone CLI |
| `kustomize build <dir> \| kubectl apply -f -` | Build with standalone CLI and pipe to kubectl |
| `kustomize version` | Show installed kustomize version |

---

## Quick Reference — kustomization.yaml Fields

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Include files or other kustomize directories
resources:
  - deployment.yaml
  - service.yaml
  - ../../base

# Set namespace for all resources
namespace: prod

# Add prefix to all resource names
namePrefix: prod-

# Add suffix to all resource names
nameSuffix: -v2

# Add labels to all resources
commonLabels:
  env: prod
  app: myapp

# Add annotations to all resources
commonAnnotations:
  owner: platform-team

# Override image tags
images:
  - name: myrepo/myapp
    newTag: "v1.0.0"

# Apply strategic merge patches
patches:
  - path: my-patch.yaml
  - target:             # inline JSON patch
      kind: Deployment
      name: myapp
    patch: |
      - op: replace
        path: /spec/replicas
        value: 3

# Generate ConfigMaps
configMapGenerator:
  - name: app-config
    literals:
      - KEY=value
    envs:
      - app.env
    files:
      - config.properties

# Generate Secrets
secretGenerator:
  - name: db-secret
    literals:
      - DB_PASS=secret
    type: Opaque

# Include reusable components
components:
  - ../../components/monitoring
```

---

## Key Takeaways

- Kustomize is **not a templating tool** — there are no `{{ }}` placeholders. Everything is plain YAML.
- The `kustomization.yaml` file is **mandatory** in every Kustomize directory.
- **Base** = the common config. **Overlay** = environment-specific changes on top.
- Patches only contain the **fields you want to change** — the rest comes from the base.
- `kubectl kustomize <dir>` lets you **preview** the final YAML before applying — always do this first.
- Kustomize is **built into kubectl** — no extra install needed for basic use.
- Store everything in **Git** — bases, overlays, and patches. This is the GitOps way.
