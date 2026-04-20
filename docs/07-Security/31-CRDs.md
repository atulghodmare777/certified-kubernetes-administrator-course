# Kubernetes Controllers, CRDs, and Custom Resources — Complete Guide

## 📌 Overview

Kubernetes (K8s) manages workloads using **resources** such as:

* Deployment
* ReplicaSet
* Pod
* Job
* CronJob
* StatefulSet
* Namespace

When any resource is created, it is stored in the **etcd database**, which acts as the cluster’s source of truth.

---

## ⚙️ How Changes Actually Happen in Kubernetes

When you create or modify a resource (e.g., update a Deployment YAML):

1. The change is submitted to the **Kubernetes API Server**
2. The API Server stores the updated state in **etcd**
3. **Controllers** continuously monitor the state
4. If desired state ≠ current state → controller takes action

---

## 🤖 What are Controllers?

Controllers are core Kubernetes components that:

* Continuously **watch resources**
* Compare:

  * Desired state (from YAML)
  * Current state (actual cluster)
* Take corrective actions to match both

### Examples of Controllers

* Deployment Controller
* ReplicaSet Controller
* Job Controller
* CronJob Controller

📌 These controllers:

* Run inside Kubernetes control plane
* Are written in **Go (Golang)**

---

## 🧩 Custom Resources in Kubernetes

By default, Kubernetes supports predefined resource types.

If you try to create a custom resource like this:

```yaml
apiVersion: flights.com/v1
kind: FlightTicket
metadata:
  name: my-flight-ticket
spec:
  from: Mumbai
  to: London
  number: 2
```

And run:

```bash
kubectl create -f flightticket.yaml
```

You will get:

```bash
no matches for kind "FlightTicket" in version "flights.com/v1"
```

---

## ❗ Why This Error Occurs

Kubernetes does not recognize this resource because:

👉 It is **not registered in the Kubernetes API**

---

## 🛠️ Solution: Custom Resource Definition (CRD)

A **CRD (Custom Resource Definition)** allows you to:

* Define new resource types
* Extend Kubernetes API
* Add validation and schema

---

## 📄 CRD Structure Explained

CRDs also follow standard Kubernetes structure:

* apiVersion
* kind
* metadata
* spec

---

### 🔑 Important Fields in CRD

#### 1. Scope

Defines whether resource is:

* Namespaced (e.g., Pods, Deployments)
* Cluster-wide (e.g., Nodes, PVs)

---

#### 2. Group

Used in API version:

```
apiVersion: flights.com/v1
```

Here:

```
group = flights.com
```

---

#### 3. Names

Defines how resource is referenced:

* kind → FlightTicket
* singular → flightticket
* plural → flighttickets
* shortNames → ft

---

#### 4. Versions

* Multiple versions supported
* Only one version can be:

  * storage: true (used in etcd)
* served: true → available via API

---

#### 5. Schema

Defines:

* Allowed fields
* Data types
* Validation rules

---

## 📘 Example CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: flighttickets.flights.com
spec:
  scope: Namespaced
  group: flights.com
  names:
    kind: FlightTicket
    singular: flightticket
    plural: flighttickets
    shortNames:
      - ft
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                from:
                  type: string
                to:
                  type: string
                number:
                  type: integer
                  minimum: 1
                  maximum: 10
```

---

## 🚀 Step-by-Step Execution

### 1. Create CRD

```bash
kubectl create -f flightticket-custom-definition.yaml
```

Output:

```bash
customresourcedefinition "flighttickets.flights.com" created
```

---

### 2. Create Custom Resource

```bash
kubectl create -f flightticket.yaml
```

Output:

```bash
flightticket "my-flight-ticket" created
```

---

### 3. Get Resource

```bash
kubectl get flightticket
```

---

### 4. Delete Resource

```bash
kubectl delete -f flightticket.yaml
```

---

## ⚠️ Important Limitation

CRD only:

✅ Registers new resource type
✅ Stores object in etcd
✅ Validates schema

❌ Does NOT perform any action

---

## 🧠 Why?

Because:

👉 There is **NO controller for your custom resource**

---

## 🔥 What’s Missing?

To make CRDs useful, you need:

### 👉 Custom Controller (Operator)

A controller will:

* Watch FlightTicket resources
* Take action (e.g., book ticket, call API, etc.)

---

## 🏗️ Final Architecture

```
User YAML → API Server → etcd
                       ↓
                  Controller watches
                       ↓
             Takes action to match state
```

---

## 🎯 Summary

| Concept               | Description                 |
| --------------------- | --------------------------- |
| etcd                  | Stores cluster state        |
| API Server            | Entry point for all changes |
| Controllers           | Maintain desired state      |
| CRD                   | Define new resource types   |
| Custom Resource       | Instance of CRD             |
| Controller (Operator) | Adds actual logic           |

---

## 🚀 Key Takeaway

> Kubernetes is not just a container orchestrator — it is an extensible platform.

* CRD = Define resource
* Controller = Add behavior

👉 Together → build powerful automation systems

---

## 📌 Example Use Cases of CRDs

* Databases (MySQL Operator, MongoDB Operator)
* CI/CD pipelines
* Backup systems
* Custom workflows

---

## 🧾 Conclusion

CRDs allow you to extend Kubernetes, but without controllers they are just **data stored in etcd**.

To unlock full power:
👉 Always combine CRD + Controller

---
