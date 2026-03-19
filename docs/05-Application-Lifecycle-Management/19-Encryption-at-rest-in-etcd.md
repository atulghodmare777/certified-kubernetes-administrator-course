🔐 Kubernetes Secrets Encryption at Rest
📌 Overview

In Kubernetes:

Secrets are Base64 encoded (not encrypted by default).

Base64 encoding is NOT secure — it is easily reversible.

By default, Secrets are stored in etcd as plain text.

To secure them, we must enable encryption at rest.

🔍 How to Check if Encryption is Enabled
✅ Method 1: Verify from etcd (Most Reliable)
Step A: Create a test secret
kubectl create secret generic secret1 \
  --from-literal=mykey=mydata \
  -n default
Step B: Read the secret directly from etcd
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/secret1 | hexdump -C
🔎 Interpretation:

If you see plain text (e.g., mydata) → ❌ Encryption is NOT enabled

If you see unreadable binary data → ✅ Encryption is enabled

✅ Method 2: Check API Server Configuration
ps -ef | grep kube-apiserver
🔎 Look for this flag:
--encryption-provider-config=/etc/kubernetes/enc/enc.yaml

If present → ✅ Encryption enabled

If absent → ❌ Encryption NOT enabled

🔐 How to Enable Encryption at Rest
Step 1: Generate Encryption Key
head -c 32 /dev/urandom | base64

Example output:

8rTB3KaNMdVzDdOm5oQiuItQESctxgrKxrx13+/GmNk=
Step 2: Create Encryption Configuration File
vi /etc/kubernetes/enc/enc.yaml

Paste the following:

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

👉 Replace <BASE64_KEY> with your generated key.

⚠️ Important Notes

aescbc → Used for encryption

identity → Fallback to read unencrypted data

Order matters:

aescbc must come before identity

If identity is first → ❌ No encryption happens

Step 3: Update kube-apiserver

Edit the static pod manifest:

vi /etc/kubernetes/manifests/kube-apiserver.yaml
Add the following:
1. Add flag in command:
- --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
2. Add volume mount:
volumeMounts:
- name: enc
  mountPath: /etc/kubernetes/enc
  readOnly: true
3. Add volume:
volumes:
- name: enc
  hostPath:
    path: /etc/kubernetes/enc
    type: DirectoryOrCreate
🔁 What Happens Next?

kube-apiserver will automatically restart

Encryption will now be enabled for new secrets

🔄 Important: Re-encrypt Existing Secrets

⚠️ Enabling encryption does NOT encrypt old secrets automatically

To re-encrypt all existing secrets:
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
✅ Summary
Feature	Default Behavior	After Enabling
Secret Storage	Plain text in etcd	Encrypted
Security	Weak (Base64 only)	Strong
Existing Secrets	Not encrypted	Must be re-applied
