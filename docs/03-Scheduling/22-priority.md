# Kubernetes PriorityClass & Preemption

## 📌 Overview

- **PriorityClass** defines pod priority for scheduling  
- Higher priority pods are scheduled **before** lower priority ones  
- If resources are insufficient → lower priority pods **may be evicted** (preemption)

---

## 🔢 Priority Values

- Range: **-2147483648 to 1000000000**
- Default priority (no class): **0**

### 💡 Recommended Ranges

| Use Case | Priority Value |
|----------|--------------|
| Critical system pods | 1000000+ |
| Production workloads | 10000 – 100000 |
| Normal workloads | 1000 – 10000 |
| Batch / background jobs | 0 – 1000 |
| Low priority / best-effort | < 0 |

> ⚠️ Avoid extremely high values unless required

---

## 🔍 List PriorityClasses

```bash
kubectl get priorityclass
🏗️ Create PriorityClass
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "High priority workloads"
Notes

Only one PriorityClass can have globalDefault: true

If not set → pods get priority 0

🧩 Use in Pod
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: nginx
🚨 Preemption Policy

Controls whether high-priority pods can evict lower-priority pods

1. Default (PreemptLowerPriority)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
preemptionPolicy: PreemptLowerPriority

Evicts lower priority pods

Schedules high priority pod

2. No Preemption
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-no-preempt
value: 100000
preemptionPolicy: Never

No eviction

Pod waits in Pending

🧠 Key Points

Priority = who gets scheduled first

Preemption = who gets evicted

Default behavior allows eviction

Use Never to avoid disruption

⚠️ Common Mistakes

Multiple globalDefault: true ❌

Assuming priority guarantees scheduling ❌

Ignoring preemption behavior ❌
