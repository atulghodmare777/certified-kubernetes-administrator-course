# Kustomize — Notes from Zero to Real Usage
---

## Table of Contents

1. [What is Kustomize and Why Does It Exist](#1-what-is-kustomize-and-why-does-it-exist)
2. [Installation](#2-installation)
3. [Your First Kustomize Setup — Managing a Directory](#3-your-first-kustomize-setup--managing-a-directory)
4. [Managing Subdirectories](#4-managing-subdirectories)
5. [Common Transformers](#5-common-transformers)
6. [Image Transformer](#6-image-transformer)
7. [Patches](#7-patches)
   - [Strategic Merge Patch](#71-strategic-merge-patch)
   - [JSON 6902 Patch](#72-json-6902-patch)
   - [Inline Patch vs Patch File](#73-inline-patch-vs-patch-file)
   - [Patches Directory](#74-patches-directory)
   - [Patches List](#75-patches-list)
8. [ConfigMap and Secret Generators](#8-configmap-and-secret-generators)
9. [Overlays — Managing Multiple Environments](#9-overlays--managing-multiple-environments)
10. [Components — Reusable Kustomize Modules](#10-components--reusable-kustomize-modules)
11. [Command Reference](#11-command-reference)
12. [Quick Reference Cheatsheet](#12-quick-reference-cheatsheet)

---

## 1. What is Kustomize and Why Does It Exist

### The Problem

You have a Kubernetes app. You want to deploy it to three environments — dev, staging, prod.

Your `deployment.yaml` looks like this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
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

Each environment needs different values:

| Setting    | Dev     | Staging        | Prod           |
| ---------- | ------- | -------------- | -------------- |
| Namespace  | dev     | staging        | prod           |
| Replicas   | 1       | 2              | 5              |
| Image tag  | latest  | v1.4.0-rc1     | v1.4.0         |

**Without Kustomize** — your options are:

- Copy the YAML 3 times and change values manually → files go out of sync, hard to manage
- Use `sed` in shell scripts to replace values → fragile, unreadable
- Write one giant YAML with everything hardcoded → not reusable at all

Any time you add a new label, fix a typo, or change a port — you do it in all three copies.

### The Solution — Kustomize

Kustomize lets you write your Kubernetes YAML **once** as a base, and then define only the **differences** for each environment as separate small files called overlays.

```
base YAML (written once)
    +
overlay for dev (only the differences)
    =
final YAML for dev cluster
```

```
base YAML (same)
    +
overlay for prod (only the differences)
    =
final YAML for prod cluster
```

### What Kustomize is NOT

- It is **not a templating engine** — there are no `{{ }}` or `${}` placeholders
- It is **not Helm** — it does not package and distribute charts
- It does **not touch your original files** — base files stay exactly as written

Everything in Kustomize is **plain YAML**. No new syntax to learn.

---

## 2. Installation

### Option 1 — Already inside kubectl (recommended for most users)

Kustomize has been **built into kubectl** since Kubernetes 1.14. If you have kubectl installed, you already have Kustomize.

```bash
kubectl version --client
```

Use it with:
```bash
kubectl apply -k <directory>
kubectl kustomize <directory>
```

### Option 2 — Standalone kustomize CLI

The standalone CLI has newer features and is updated more frequently than the kubectl-bundled version.

**On Linux:**
```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
```

**On macOS:**
```bash
brew install kustomize
```

**On Windows:**
```powershell
choco install kustomize
```

**Verify installation:**
```bash
kustomize version
```

### Which one to use?

Use `kubectl apply -k` for day-to-day use.
Use `kustomize build` when you want to preview the final YAML before applying.

---

## 3. Your First Kustomize Setup — Managing a Directory

### What is kustomization.yaml?

Every Kustomize setup starts with one mandatory file — `kustomization.yaml`.

This file is the **control file**. It tells Kustomize:
- Which YAML files to include
- What changes to apply on top of them

Without this file, Kustomize does nothing.

### Minimal Example

Create this folder structure:

```
myapp/
├── kustomization.yaml
├── deployment.yaml
└── service.yaml
```

**deployment.yaml**

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
          image: myrepo/myapp:v1.0
          ports:
            - containerPort: 8080
```

**service.yaml**

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

**kustomization.yaml**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

### Preview the output (does NOT apply anything)

```bash
kubectl kustomize ./myapp/
```

This prints the final merged YAML to your terminal. Use this before every apply to verify what will be sent to Kubernetes.

### Apply to the cluster

```bash
kubectl apply -k ./myapp/
```

### Delete from the cluster

```bash
kubectl delete -k ./myapp/
```

### What happened?

Right now the output is identical to the input — we added no changes yet.
The point of this step is to understand the basic wiring:

```
kustomization.yaml lists the files
        ↓
kustomize reads and merges them
        ↓
final YAML is produced
        ↓
kubectl sends it to the cluster
```

---

## 4. Managing Subdirectories

As your app grows, you may have many YAML files organized into subfolders.
Kustomize can reference files in subdirectories directly.

### Example — App with subfolders

```
myapp/
├── kustomization.yaml
├── deployments/
│   ├── app-deployment.yaml
│   └── worker-deployment.yaml
├── services/
│   └── app-service.yaml
└── configs/
    └── configmap.yaml
```

**kustomization.yaml**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployments/app-deployment.yaml
  - deployments/worker-deployment.yaml
  - services/app-service.yaml
  - configs/configmap.yaml
```

Kustomize follows the paths and reads each file. You can mix files from any subfolder freely.

### Referencing another Kustomize directory

You can also point to an entire directory that itself has a `kustomization.yaml`:

```yaml
resources:
  - ./another-kustomize-folder/     # Kustomize reads that folder's kustomization.yaml
  - deployment.yaml
```

This is how **overlays** reference a **base** — covered in detail in Section 9.

---

## 5. Common Transformers

Transformers are **built-in operations** in Kustomize that automatically apply a change
across **all resources** listed in your `kustomization.yaml`.
You declare them once and they apply everywhere — no need to touch individual files.

---

### 5.1 namespace — Set Namespace on All Resources

Sets the `metadata.namespace` field on every resource at once.

**Without Kustomize** — you'd have to add `namespace:` to every single YAML file.

**With Kustomize:**

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

namespace: production
```

Both the Deployment and the Service will have `namespace: production` in the final output,
even though neither file mentions a namespace.

---

### 5.2 namePrefix and nameSuffix — Add Prefix or Suffix to All Names

Automatically prepends or appends a string to `metadata.name` of every resource.

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

namePrefix: prod-
nameSuffix: -v2
```

A resource named `myapp` becomes `prod-myapp-v2` in the final output.

**Why use this?**
When you deploy the same app to multiple environments in the same cluster,
name collisions would otherwise occur. Adding `dev-` or `prod-` as a prefix makes every resource name unique.

---

### 5.3 commonLabels — Add Labels to All Resources

Adds the same set of labels to the `metadata.labels` of every resource.
For Deployments and Services it also adds labels to `spec.selector` and `spec.template.metadata.labels`.

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  env: production
  team: backend
  managed-by: kustomize
```

Every resource in the output will carry these three labels.

**Why use this?**
Labels are how you filter and query resources with `kubectl get`, and how tools
like Prometheus, Datadog, or ArgoCD identify and group resources.

---

### 5.4 commonAnnotations — Add Annotations to All Resources

Same idea as `commonLabels` but for annotations.

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonAnnotations:
  owner: "platform-team"
  contact: "platform@company.com"
  managed-by: "kustomize"
```

---

### Full Transformer Example Together

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

namespace: production
namePrefix: prod-

commonLabels:
  env: production
  app: myapp

commonAnnotations:
  owner: platform-team
```

**Preview the result:**
```bash
kubectl kustomize ./myapp/
```

The output will show both the Deployment and Service with:
- `namespace: production`
- `name: prod-myapp`
- Labels `env: production` and `app: myapp`
- Annotation `owner: platform-team`

None of the original YAML files were touched.

---

## 6. Image Transformer

The image transformer lets you change the **container image name, tag, or digest**
across all resources without touching any YAML file.

This is one of the most useful features in Kustomize for CI/CD pipelines.

### Syntax

```yaml
# kustomization.yaml
images:
  - name: <image-name-to-match>      # exact name as written in your deployment
    newName: <replacement-image>     # optional: change the image name
    newTag: <replacement-tag>        # optional: change the tag
    digest: <sha256:...>             # optional: pin to a specific digest instead of tag
```

### Example 1 — Change only the tag

Your `deployment.yaml` has:
```yaml
containers:
  - name: myapp
    image: myrepo/myapp:latest
```

**kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml

images:
  - name: myrepo/myapp
    newTag: "v1.5.0"
```

Output in final YAML:
```yaml
image: myrepo/myapp:v1.5.0
```

The deployment file still says `latest` — Kustomize changed it in the final output only.

---

### Example 2 — Change image name and tag

```yaml
images:
  - name: myrepo/myapp          # match this in deployment
    newName: gcr.io/myproject/myapp   # use this registry instead
    newTag: "v2.0.0"
```

Output:
```yaml
image: gcr.io/myproject/myapp:v2.0.0
```

---

### Example 3 — Multiple images in one deployment

```yaml
images:
  - name: myrepo/frontend
    newTag: "v3.1.0"
  - name: myrepo/backend
    newTag: "v2.5.0"
  - name: redis
    newTag: "7.2-alpine"
```

Each image is matched by name. Kustomize updates them independently.

---

### Why this is powerful in CI/CD

In your pipeline, after building and pushing an image:

```bash
# In your CI script
IMAGE_TAG=$(git rev-parse --short HEAD)

kustomize edit set image myrepo/myapp=myrepo/myapp:$IMAGE_TAG
kubectl apply -k .
```

`kustomize edit set image` updates `kustomization.yaml` automatically with the new tag.
Then you apply — the deployment gets the new image with zero manual file editing.

---

## 7. Patches

A **patch** is a file or inline block that modifies a specific field inside a specific resource.

Unlike transformers (which apply to all resources), patches **target one specific resource**
and change only the fields you specify. Everything else in that resource stays untouched.

---

### 7.1 Strategic Merge Patch

A **Strategic Merge Patch** looks like a partial copy of the resource you want to modify.
You write only the fields you want to change. Kustomize finds the matching resource
(by `kind` and `name`) and merges your patch into it.

**Rules:**
- The patch must have the same `apiVersion`, `kind`, and `metadata.name` as the resource
- You only include the fields you want to change
- Everything not mentioned in the patch stays as-is from the base

**Example — change replicas and add resource limits**

Base `deployment.yaml`:
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
          image: myrepo/myapp:v1.0
          ports:
            - containerPort: 8080
```

Patch file `increase-replicas.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp           # must match the base deployment name exactly
spec:
  replicas: 5           # only this changes
  template:
    spec:
      containers:
        - name: myapp   # must match the container name exactly
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

**kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml

patches:
  - path: increase-replicas.yaml
```

**Final output after build:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5           # ← changed by patch
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
          image: myrepo/myapp:v1.0   # ← unchanged, came from base
          ports:
            - containerPort: 8080    # ← unchanged, came from base
          resources:                 # ← added by patch
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

The base file itself is never modified.

---

### 7.2 JSON 6902 Patch

A **JSON 6902 Patch** is more precise. Instead of writing a partial YAML,
you write explicit operations with a path to the exact field.

This is useful when:
- You want to add a new item to a list (like adding an env variable)
- You want to remove a specific field
- The Strategic Merge Patch is being too aggressive (merging when you want to replace)

**Operations:**

| Operation | What it Does                               |
| --------- | ------------------------------------------ |
| `replace` | Replace an existing value at a given path  |
| `add`     | Add a new key or append to a list          |
| `remove`  | Remove a field or item from a list         |

**Path syntax:**
Paths use `/` as separator, following the JSON Pointer format.

```
/spec/replicas                           → spec.replicas
/spec/template/spec/containers/0/image   → first container's image
/metadata/labels/env                     → labels.env
```

**Example 1 — Replace replicas**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml

patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |
      - op: replace
        path: /spec/replicas
        value: 3
```

**Example 2 — Add an environment variable to a container**

```yaml
patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |
      - op: add
        path: /spec/template/spec/containers/0/env
        value:
          - name: APP_ENV
            value: "production"
          - name: LOG_LEVEL
            value: "warn"
```

**Example 3 — Remove a field**

```yaml
patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |
      - op: remove
        path: /spec/template/spec/containers/0/resources/limits
```

**Example 4 — Multiple operations in one patch**

```yaml
patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |
      - op: replace
        path: /spec/replicas
        value: 5
      - op: replace
        path: /spec/template/spec/containers/0/image
        value: "myrepo/myapp:v2.0.0"
      - op: add
        path: /spec/template/spec/containers/0/env
        value:
          - name: ENV
            value: "production"
```

---

### 7.3 Inline Patch vs Patch File

You have two ways to write a patch:

**Way 1 — Inline (inside kustomization.yaml)**

The patch is written directly inside `kustomization.yaml` using the `patch: |` block.
Good for small, simple patches.

```yaml
patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |
      - op: replace
        path: /spec/replicas
        value: 3
```

**Way 2 — Separate file (recommended for anything bigger)**

The patch lives in its own `.yaml` file and is referenced by path.
Good for strategic merge patches and anything complex.

```yaml
patches:
  - path: patches/increase-replicas.yaml
  - path: patches/add-env-vars.yaml
```

**When to use which:**

| | Inline | File |
|---|---|---|
| Simple one-liner changes | ✅ Good | Works too |
| Strategic merge patches | Not ideal | ✅ Recommended |
| Complex multi-field patches | Hard to read | ✅ Recommended |
| Git diffs and reviews | Hard to read | ✅ Easier to review |

---

### 7.4 Patches Directory

When you have many patches, keep them in a dedicated `patches/` subdirectory
to keep the folder clean and organized.

**Folder structure:**
```
myapp/
├── kustomization.yaml
├── deployment.yaml
├── service.yaml
└── patches/
    ├── replica-patch.yaml
    ├── resource-limits-patch.yaml
    └── env-vars-patch.yaml
```

**kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

patches:
  - path: patches/replica-patch.yaml
  - path: patches/resource-limits-patch.yaml
  - path: patches/env-vars-patch.yaml
```

**patches/replica-patch.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
```

**patches/resource-limits-patch.yaml:**
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

**patches/env-vars-patch.yaml:**
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
          env:
            - name: APP_ENV
              value: production
            - name: LOG_LEVEL
              value: warn
```

Kustomize applies all three patches in order. The final Deployment has all three sets of changes merged together.

---

### 7.5 Patches List — Targeting Multiple Resources with One Entry

When you have a patch that applies to **multiple resources**, you can use a `target`
with broader selectors instead of writing separate patch entries.

**Target by kind only (applies to all Deployments):**

```yaml
patches:
  - target:
      kind: Deployment      # applies to ALL Deployments in the resource list
    patch: |
      - op: add
        path: /spec/template/metadata/annotations
        value:
          prometheus.io/scrape: "true"
          prometheus.io/port: "8080"
```

**Target by label selector:**

```yaml
patches:
  - target:
      kind: Deployment
      labelSelector: "env=production"   # only Deployments with this label
    patch: |
      - op: replace
        path: /spec/replicas
        value: 3
```

**Target by name + kind (most specific):**

```yaml
patches:
  - target:
      kind: Deployment
      name: myapp           # only the Deployment named myapp
    patch: |
      - op: replace
        path: /spec/replicas
        value: 5
  - target:
      kind: Deployment
      name: worker          # only the Deployment named worker
    patch: |
      - op: replace
        path: /spec/replicas
        value: 2
```

This is a **patches list** — multiple patch entries, each targeting different resources.
They are all processed in the order listed.

---

## 8. ConfigMap and Secret Generators

Generators let Kustomize **create** Kubernetes resources for you from plain files
or key-value pairs — without you having to write the full YAML for them.

---

### 8.1 configMapGenerator

Instead of writing a ConfigMap YAML by hand, you can generate it from:
- Literal key=value pairs
- A `.env` file
- A plain file (the file content becomes the ConfigMap value)

**From literals:**

```yaml
configMapGenerator:
  - name: app-config
    literals:
      - APP_ENV=production
      - LOG_LEVEL=warn
      - DB_PORT=5432
```

This generates:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-7f4k9m2   # ← hash appended automatically
data:
  APP_ENV: production
  LOG_LEVEL: warn
  DB_PORT: "5432"
```

**From a .env file:**

Create `app.env`:
```
APP_ENV=production
LOG_LEVEL=warn
DB_HOST=postgres.prod.svc.cluster.local
DB_PORT=5432
```

```yaml
configMapGenerator:
  - name: app-config
    envs:
      - app.env
```

**From a plain file:**

Create `nginx.conf`:
```
server {
  listen 80;
  server_name myapp.com;
}
```

```yaml
configMapGenerator:
  - name: nginx-config
    files:
      - nginx.conf
```

The file content becomes a key in the ConfigMap where the key is the filename.

**Combining all three:**
```yaml
configMapGenerator:
  - name: app-config
    literals:
      - APP_ENV=production
    envs:
      - app.env
    files:
      - nginx.conf
```

### The Hash — Why It Exists and How to Disable It

Kustomize appends a hash to the ConfigMap name (e.g., `app-config-7f4k9m2`).

**Why?** When you change the ConfigMap content, the hash changes, which changes
the name, which forces Kubernetes to do a **rolling restart** of all Pods that use it.
Without this, updating a ConfigMap would not restart Pods — they'd keep using the old values.

**To disable the hash** (if you manage restarts yourself):
```yaml
configMapGenerator:
  - name: app-config
    options:
      disableNameSuffixHash: true
    literals:
      - APP_ENV=production
```

**Using the generated ConfigMap in your Deployment:**
```yaml
# deployment.yaml
spec:
  containers:
    - name: myapp
      image: myrepo/myapp:v1.0
      envFrom:
        - configMapRef:
            name: app-config    # write the base name — Kustomize resolves the hash automatically
```

You write `app-config` in the deployment — Kustomize automatically rewrites it to
`app-config-7f4k9m2` (or whatever the current hash is) in the final output.

---

### 8.2 secretGenerator

Same as `configMapGenerator` but creates a Kubernetes Secret.
Values are base64 encoded automatically by Kubernetes.

```yaml
secretGenerator:
  - name: db-credentials
    literals:
      - DB_USER=admin
      - DB_PASSWORD=supersecretpassword
    type: Opaque
```

From a file (e.g., TLS cert):
```yaml
secretGenerator:
  - name: tls-secret
    files:
      - tls.crt
      - tls.key
    type: kubernetes.io/tls
```

> ⚠️ **Never commit actual passwords to Git in plain text.**
> Use **Sealed Secrets** or **SOPS** to encrypt secrets before committing.
> The `secretGenerator` is fine for local testing but not for production Git workflows
> without encryption.

---

## 9. Overlays — Managing Multiple Environments

This is the most important concept in Kustomize. Everything you learned so far
builds up to this.

### The Pattern

```
project/
├── base/               ← common YAML, written once
└── overlays/
    ├── dev/            ← what changes for dev
    ├── staging/        ← what changes for staging
    └── prod/           ← what changes for prod
```

**Base** = your neutral, shared Kubernetes config. No environment-specific values.
**Overlay** = a thin layer on top of base that defines only what is different for that environment.

---

### Step 1 — Write the Base

```
base/
├── kustomization.yaml
├── deployment.yaml
└── service.yaml
```

**base/deployment.yaml**
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

**base/service.yaml**
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

**base/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

The base has no namespace, no env-specific values. It is just the plain, reusable config.

---

### Step 2 — Write the Dev Overlay

```
overlays/dev/
├── kustomization.yaml
└── patches/
    └── dev-patch.yaml
```

**overlays/dev/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base          # ← this points to the base directory

namespace: dev
namePrefix: dev-

commonLabels:
  env: dev

images:
  - name: myrepo/myapp
    newTag: "latest"

patches:
  - path: patches/dev-patch.yaml
```

**overlays/dev/patches/dev-patch.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1           # dev runs 1 replica
```

---

### Step 3 — Write the Staging Overlay

```
overlays/staging/
├── kustomization.yaml
└── patches/
    └── staging-patch.yaml
```

**overlays/staging/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: staging
namePrefix: staging-

commonLabels:
  env: staging

images:
  - name: myrepo/myapp
    newTag: "v1.4.0-rc1"      # release candidate for testing

patches:
  - path: patches/staging-patch.yaml
```

**overlays/staging/patches/staging-patch.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
```

---

### Step 4 — Write the Prod Overlay

```
overlays/prod/
├── kustomization.yaml
└── patches/
    ├── prod-replica-patch.yaml
    └── prod-resources-patch.yaml
```

**overlays/prod/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: prod
namePrefix: prod-

commonLabels:
  env: prod

commonAnnotations:
  owner: platform-team

images:
  - name: myrepo/myapp
    newTag: "v1.4.0"          # stable pinned version for prod

patches:
  - path: patches/prod-replica-patch.yaml
  - path: patches/prod-resources-patch.yaml
```

**overlays/prod/patches/prod-replica-patch.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
```

**overlays/prod/patches/prod-resources-patch.yaml**
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

### Full Directory Structure

```
project/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── dev-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── staging-patch.yaml
    └── prod/
        ├── kustomization.yaml
        └── patches/
            ├── prod-replica-patch.yaml
            └── prod-resources-patch.yaml
```

---

### What Each Environment Gets

| Setting         | Dev       | Staging        | Prod            |
| --------------- | --------- | -------------- | --------------- |
| Namespace       | `dev`     | `staging`      | `prod`          |
| Name prefix     | `dev-`    | `staging-`     | `prod-`         |
| Replicas        | 1         | 2              | 5               |
| Image tag       | `latest`  | `v1.4.0-rc1`  | `v1.4.0`        |
| Resource limits | ❌ None   | ❌ None        | ✅ Set          |
| Annotations     | ❌ None   | ❌ None        | ✅ owner label  |

All from the **same base**. Zero duplication.

---

### Preview and Apply Each Environment

```bash
# Always preview before applying
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

## 10. Components — Reusable Kustomize Modules

A **Component** is a reusable piece of Kustomize config — like a plugin that you can
switch on or off per environment.

### Problem it Solves

Imagine you have a monitoring setup (ServiceMonitor, labels, annotations) that you want
in staging and prod but NOT in dev.

You could copy the monitoring config into both staging and prod overlays — but then
you have duplication again.

**With Components**, you define the monitoring config once in a shared location
and simply include it in the overlays that need it.

---

### Step 1 — Create the Component

```
components/
└── monitoring/
    ├── kustomization.yaml
    └── servicemonitor.yaml
```

**components/monitoring/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1   # ← note: v1alpha1 for components
kind: Component                                 # ← note: kind is Component not Kustomization

resources:
  - servicemonitor.yaml

commonLabels:
  prometheus.io/scrape: "true"

commonAnnotations:
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics"
```

**components/monitoring/servicemonitor.yaml**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

---

### Step 2 — Include the Component in the Overlays That Need It

**overlays/staging/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: staging

components:
  - ../../components/monitoring    # ← monitoring added for staging
```

**overlays/prod/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: prod

components:
  - ../../components/monitoring    # ← monitoring added for prod
```

**overlays/dev/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: dev

# no components section — dev does not get monitoring
```

---

### Another Component Example — External Secrets (only in prod)

```
components/
└── external-secrets/
    ├── kustomization.yaml
    └── externalsecret.yaml
```

**components/external-secrets/kustomization.yaml**
```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - externalsecret.yaml
```

**components/external-secrets/externalsecret.yaml**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: myapp-secret
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: myapp/prod
        property: db_password
```

In the prod overlay:
```yaml
components:
  - ../../components/monitoring
  - ../../components/external-secrets   # only prod uses Vault secrets
```

---

### Component vs Overlay — When to Use Which

| | Overlay | Component |
|---|---|---|
| Purpose | Full environment config (dev/staging/prod) | Optional feature that toggles on/off |
| Has its own base | ✅ References base with `resources` | ❌ Cannot reference a base |
| Can be included in overlays | N/A | ✅ Yes, with `components:` |
| Examples | dev config, prod config | monitoring, external secrets, HPA, ingress |

---

## 11. Command Reference

| Command | What it Does |
|---|---|
| `kubectl kustomize <dir>` | Build and **print** the final YAML — does NOT apply to cluster |
| `kubectl apply -k <dir>` | Build and **apply** to cluster |
| `kubectl delete -k <dir>` | Build and **delete** all those resources from cluster |
| `kubectl diff -k <dir>` | Show what would **change** if you applied — compares cluster state vs kustomize output |
| `kustomize build <dir>` | Same as `kubectl kustomize` using standalone CLI |
| `kustomize build <dir> \| kubectl apply -f -` | Build with standalone CLI then pipe to kubectl |
| `kustomize edit set image <name>=<name>:<tag>` | Auto-update the image tag in `kustomization.yaml` |
| `kustomize version` | Show installed kustomize version |

### Recommended workflow before every apply

```bash
# Step 1 — Preview what will be sent to Kubernetes
kubectl kustomize ./overlays/prod/

# Step 2 — See what will change vs what is already in the cluster
kubectl diff -k ./overlays/prod/

# Step 3 — Apply only after you've reviewed
kubectl apply -k ./overlays/prod/
```

---

## 12. Quick Reference Cheatsheet

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# ── RESOURCES ──────────────────────────────────────────
resources:
  - deployment.yaml             # include a file
  - service.yaml
  - ../../base                  # include another kustomize directory (base)

# ── TRANSFORMERS ───────────────────────────────────────
namespace: prod                 # set namespace on all resources

namePrefix: prod-               # prepend to all resource names
nameSuffix: -v2                 # append to all resource names

commonLabels:
  env: prod
  app: myapp

commonAnnotations:
  owner: platform-team

# ── IMAGE TRANSFORMER ──────────────────────────────────
images:
  - name: myrepo/myapp          # image name to match in manifests
    newName: gcr.io/proj/myapp  # optional: change registry/name
    newTag: "v1.0.0"            # change tag
    # digest: sha256:abc123     # or pin by digest

# ── PATCHES ────────────────────────────────────────────
patches:
  # Strategic merge patch from file
  - path: patches/replica-patch.yaml

  # Inline JSON 6902 patch
  - target:
      kind: Deployment
      name: myapp
    patch: |
      - op: replace
        path: /spec/replicas
        value: 3
      - op: add
        path: /spec/template/spec/containers/0/env
        value:
          - name: ENV
            value: "production"

# ── GENERATORS ─────────────────────────────────────────
configMapGenerator:
  - name: app-config
    literals:
      - APP_ENV=production
    envs:
      - app.env                 # from .env file
    files:
      - nginx.conf              # file content as ConfigMap value
    options:
      disableNameSuffixHash: true   # disable the auto hash

secretGenerator:
  - name: db-secret
    literals:
      - DB_USER=admin
      - DB_PASSWORD=secret
    type: Opaque

# ── COMPONENTS ─────────────────────────────────────────
components:
  - ../../components/monitoring
  - ../../components/external-secrets
```
