# 🔐 Kubernetes Secrets Encryption at Rest

---

## 📌 Overview

In Kubernetes:

* Secrets are **Base64 encoded** (NOT encrypted by default)
* Base64 encoding is **easily reversible**
* Secrets are stored in **etcd as plain text (by default)**
* To secure them, we must enable **Encryption at Rest**

---

## 🔍 How to Check if Encryption is Enabled

### ✅ Method 1: Verify from etcd (Most Reliable)

#### Step 1: Create a test secret

```
kubectl create secret generic secret1 \
  --from-literal=mykey=mydata \
  -n default
```

#### Step 2: Read the secret directly from etcd

```
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/secret1 | hexdump -C
```

#### 🔎 Result Interpretation

* If you see readable text like `mydata` → ❌ Encryption NOT enabled
* If you see unreadable/binary output → ✅ Encryption enabled

---

### ✅ Method 2: Check API Server Configuration

```
ps -ef | grep kube-apiserver
```

#### 🔎 Look for this flag:

```
--encryption-provider-config=/etc/kubernetes/enc/enc.yaml
```

* Present → ✅ Encryption enabled
* Not present → ❌ Encryption NOT enabled

---

## 🔐 How to Enable Encryption at Rest

### 🧩 Step 1: Generate Encryption Key

```
head -c 32 /dev/urandom | base64
```

Example output:

```
8rTB3KaNMdVzDdOm5oQiuItQESctxgrKxrx13+/GmNk=
```

---

### 🧩 Step 2: Create Encryption Configuration File

```
vi /etc/kubernetes/enc/enc.yaml
```

Paste the following:

```
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration

resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <BASE64_KEY>
      - identity: {}
```

👉 Replace `<BASE64_KEY>` with your generated key

---

### ⚠️ Important Notes

* `aescbc` → Used for encryption
* `identity` → Fallback (used to read old unencrypted data)

#### 🚨 Order is IMPORTANT

* ✅ Correct: `aescbc → identity`
* ❌ Wrong: `identity → aescbc` (No encryption will happen)

---

### 🧩 Step 3: Update kube-apiserver

Edit the static pod manifest:

```
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

#### ➤ Add encryption flag

```
- --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
```

#### ➤ Add volume mount

```
volumeMounts:
- name: enc
  mountPath: /etc/kubernetes/enc
  readOnly: true
```

#### ➤ Add volume

```
volumes:
- name: enc
  hostPath:
    path: /etc/kubernetes/enc
    type: DirectoryOrCreate
```

---

## 🔁 What Happens Next?

* kube-apiserver will automatically restart
* Encryption will apply to newly created secrets

---

## 🔄 Re-encrypt Existing Secrets

⚠️ Enabling encryption does NOT affect existing secrets

To re-encrypt all secrets:

```
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

---

## 📊 Summary

| Feature          | Default Behavior   | After Enabling     |
| ---------------- | ------------------ | ------------------ |
| Secret Storage   | Plain text in etcd | Encrypted          |
| Security         | Weak (Base64 only) | Strong             |
| Existing Secrets | Not encrypted      | Must be re-applied |

---
ref: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
