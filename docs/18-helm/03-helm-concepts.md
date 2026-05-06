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

## Step 3 — Pull the Chart Locally and Edit Before Installing

Sometimes you don't want to install a chart directly from the repository. Instead, you want to **download the chart to your machine first**, look at everything inside it, edit the `values.yaml` directly, and then install from your local copy.

This is the most transparent and controlled way to deploy — you know exactly what is going in.

### Why Do This?
- You want to see **all available configuration options** in `values.yaml` before deciding what to change
- You want to **edit templates** inside the chart for custom behaviour
- You want to inspect what Kubernetes resources the chart will create
- You want to store the chart in your own Git repository
- You don't fully trust a remote chart and want to audit it first

---

### Pull the Chart (as a `.tgz` archive)

```bash
helm pull bitnami/wordpress
```

This downloads the chart as a compressed archive:
```
wordpress-18.1.0.tgz
```

This is just the packaged form — you cannot edit it directly yet.

---

### Pull and Untar in One Command

```bash
helm pull bitnami/wordpress --untar
```

This downloads and **extracts** the chart into a local directory in one step:

```
wordpress/
├── Chart.yaml
├── values.yaml          ← this is what you edit
├── charts/
│   └── mariadb/
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ...
└── crds/
```

Now you have the full chart folder on your machine and can open any file.

---

### Pull a Specific Version

If you need a particular version of the chart instead of the latest:

```bash
helm pull bitnami/wordpress --untar --version 18.1.0
```

---

### Open and Edit `values.yaml`

The `values.yaml` file inside the chart contains **every single configurable option** for that chart, with comments explaining what each one does. This is the full reference — far more complete than any documentation.

```bash
# open in any editor
vim wordpress/values.yaml
nano wordpress/values.yaml
code wordpress/values.yaml
```

Inside `values.yaml` you will see everything available:

```yaml
## WordPress credentials
wordpressUsername: user          # ← change this
wordpressPassword: ""            # ← change this
wordpressEmail: user@example.com # ← change this
wordpressBlogName: User's Blog   # ← change this

## Service configuration
service:
  type: LoadBalancer             # ← change to NodePort or ClusterIP

## Replica count
replicaCount: 1                  # ← change as needed

## MariaDB configuration
mariadb:
  auth:
    rootPassword: ""             # ← set this
    password: ""                 # ← set this

## Resource limits
resources:
  requests:
    memory: 512Mi
    cpu: 300m
  # limits:
  #   memory: 1Gi
  #   cpu: 600m

## Ingress
ingress:
  enabled: false                 # ← set true if you want Ingress
  hostname: wordpress.local
```

Edit whatever values you need directly in this file and save.

---

### Install from the Local Chart Directory

Once you have edited `values.yaml`, install directly from the local folder instead of the remote repo:

```bash
helm install my-wordpress ./wordpress \
  --namespace wordpress-ns \
  --create-namespace
```

Notice `./wordpress` — this points to the **local chart directory** you pulled, not `bitnami/wordpress` from the remote repo.

Helm will use the `values.yaml` inside that folder automatically. No `-f` flag needed since you edited the file in place.

---

### Alternatively — Keep `values.yaml` Clean and Use a Separate Override File

A better practice is to **not edit the chart's own `values.yaml`** directly. Instead, keep it untouched as a reference and create a separate override file with only the values you want to change.

```yaml
# my-overrides.yaml  (only what you're changing)

wordpressUsername: admin
wordpressPassword: admin123
wordpressBlogName: My Blog
service:
  type: NodePort
replicaCount: 2
```

Then install with:
```bash
helm install my-wordpress ./wordpress \
  --namespace wordpress-ns \
  --create-namespace \
  -f my-overrides.yaml
```

This keeps the original `values.yaml` as a **clean reference** and your overrides in a separate tracked file.

---

### Full Flow — Pull, Edit, Install

```
helm pull bitnami/wordpress --untar
        ↓
  wordpress/ folder on your machine
        ↓
  open values.yaml — read all options, change what you need
        ↓
  helm install my-wordpress ./wordpress --namespace wordpress-ns --create-namespace
        ↓
  Release created from your local edited chart
```

---

## Step 4 — Passing Configuration Values (When Installing Directly from Repo)

If you are installing directly from a repo (not using a local pulled chart), you need to pass configuration externally. There are two ways to do this.

---

### Way 1 — Using `--set` (Inline, for quick/simple overrides)

You pass individual values directly in the command line using `--set key=value`.

```bash
helm install my-wordpress bitnami/wordpress \
  --set wordpressUsername=admin \
  --set wordpressPassword=admin123 \
  --set service.type=NodePort
```

**When to use `--set`:**
- Quick one-off overrides during testing
- When you only need to change one or two values
- Not recommended for production — values are not stored in a file and hard to track or version

---

### Way 2 — Using `-f values.yaml` (Values File, recommended)

You create a YAML file with all your custom values and pass it using `-f` or `--values`.

**Create your custom values file:**

```yaml
# wordpress-values.yaml

wordpressUsername: admin
wordpressPassword: admin123
wordpressEmail: admin@example.com
wordpressBlogName: My Blog

service:
  type: NodePort

replicaCount: 2

mariadb:
  auth:
    rootPassword: rootpass123
    password: dbpass123

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

**Then install using the values file:**

```bash
helm install my-wordpress bitnami/wordpress \
  --namespace wordpress-ns \
  --create-namespace \
  -f wordpress-values.yaml
```

or using the long flag:
```bash
helm install my-wordpress bitnami/wordpress \
  --namespace wordpress-ns \
  --create-namespace \
  --values wordpress-values.yaml
```

**Why use a values file:**
- All configuration is in one place — easy to read and maintain
- Can be **version controlled** in Git alongside your code
- Makes deployments **reproducible** — anyone with the file gets the same result
- Easier to manage many values compared to a long chain of `--set` flags
- Separates config from commands cleanly

---

### Combining Both `-f` and `--set`

You can use both together. `--set` values always **override** what is in the values file.

```bash
helm install my-wordpress bitnami/wordpress \
  -f wordpress-values.yaml \
  --set wordpressPassword=overridepassword
```

This is useful when you have a base values file and want to override just one thing at deploy time (e.g., a secret that should not be stored in the file).

---

### Priority Order of Values
```
Default values.yaml in chart  (lowest priority)
        ↓
-f custom-values.yaml
        ↓
--set key=value               (highest priority, always wins)
```

---

## Step 5 — Install the WordPress Chart

```bash
helm install my-wordpress bitnami/wordpress \
  --namespace wordpress-ns \
  --create-namespace \
  -f wordpress-values.yaml
```

**Breaking this down:**

| Part | What it Does |
|---|---|
| `helm install` | Command to deploy a chart |
| `my-wordpress` | The **Release name** you are giving this deployment |
| `bitnami/wordpress` | The **Chart** to use (repo/chart-name) |
| `--namespace wordpress-ns` | Deploy into this namespace |
| `--create-namespace` | Create the namespace if it doesn't exist |
| `-f wordpress-values.yaml` | Use this file for all configuration values |

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

## Step 6 — List All Releases

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
helm list -A
```

---

## Step 7 — Upgrade the Release Using an Updated Values File

When you want to change configuration, update your values file and run upgrade.

**Update the values file:**
```yaml
# wordpress-values.yaml (updated)

wordpressUsername: admin
wordpressPassword: newpassword456      # changed
wordpressEmail: admin@example.com
wordpressBlogName: My Updated Blog    # changed

service:
  type: NodePort

replicaCount: 3                        # changed from 2 to 3
```

**Run the upgrade:**
```bash
helm upgrade my-wordpress bitnami/wordpress \
  --namespace wordpress-ns \
  -f wordpress-values.yaml
```

**Output:**
```
Release "my-wordpress" has been upgraded. Happy Helming!
REVISION: 2
STATUS: deployed
```

Now running `helm list` shows `REVISION: 2`.

---

## Step 8 — Check Release History (All Revisions)

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

## Step 9 — Rollback to a Previous Revision

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

## Step 10 — Uninstall a Release

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
| `helm pull <repo/chart>` | Download chart as a `.tgz` archive |
| `helm pull <repo/chart> --untar` | Download and extract chart into a local folder |
| `helm pull <repo/chart> --untar --version <ver>` | Pull a specific chart version |
| `helm install <release> ./chart-dir` | Install from a local extracted chart folder |
| `helm install <release> <chart> -f values.yaml` | Install using a values file |
| `helm install <release> <chart> --set key=val` | Install with inline value overrides |
| `helm install <release> <chart> -f values.yaml --set key=val` | Install with both (--set wins) |
| `helm list` or `helm ls` | List all releases in current namespace |
| `helm list -A` | List releases across all namespaces |
| `helm status <release>` | Show current status of a release |
| `helm history <release>` | Show all revisions of a release |
| `helm upgrade <release> <chart> -f values.yaml` | Upgrade a release using a values file |
| `helm rollback <release> <revision>` | Rollback to a specific revision |
| `helm uninstall <release>` | Fully remove a release |
| `helm uninstall <release> --keep-history` | Remove release but keep revision history |
