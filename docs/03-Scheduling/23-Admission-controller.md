## 📌 What are Admission Controllers?

When you run a command like:

kubectl apply -f pod.yaml

The request goes through multiple steps inside Kubernetes:

1. Authentication → Who are you?
2. Authorization → Are you allowed?
3. **Admission Controllers → Should this request be modified/allowed?**
4. Object is persisted in etcd

👉 Admission Controllers act as a **final gatekeeper**

---

## 🔄 Request Flow (Important)


kubectl → API Server → Authentication → Authorization → Admission Controllers → etcd


---

## ❗ Why Admission Controllers?

Authentication & Authorization cannot enforce rules like:

- ❌ Prevent pulling images from public registries  
- ❌ Ensure labels/metadata are present  
- ❌ Restrict running containers as root  
- ❌ Enforce resource limits  

👉 These are enforced using **Admission Controllers**

---

# 🧩 Types of Admission Controllers

## 1. Mutating Admission Controllers
- Can **modify the request**
- Example:
  - Add default labels
  - Set default imagePullPolicy

## 2. Validating Admission Controllers
- Can **only allow or deny**
- Example:
  - Reject pod without limits
  - Reject privileged containers

---

# ⚙️ Default Admission Controllers (Examples)

Some commonly enabled controllers:

| Admission Controller | Purpose |
|---------------------|--------|
| DefaultStorageClass | Assign default storage class |
| AlwaysPullImages    | Always pull images |
| NamespaceLifecycle  | Manage namespace rules |
| ServiceAccount      | Inject service account |
| ResourceQuota       | Enforce quotas |

---

# 🔍 How to Check Enabled Admission Controllers

## Method 1 (Direct Binary)

```bash
kube-apiserver -h | grep enable-admission-plugins
Sample Output
--enable-admission-plugins strings
    Admission plugins to enable.
    Default: NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,...
Method 2 (kubeadm cluster)
kubectl -n kube-system exec -it kube-apiserver-controlplane -- kube-apiserver -h | grep enable-admission-plugins
Method 3 (Most Practical ✅)
kubectl -n kube-system get pod kube-apiserver-controlplane -o yaml | grep admission
Sample Output
- --enable-admission-plugins=NodeRestriction,NamespaceLifecycle,ServiceAccount,...
⚙️ Enable / Disable Admission Controllers
📍 kubeadm Setup (Most Common)

Edit API server manifest:

vi /etc/kubernetes/manifests/kube-apiserver.yaml
✅ Enable Plugins
- --enable-admission-plugins=NodeRestriction,NamespaceLifecycle
❌ Disable Plugins
- --disable-admission-plugins=ServiceAccount
🔁 What Happens After Edit?

kubelet automatically restarts kube-apiserver

Changes are applied immediately

🚨 Important Controllers (Must Know)
NamespaceLifecycle (Important)

Replaces deprecated:

NamespaceAutoProvision ❌

NamespaceExists ❌

What it does:

✅ Prevents creating resources in non-existing namespaces

✅ Prevents deletion of:

kube-system

kube-public

NodeRestriction

Restricts kubelet permissions

Prevents node from modifying other node objects

AlwaysPullImages

Ensures image is always pulled

Improves security (no cached images)

❌ Deprecated Admission Controllers
Old	Replaced By
NamespaceAutoProvision	NamespaceLifecycle
NamespaceExists	NamespaceLifecycle
🧪 Example Scenario
Case 1: NamespaceLifecycle Enabled
kubectl run nginx --image=nginx -n test-ns
Output
Error from server (NotFound): namespaces "test-ns" not found
Case 2: NamespaceAutoProvision (Old behavior)

👉 Namespace would be auto-created (now deprecated)

🔥 Real Use Cases
Requirement	Admission Controller
Enforce security	PodSecurity / OPA
Limit resources	ResourceQuota
Default values	Mutating controllers
Namespace validation	NamespaceLifecycle
