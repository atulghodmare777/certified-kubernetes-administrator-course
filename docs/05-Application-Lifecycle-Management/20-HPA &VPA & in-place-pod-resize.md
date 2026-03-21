# 🚀 Kubernetes Autoscaling – Complete Guide (VPA, HPA, In-Place Resize)

---

## 🧠 1. Problem Statement

In Kubernetes, every Pod requires:
- CPU
- Memory

But defining correct values is difficult:
- Too low → Pod crashes (OOMKilled)
- Too high → Resource wastage

👉 Solution: **Autoscaling**

---

## 📌 2. Types of Autoscaling

| Type | What it scales | Example |
|------|--------------|--------|
| HPA  | Number of Pods | 2 → 10 Pods |
| VPA  | CPU/Memory inside Pod | 200m → 500m CPU |
| Cluster Autoscaler | Nodes | Add/remove VM |

---

## 📌 3. Vertical Pod Autoscaler (VPA)

### 🔹 What Problem VPA Solves
👉 "How much CPU/Memory does my app actually need?"

---

### 🔹 Core Idea

```
Observe → Recommend → Apply
```

---

### 🔹 Installation

```bash
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml
```

---

### 🔹 Components (Very Important)

#### 1. Recommender (Brain)

- Reads data from:
  - Metrics Server
  - Historical usage
- Calculates:
  - Ideal CPU
  - Ideal Memory

**Output Example:**
```
Recommended CPU: 500m (current: 200m)
```

❗ Does NOT modify Pods

---

#### 2. Updater (Executor)

- Compares running Pods with recommendations
- If mismatch → Evicts the Pod

**Why eviction?**  
Kubernetes cannot change CPU/Memory of a running Pod

---

#### 3. Admission Controller (Mutator)

- Works during Pod creation
- Modifies Pod spec before it starts

**Example:**
```
User request: 200m CPU
VPA recommendation: 500m CPU
Final Pod: 500m CPU
```

---

### 🔹 End-to-End Flow

1. Pod starts with initial resources  
2. Recommender analyzes usage  
3. Recommendation is generated  
4. Updater evicts Pod (if needed)  
5. New Pod is created  
6. Admission Controller injects updated values  

---

### 🔹 VPA Modes

| Mode | Behavior |
|------|--------|
| Off | Only recommendation |
| Initial | Apply only at creation |
| Recreate | Kill & recreate Pod |
| Auto | Automatically decide |

---

### 🔹 Example (VPA YAML)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  updatePolicy:
    updateMode: "Auto"
```

---

### 🔹 When to Use VPA

✅ Suitable for:
- Databases
- ML workloads
- Stateful applications

❌ Avoid for:
- Stateless high-availability apps (due to restarts)

---

### ⚠️ Limitation

- Requires **Pod restart**

---

## 📌 4. Horizontal Pod Autoscaler (HPA)

### 🔹 What Problem HPA Solves
👉 "My traffic is increasing, I need more Pods"

---

### 🔹 Working Principle

```
Increase load → Increase CPU → Add Pods
```

---

### 🔹 Flow

1. Metrics Server collects CPU usage  
2. HPA compares with target  
3. Adjusts replica count  

---

### 🔹 Example (Imperative)

```bash
kubectl autoscale deployment nginx --cpu-percent=50 --min=1 --max=5
```

---

### 🔹 Example (YAML)

```yaml
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
```

---

### 🔹 When to Use HPA

✅ Best for:
- Web applications
- APIs
- Microservices

---

## ⚖️ 5. VPA vs HPA

| Feature | VPA | HPA |
|--------|-----|-----|
| Scaling | CPU/Memory | Pods |
| Restart Required | Yes | No |
| Best For | Stateful | Stateless |

---

## ⚠️ 6. VPA + HPA Conflict

### 🔹 Problem

HPA uses:
```
CPU Usage / CPU Request
```

VPA changes CPU request → breaks HPA calculation

---

### ❌ Bad Combination
- HPA (CPU-based)
- VPA (CPU updates)

---

### ✅ Safe Combination
- HPA → Custom metrics (QPS, RPS)
- VPA → CPU/Memory tuning

---

## 📌 7. In-Place Pod Resize

### 🔹 Problem

Why does VPA restart Pods?

```
Kubernetes cannot modify resources of running Pods
```

---

### 🔹 Solution

👉 In-Place Pod Resize allows:

```
Update CPU/Memory without restarting Pod
```

---

### 🔹 Benefits

- No downtime  
- No eviction  
- Faster scaling  

---

### 🔹 Example

```yaml
resources:
  requests:
    cpu: "500m"
  limits:
    cpu: "1"
```

👉 These values can be updated dynamically

---

### 🔹 Future

```
VPA + In-Place Resize = No Restart Scaling
```

---

## 🎯 8. Final Mental Model

- HPA → Add more Pods  
- VPA → Increase Pod resources  
- In-place resize → Update without restart  
