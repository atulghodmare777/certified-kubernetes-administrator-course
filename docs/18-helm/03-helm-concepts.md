# Helm Core Concepts — Charts, Release & Revision

---

## 1. Chart

### What is a Chart?
A **Chart** is a Helm package. It is a collection of files that describe a set of Kubernetes resources needed to run an application, tool, or service inside a cluster.

Think of a Chart the same way you think of:
- An **APT package** in Ubuntu
- A **Homebrew formula** on macOS
- A **Docker image** for containers

A Chart bundles everything your application needs — Deployments, Services, ConfigMaps, Ingress, Secrets — all written as reusable templates.

### What is Inside a Chart?
```
my-chart/
├── Chart.yaml          # Metadata about the chart (name, version, description)
├── values.yaml         # Default configuration values
├── charts/             # Sub-charts / dependencies
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl    # Reusable template helpers
└── crds/               # Custom Resource Definitions (installed first)
```

### What Does Chart.yaml Contain?
```yaml
apiVersion: v2
name: wordpress
description: A chart to deploy WordPress on Kubernetes
type: application
version: 1.0.0        # Version of the chart itself
appVersion: "6.4.0"   # Version of the actual app (WordPress)
```

### What is the Use of a Chart?
- Packages all Kubernetes manifests into one installable unit
- Makes applications **portable** — same chart works across dev, staging, prod
- Allows **configuration** through `values.yaml` without changing the templates
- Can be shared via a **Chart Repository** so anyone can pull and install it
- Supports **versioning** so you can upgrade or rollback to a specific chart version

### Types of Charts
| Type | Description |
|---|---|
| `application` | A standard chart that deploys actual resources into the cluster |
| `library` | Contains only reusable templates, cannot be installed directly |

---

## 2. Release

### What is a Release?
A **Release** is a running instance of a Chart installed into your Kubernetes cluster.

When you run `helm install`, Helm takes a Chart and deploys it — that deployed instance is called a Release. Every time you install the same chart, a new and independent Release is created with its own name.

### Key Points About a Release
- One Chart can have **multiple Releases** running at the same time
- Each Release is **independent** — has its own resources, config, and history
- A Release is identified by its **name** and **namespace**
- All release data (values used, manifest, history) is stored as a Kubernetes Secret in that namespace

### Example to Understand
Imagine you have a `wordpress` chart. You can create:
```
Release: wordpress-dev    (in namespace: dev)
Release: wordpress-prod   (in namespace: prod)
Release: wordpress-qa     (in namespace: qa)
```
All three are running from the same chart but are completely separate releases with their own databases, configs, and state.

### What Does a Release Track?
- Which chart was used and its version
- What values were passed during install/upgrade
- The full history of every change made to it (via Revisions)
- Current status — deployed, failed, superseded, uninstalled

---

## 3. Revision

### What is a Revision?
A **Revision** is a versioned snapshot of a Release at a specific point in time. Every time you install, upgrade, or rollback a Release, Helm creates a new Revision and increments the revision number.

```
helm install   →  Revision 1  (initial install)
helm upgrade   →  Revision 2  (after first upgrade)
helm upgrade   →  Revision 3  (after second upgrade)
helm rollback  →  Revision 4  (rollback creates a new revision, not revert)
```

### What Does a Revision Store?
Each revision stores a complete snapshot of:
- The chart version used
- The values that were applied
- The rendered Kubernetes manifests
- Timestamp and status of that revision

### What is the Use of Revision?
- Gives you a **full history** of every change made to a release
- Allows you to **rollback** to any previous revision if something breaks
- Lets you **audit** what changed between versions
- Helm keeps the last 10 revisions by default (configurable via `--history-max`)

### Revision vs Release
| | Release | Revision |
|---|---|---|
| What it is | A running instance of a chart | A snapshot of a release at a point in time |
| Created when | `helm install` | Every `install`, `upgrade`, `rollback` |
| Identified by | Name + Namespace | Number (1, 2, 3...) |
| Purpose | Represents the application | Represents a version of that application's config |

---

## 4. How They All Connect

```
      CHART
   (the package)
        |
        | helm install
        ↓
     RELEASE
  (running instance)
        |
        | every change (upgrade/rollback)
        ↓
   REVISION 1 → REVISION 2 → REVISION 3 ...
  (snapshots of the release over time)
```

A **Chart** is the template.  
A **Release** is what you get when you deploy that template.  
A **Revision** is the history of every state that Release has been in.

---

---

# Practical Example — WordPress on Kubernetes

---

## Step 1 — Add the Bitnami Helm Repository

A **Chart Repository** is a hosted collection of charts. Before you can install a chart, you need to add the repository that hosts it.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

This tells Helm: "I want to use charts from Bitnami — go fetch the index from this URL."

```bash
helm repo update
```

This refreshes the local cache of available charts from all added repositories.

---

## Step 2 — Check Your Repositories

```bash
helm repo list
```

**Output:**
```
NAME      URL
bitnami   https://charts.bitnami.com/bitnami
```

This shows all repositories currently configured in your local Helm setup. Each entry has:
- `NAME` — the alias you use to reference this repo
- `URL` — the actual URL where Helm fetches chart metadata from

To search for a chart inside a repo:
```bash
helm search repo bitnami/wordpress
```

**Output:**
```
NAME                    CHART VERSION   APP VERSION   DESCRIPTION
bitnami/wordpress       18.1.0          6.4.2         WordPress is the world's most popular blogging ...
```

---

## Step 3 — Install the WordPress Chart

```bash
helm install my-wordpress bitnami/wordpress \
  --namespace wordpress-ns \
  --create-namespace \
  --set wordpressUsername=admin \
  --set wordpressPassword=admin123 \
  --set service.type=NodePort
```

**Breaking this down:**

| Part | What it Does |
|---|---|
| `helm install` | Command to deploy a chart |
| `my-wordpress` | The **Release name** you are giving this deployment |
| `bitnami/wordpress` | The **Chart** to use (repo/chart-name) |
| `--namespace wordpress-ns` | Deploy into this namespace |
| `--create-namespace` | Create the namespace if it doesn't exist |
| `--set wordpressUsername=admin` | Override a value from `values.yaml` |
| `--set service.type=NodePort` | Override the service type |

**Output:**
```
NAME: my-wordpress
LAST DEPLOYED: Wed May 06 2026
NAMESPACE: wordpress-ns
STATUS: deployed
REVISION: 1

NOTES:
  WordPress is now deployed. Access it at your NodePort IP.
```

This created:
- A **Release** named `my-wordpress`
- **Revision 1** for this release
- All Kubernetes resources (Deployment, Service, PVC, Secret, etc.) defined in the WordPress chart

---

## Step 4 — List All Releases

```bash
helm list -n wordpress-ns
```

**Output:**
```
NAME            NAMESPACE       REVISION   UPDATED                  STATUS     CHART              APP VERSION
my-wordpress    wordpress-ns    1          2026-05-06 10:00:00      deployed   wordpress-18.1.0   6.4.2
```

**What each column means:**

| Column | Meaning |
|---|---|
| `NAME` | The Release name |
| `NAMESPACE` | Kubernetes namespace the release lives in |
| `REVISION` | Current revision number of this release |
| `UPDATED` | When this release was last modified |
| `STATUS` | Current state — `deployed`, `failed`, `pending`, `superseded` |
| `CHART` | Chart name and version used |
| `APP VERSION` | Version of the actual application (WordPress) |

To list releases across **all namespaces**:
```bash
helm list --all-namespaces
```
or shorthand:
```bash
helm list -A
```

---

## Step 5 — Upgrade the Release (Creates a New Revision)

```bash
helm upgrade my-wordpress bitnami/wordpress \
  --namespace wordpress-ns \
  --set wordpressPassword=newpassword456
```

**Output:**
```
Release "my-wordpress" has been upgraded. Happy Helming!
REVISION: 2
STATUS: deployed
```

Now running `helm list` shows `REVISION: 2`.

---

## Step 6 — Check Release History (All Revisions)

```bash
helm history my-wordpress -n wordpress-ns
```

**Output:**
```
REVISION   UPDATED                  STATUS       CHART              DESCRIPTION
1          2026-05-06 10:00:00      superseded   wordpress-18.1.0   Install complete
2          2026-05-06 10:15:00      deployed     wordpress-18.1.0   Upgrade complete
```

- **Revision 1** is now `superseded` (replaced by a newer revision)
- **Revision 2** is the current `deployed` state

---

## Step 7 — Rollback to a Previous Revision

```bash
helm rollback my-wordpress 1 -n wordpress-ns
```

This rolls the release back to Revision 1's configuration. But it does **not go back to Revision 1** — it creates a **new Revision 3** with the same config as Revision 1.

```bash
helm history my-wordpress -n wordpress-ns
```

**Output:**
```
REVISION   STATUS       DESCRIPTION
1          superseded   Install complete
2          superseded   Upgrade complete
3          deployed     Rollback to 1
```

---

## Step 8 — Uninstall a Release

```bash
helm uninstall my-wordpress -n wordpress-ns
```

**Output:**
```
release "my-wordpress" uninstalled
```

This completely removes:
- All Kubernetes resources created by the release (Pods, Services, PVCs, Secrets, etc.)
- The release record and its revision history from the cluster

To verify it is gone:
```bash
helm list -n wordpress-ns
```

**Output:**
```
NAME   NAMESPACE   REVISION   UPDATED   STATUS   CHART   APP VERSION
(no output — release is fully removed)
```

If you want to remove the release but **keep the history** for audit purposes:
```bash
helm uninstall my-wordpress -n wordpress-ns --keep-history
```

After this, `helm list` will not show it, but `helm history my-wordpress -n wordpress-ns` will still return the revision records.

---

## Command Reference

| Command | What it Does |
|---|---|
| `helm repo add <name> <url>` | Add a chart repository |
| `helm repo update` | Refresh chart index from all repos |
| `helm repo list` | List all configured repositories |
| `helm search repo <keyword>` | Search for charts in added repos |
| `helm install <release> <chart>` | Install a chart as a new release |
| `helm list` or `helm ls` | List all releases in current namespace |
| `helm list -A` | List releases across all namespaces |
| `helm status <release>` | Show current status of a release |
| `helm history <release>` | Show all revisions of a release |
| `helm upgrade <release> <chart>` | Upgrade an existing release |
| `helm rollback <release> <revision>` | Rollback to a specific revision |
| `helm uninstall <release>` | Fully remove a release |
| `helm uninstall <release> --keep-history` | Remove release but keep revision history |
