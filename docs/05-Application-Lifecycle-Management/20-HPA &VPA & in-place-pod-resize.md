# 🚀 Kubernetes Autoscaling – VPA, HPA & In-Place Pod Resize

---

# 📌 1. Vertical Pod Autoscaler (VPA)

## 🔹 Overview
- Vertical Pod Autoscaler (VPA) automatically adjusts the CPU and memory requests/limits of containers.
- Unlike HPA, which scales the number of pods, VPA scales resources inside a pod.
- VPA is NOT built into Kubernetes by default — it must be installed separately.

---

## 🔹 Installation

```bash
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml
🔹 Components of VPA

After deployment:

kubectl get pods -n kube-system | grep vpa

You will see 3 main components:

1️⃣ VPA Recommender

Continuously monitors resource usage from Kubernetes Metrics API

Collects:

Historical usage data

Live usage data

Provides optimal CPU and memory recommendations

❗ Does NOT modify pods directly

2️⃣ VPA Updater

Detects pods running with suboptimal resources

Gets recommendations from Recommender

Evicts pods so they restart with updated resources

3️⃣ VPA Admission Controller

Intercepts pod creation requests

Mutates pod specs based on recommendations

Automatically injects correct resource requests/limits

🔹 VPA Update Modes (Policies)
Mode	Description
Off	Only provides recommendations (no changes)
Initial	Applies recommendations only at pod creation
Recreate	Evicts and recreates pods with new resources
Auto	Automatically decides when to update pods
🔹 Example: VPA Configuration
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: nginx
  updatePolicy:
    updateMode: "Auto"
🔹 When to Use VPA

Best suited for:

Databases (MySQL, PostgreSQL)

Machine Learning workloads

Stateful applications

Applications with unpredictable memory usage

⚠️ Limitations of VPA

Pod restart is required (except with in-place resize support)

Not ideal for highly available stateless applications

Cannot be safely used with HPA (on CPU/memory) in most cases

📌 2. Horizontal Pod Autoscaler (HPA)
🔹 Overview

HPA automatically scales the number of pods (replicas)

Based on:

CPU utilization

Memory utilization

Custom metrics (Prometheus, etc.)

🔹 How HPA Works

Collects metrics from Metrics Server

Compares with target values

Scales pods up/down accordingly

🔹 Example: HPA (Imperative)
kubectl autoscale deployment nginx --cpu-percent=50 --min=1 --max=5
🔹 Example: HPA (YAML)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
🔹 When to Use HPA

Best suited for:

Stateless applications

Web applications with variable traffic

Microservices

⚖️ VPA vs HPA
Feature	VPA	HPA
Scaling Type	Vertical (resources)	Horizontal (replicas)
Pod Restart	Required	Not required
Best For	Stateful apps	Stateless apps
Metrics	CPU & Memory	CPU, Memory, Custom
Conflict	Yes (CPU/memory overlap)	—
📌 3. In-Place Pod Resize
🔹 Overview

Allows changing CPU/memory without restarting pods

Solves the biggest limitation of VPA

🔹 Key Benefits

No pod eviction

Faster scaling

Better for production workloads

🔹 How It Works

Kubernetes updates container resources without killing the pod

Supported in newer Kubernetes versions (feature may be beta/alpha)

🔹 Example
resources:
  requests:
    cpu: "500m"
  limits:
    cpu: "1"

With in-place resize → values can be updated dynamically without restart

🔹 Relation with VPA

Future direction: VPA + In-place resize

Eliminates:

Pod eviction

Restart disruptions

⚠️ VPA + HPA Together?

❌ Not recommended when both use CPU/memory

Reason:

HPA scales based on utilization %

VPA changes resource requests → affects utilization calculation

✅ When They Can Work Together

HPA → custom metrics (e.g., QPS, requests/sec)

VPA → CPU/memory tuning

🎯 Final Summary

HPA → Scale OUT (increase number of pods)

VPA → Scale UP (increase resources per pod)

In-place resize → Scale WITHOUT restart
