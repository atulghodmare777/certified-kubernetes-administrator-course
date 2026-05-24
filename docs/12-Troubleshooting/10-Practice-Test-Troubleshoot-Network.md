# Kubernetes Network Troubleshooting Guide

## Overview

When facing network issues in Kubernetes, follow a systematic approach: start from the pod level, move up to services, then DNS, CNI plugins, and finally kube-proxy.

---

## 1. Pod-Level Troubleshooting

### Step 1: Check Pod Status

```bash
kubectl get po -o wide
```

### Step 2: Get Pod IPs

```bash
# Option A: Wide output
kubectl get po -o wide

# Option B: JSONPath for just IPs
kubectl get po -l app=hostnames -o=jsonpath='{.items[*].status.podIP}'
# Output: 10.244.0.5 10.244.0.6 10.244.0.7
```

### Step 3: Directly Test Pod Connectivity (Bypass Service)

Spin up a temporary `busybox` pod and hit each pod IP directly. This confirms the pod itself is healthy, independent of any service routing.

```bash
kubectl run -it --rm --restart=Never busybox --image=busybox sh

# Inside the busybox pod:
for ip in 10.244.0.5 10.244.0.6 10.244.0.7; do
  wget -qO- $ip:9376
done
```

> **Why?** If pods respond directly but not through the service, the issue is with the service configuration — not the pods.

---

## 2. Service & Endpoint Troubleshooting

When direct pod access works but the service doesn't, inspect the service configuration.

### Checklist

- [ ] Selector labels on the Service match pod labels
- [ ] `port` and `targetPort` are correctly configured
- [ ] Endpoints exist and show correct pod IPs

### Commands

```bash
# View service details (check selector, port, targetPort)
kubectl describe svc hostnames

# View endpoint slices — should list all matching pod IPs
kubectl get endpointslices -l kubernetes.io/service-name=hostnames -n default
```

> **Tip:** If endpoints are empty, your service selector doesn't match any pod labels. Compare `kubectl describe svc <name>` with `kubectl get po --show-labels`.

---

## 3. CoreDNS Troubleshooting

Kubernetes uses **CoreDNS** to resolve service names (e.g., `my-service.default.svc.cluster.local`) to cluster IPs. If CoreDNS is broken, pods can't find each other by name.

### How CoreDNS Works

```
App Pod
  └─> /etc/resolv.conf (points to kube-dns ClusterIP)
        └─> CoreDNS pods (in kube-system)
              ├─> In-cluster: resolves via Kubernetes API
              └─> Out-of-cluster: forwards to host's /etc/resolv.conf
```

### Key Resources

| Resource | Purpose |
|---|---|
| CoreDNS Deployment | Manages CoreDNS pods |
| kube-dns Service | Exposes CoreDNS on port 53 |
| CoreDNS ConfigMap | Holds the Corefile configuration |
| CoreDNS ServiceAccount | RBAC access for CoreDNS |

### Corefile Example

```
kubernetes cluster.local in-addr.arpa ip6.arpa {
  pods insecure
  fallthrough in-addr.arpa ip6.arpa
  ttl 30
}
proxy . /etc/resolv.conf
```

This config makes CoreDNS the DNS backend for `cluster.local` domains and forwards all other queries (e.g., google.com) to the node's `/etc/resolv.conf`.

### Troubleshooting Steps

**1. Check CoreDNS pod status**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

- `Running` → CoreDNS is up
- `Pending` → CNI plugin might not be installed or working

**2. Check kube-dns service endpoints**

```bash
kubectl get endpointslice -l k8s.io/service-name=kube-dns -n kube-system
```

If no endpoints appear, the service selector isn't matching the CoreDNS pods.

**3. Inspect a pod's /etc/resolv.conf**

```bash
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
```

- `nameserver` should point to the `kube-dns` service ClusterIP
- `search` paths enable short name resolution (e.g., `my-svc` → `my-svc.default.svc.cluster.local`)

**4. Test DNS resolution from a pod**

```bash
# Test cluster DNS (built-in kubernetes service)
kubectl exec -it busybox -- nslookup kubernetes.default.svc.cluster.local

# Test your own service
kubectl exec -it busybox -- nslookup my-app-service.default.svc.cluster.local
```

> If `nslookup` fails → DNS is definitely broken. If it succeeds → DNS is fine, look elsewhere.

---

## 4. CNI Plugin Troubleshooting

**CNI (Container Network Interface)** plugins assign IPs to pods and set up pod-to-pod communication across nodes.

### Common CNI Plugins

| Plugin | Overlay Network | Network Policy Support |
|---|---|---|
| Flannel | Yes | No |
| Calico | Optional | Yes (robust) |

> **Gotcha:** If multiple CNI config files exist in the CNI directory, kubelet picks the first one alphabetically. Remove leftover configs to avoid unexpected behavior.

### Symptoms of CNI Issues

- Pod stuck in `Pending` state
- Pod-to-pod communication completely broken

### Troubleshooting Steps

**1. Check CNI DaemonSet pods — every node should have one running**

```bash
# Calico
kubectl get pods -n kube-system -l k8s-app=calico-node

# Flannel
kubectl get pods -n kube-system -l app=flannel
```

**2. Check CNI pod logs for errors**

```bash
kubectl logs -n kube-system <cni-pod-name>
```

---

## 5. Kube-Proxy Troubleshooting

**kube-proxy** runs on every node and maintains network rules (iptables or IPVS) to route traffic from a Service ClusterIP to the actual pod IPs.

```
Client → Service ClusterIP → kube-proxy rules (iptables/IPVS) → Pod IP
```

> **When to suspect kube-proxy:** DNS resolution works fine (`nslookup` succeeds), but connecting to the service still fails.

### Troubleshooting Steps

**1. Check kube-proxy pod status**

```bash
kubectl get po -n kube-system -l k8s-app=kube-proxy
```

**2. Check kube-proxy logs**

```bash
kubectl logs -n kube-system <kube-proxy-pod-name>
```

**3. Inspect kube-proxy configuration**

```bash
kubectl get cm kube-proxy -n kube-system -o yaml
```

Look for the `mode` field — it will be `iptables` or `ipvs`.

**4. Verify network rules directly on a node**

```bash
# iptables mode
iptables -t nat -L -n -v | grep <service-name>

# IPVS mode
ipvsadm -ln
```

---

## Troubleshooting Decision Tree

```
Pod can't reach another service
          │
          ▼
  Can pod reach other pod IPs directly?
    ├─ NO  → CNI plugin issue (check DaemonSet logs)
    └─ YES ▼
        Does nslookup resolve service name?
          ├─ NO  → CoreDNS issue (check pods, endpoints, resolv.conf)
          └─ YES ▼
              Does the service have correct endpoints?
                ├─ NO  → Service selector / label mismatch
                └─ YES → kube-proxy issue (check rules and logs)
```
   




# Solution Troubleshoot Network

Lets have a look at the [Practice Test](https://kodekloud.com/topic/practice-test-troubleshoot-network/) of the Troubleshoot Network

Note that this lab is sequential. You must solve test 1 completely before you can solve test 2, i.e. you cannot skip test 1 and do test 2 only.

1. <details>
   <summary>Troubleshooting Test 1</summary>

   We are asked to ensure all the components are working, so first let's examine the cluster to see what state it is in.

   How many nodes, and their status?

   ```
   kubectl get nodes
   ```

   Seems OK...

   Next, the pods

   ```
   kubectl get pods -A
   ```

   Now we see that the `webapp` and `mysql` pods are stuck at `ContainerCreating`. We need to describe the pods and check the errors.

   You will note that they are complaining about `plugin type="weave-net" name="weave" failed (add): unable to allocate IP address`, so clearly we have a networking issue and it's related to Weave.

   When you did the `get pods` above, did you see any evidence of network support containers, like `weave`?

   No - so we need to install networking support.

   Let's install `Weave`

   ```
   kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
   ```

   Now wait for a minute or so for it to initialize, then check the application pods

   ```
   kubectl get pods -n triton
   ```

   </details>


1. <details>
   <summary>Troubleshooting Test 2</summary>

   Once again let's examine the cluster to see what state it is in.

   How many nodes, and their status?

   ```
   kubectl get nodes
   ```

   Seems OK...

   Next, the pods

   ```
   kubectl get pods -A
   ```

   The kube-proxy pod is not running. It is actually crash-looping which means it tries to start, then fails. As a result the rules needed to allow connectivity to the services have not been created. First place to look when diagnosing CrashLoopBackoff is the pod logs.

   1. Check the logs of the kube-proxy pod

      ```
      kubectl -n kube-system logs <name_of_the_kube_proxy_pod>
      ```

      We see that it cannot find a configuration file.

      Now try looking for the configuration in case it has a different name

      ```
      ls -l /var/lib/kube-proxy
      ```

      The directory is not found!

   1. Inspect the pod template spec in the `kube-proxy` daemonset.

      ```
      kubectl get ds -n kube-system kube-proxy -o yaml | less
      ```

      Scroll around and check volumes and volume mounts. Notice that a config map is mounted at the path `/var/lib/kube-proxy` within the pod.

   1. Inspect the config map

      ```
      kubectl describe cm -n kube-system kube-proxy
      ```

      Here we see that the files mounted by the config map are `config.conf` and `kubeconfig.conf`, but _not_ `configuration.conf`.

      These two files are

      * `config.conf` - This is the actual configuration that kube-proxy needs to load. This file refers to `kubeconfig.conf`
      * `kubeconfig.conf` - This is simply a kubeconfig file, same as you will find on the lab terminal in `~/.kube/config`. It is the credentials and address for kube-proxy to talk to the api server.

   1. Fix the command line arguments to `kube-proxy`

      ```
      kubectl edit ds -n kube-system kube-proxy
      ```

      Set the correct filename

      ```
      --config=/var/lib/kube-proxy/config.conf
      ```

      Finally, confirm it is running.

      ```
      kubectl get pods -n kube-system
      ```

   </details>
