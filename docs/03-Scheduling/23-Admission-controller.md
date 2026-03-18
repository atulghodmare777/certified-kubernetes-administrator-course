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

# Kubernetes Admission Webhooks (Mutating & Validating)

## 📌 Overview

Admission Webhooks allow you to create **custom admission controllers**.

👉 They let you:
- Modify requests (Mutating)
- Allow/Deny requests (Validating)

They extend Kubernetes beyond built-in admission controllers.

---

# 🔄 Request Flow (Very Important)


kubectl → API Server → AuthN → AuthZ → Mutating Webhooks → Validating Webhooks → etcd


## Key Points:
- Mutating runs **first**
- Validating runs **after mutation**
- Order matters

---

# 🧩 Types of Admission Webhooks

## 1. MutatingAdmissionWebhook

- Modifies incoming requests
- Example:
  - Add default labels
  - Inject sidecars (e.g., Istio)
  - Add storageClass to PVC

---

## 2. ValidatingAdmissionWebhook

- Only allows or rejects requests
- Example:
  - Reject pods running as root
  - Enforce labels
  - Enforce resource limits

---

# 🧠 Real Example (Your Case)

PVC without storageClass:

- Mutating controller → adds default storageClass  
- Validating controller → ensures policy compliance  

---

# 🏗️ Architecture (How It Works)

## Components Required

1. **Webhook Server (Your Code)**
   - Handles admission requests
   - Returns response (allow/deny/mutate)

2. **Deployment**
   - Runs webhook server in cluster

3. **Service**
   - Exposes webhook server internally

4. **Webhook Configuration**
   - Registers webhook with API server

---

## 🔄 Flow


API Server → Webhook Config → Service → Pod (Webhook Server)


---

# ⚙️ Step-by-Step Setup

---

## 1. Webhook Server

- Must expose HTTPS endpoint
- Accepts AdmissionReview request
- Returns AdmissionReview response

### Expected Endpoint


POST /validate
POST /mutate


---

## 2. Deploy Webhook Server

```yaml id="dep1"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook-server
  namespace: webhook-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webhook
  template:
    metadata:
      labels:
        app: webhook
    spec:
      containers:
      - name: webhook
        image: your-webhook-image
        ports:
        - containerPort: 443
3. Create Service
apiVersion: v1
kind: Service
metadata:
  name: webhook-service
  namespace: webhook-ns
spec:
  selector:
    app: webhook
  ports:
  - port: 443
    targetPort: 443
4. Create Validating Webhook
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-policy.example.com

webhooks:
- name: pod-policy.example.com

  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
    scope: "Namespaced"

  clientConfig:
    service:
      namespace: webhook-ns
      name: webhook-service
      path: /validate
    caBundle: <BASE64_CA_CERT>

  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
  failurePolicy: Fail
5. Create Mutating Webhook
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: mutate-pods.example.com

webhooks:
- name: mutate-pods.example.com

  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]

  clientConfig:
    service:
      namespace: webhook-ns
      name: webhook-service
      path: /mutate
    caBundle: <BASE64_CA_CERT>

  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
  failurePolicy: Fail
🔐 TLS Requirement (Very Important)

Webhook must use HTTPS

API server needs:

CA certificate (caBundle)

Server needs:

TLS cert + key

👉 Without TLS → webhook will NOT work

⚙️ Important Fields Explained
Field	Meaning
rules	What resources/operations to intercept
clientConfig	Where webhook server is
caBundle	Trust certificate
failurePolicy	Fail or Ignore on error
sideEffects	Must be declared
timeoutSeconds	Max wait time
🚨 failurePolicy (Important)
Value	Behavior
Fail	Reject request if webhook fails
Ignore	Continue even if webhook fails
🔥 Real Use Cases
Use Case	Type
Inject sidecar (Istio)	Mutating
Add labels automatically	Mutating
Block privileged containers	Validating
Enforce naming conventions	Validating
🧪 Example Scenario
Reject Pods Without Labels

Validating webhook checks:

If label missing → reject

Inject Env Variables

Mutating webhook:

Adds env vars automatically

⚠️ Common Mistakes

Missing TLS cert ❌

Wrong service name/namespace ❌

Invalid caBundle ❌

Webhook timeout ❌

Wrong path (/validate vs /mutate) ❌
