# Configuring Kubernetes Schedulers
  - Take me to [video Tutorial](https://kodekloud.com/topic/configuring-kubernetes-scheduler/)
  
In this section, we will take a look at configuring kubernetes schedulers.

![ks](../../images/ks.PNG)

# Kubernetes Scheduler – Complete Practical Notes

## 📌 Overview
- Kubernetes scheduler assigns Pods → Nodes
- It is **plugin-based** and works in **phases (extension points)**
- Scheduler behavior is controlled via:
  - **Profiles**
  - **Plugins**
- Default scheduler name: `default-scheduler`

---

# 🔄 Scheduling Workflow (Actual Internals)

## 1. Queue Phase
- Pods enter scheduling queue
- Types:
  - Active → ready
  - Backoff → retry later
  - Unschedulable → waiting for changes
- Controlled by: **PrioritySort plugin**

---

## 2. Scheduling Cycle (Node Selection)

### PreFilter
- Pre-compute pod requirements (CPU, memory)

### Filter (Hard Constraints)
- Remove invalid nodes:
  - Resource shortage
  - NodeSelector mismatch
  - Taints not tolerated

👉 Output: feasible nodes

---

### PostFilter
- Runs if no nodes available
- Can trigger **preemption**
  - Evict lower-priority pods

---

### Score (Soft Constraints)
- Rank nodes (0–100)
- Examples:
  - Least loaded
  - Balanced usage
  - Affinity preferences

---

### Reserve
- Reserve resources on selected node

---

### Permit (Optional)
- Delay scheduling if needed

---

## 3. Binding Cycle

### PreBind → Bind → PostBind
- Final checks → assign pod → cleanup

---

# 🧩 Default Scheduler

- Name: `default-scheduler`
- Already running in control plane
- Uses built-in plugins automatically

## Default Plugins (Important)

| Phase | Plugins |
|------|--------|
| Queue | PrioritySort |
| Filter | NodeResourcesFit, NodeAffinity, TaintToleration |
| Score | NodeResourcesBalancedAllocation, ImageLocality |
| Other | PodTopologySpread |

👉 Handles most real-world scenarios

---

# ⚙️ Scheduler Configuration (Key Concept)

Scheduler is configured using:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
🧠 Profiles (VERY IMPORTANT)

A profile = logical scheduler inside same binary

Each profile has:

Unique schedulerName

Its own plugin config

👉 Multiple profiles = multiple schedulers (no need to run multiple pods)

Example: Multiple Profiles
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration

profiles:
- schedulerName: default-scheduler

- schedulerName: high-perf-scheduler
  plugins:
    score:
      enabled:
      - name: NodeResourcesFit
      disabled:
      - name: NodeResourcesBalancedAllocation
🚀 How to USE Profiles (VERY IMPORTANT)
Pod chooses scheduler via:
spec:
  schedulerName: high-perf-scheduler

👉 This tells Kubernetes:

Use that profile inside scheduler

⚠️ Key Understanding

One scheduler process

Multiple profiles

Pods select profile using schedulerName

🏗️ Custom Scheduler (Separate Process)

Instead of profiles, you can run a separate scheduler

When to use:

Completely different logic

Isolation/testing

Run Custom Scheduler
apiVersion: v1
kind: Pod
metadata:
  name: my-scheduler
  namespace: kube-system
spec:
  containers:
  - name: kube-scheduler
    image: registry.k8s.io/kube-scheduler:v1.28.0
    command:
    - kube-scheduler
    - --config=/etc/kubernetes/scheduler-config.yaml
Use It
spec:
  schedulerName: my-scheduler
⚙️ Plugin Customization (Core Power)
Enable / Disable Plugins
plugins:
  score:
    disabled:
    - name: NodeResourcesBalancedAllocation
    enabled:
    - name: NodeResourcesFit
Change Scheduling Logic
pluginConfig:
- name: NodeResourcesFit
  args:
    scoringStrategy:
      type: LeastAllocated

👉 Schedules pods to nodes with more free resources

🔥 Real Usage Scenarios
1. High Performance Workloads

Use custom profile

Prefer least utilized nodes

2. Batch Jobs

Lower priority + spread across nodes

3. Cost Optimization

Pack pods tightly (MostAllocated strategy)

🔄 End-to-End Flow (Real Understanding)

Pod created

Enters queue

Scheduler (profile) picks it

Filter nodes

Score nodes

Select best node

Bind pod

If no node:
→ PostFilter → preemption

🔍 Verification
Check scheduler used
kubectl describe pod <pod>

Look for:

Scheduled by <scheduler-name>
Check logs
kubectl logs -n kube-system <scheduler-pod>
⚠️ Common Mistakes

Thinking only one scheduler exists ❌

Not understanding profiles ❌

Wrong schedulerName → Pending ❌

Overriding plugins incorrectly ❌

🧠 Final Mental Model

Scheduler = Engine
Profile = Configuration of engine
Plugins = Decision logic
schedulerName = Which configuration to use

🚀 Key Takeaways

Scheduler is plugin-driven

Profiles allow multiple behaviors in one scheduler

Pods choose scheduler using schedulerName

Preemption happens in PostFilter

Custom scheduler = advanced use case


---




## References
- https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduler.md
- https://kubernetes.io/blog/2017/03/advanced-scheduling-in-kubernetes/
- https://jvns.ca/blog/2017/07/27/how-does-the-kubernetes-scheduler-work/
- https://stackoverflow.com/questions/28857993/how-does-kubernetes-scheduler-work

