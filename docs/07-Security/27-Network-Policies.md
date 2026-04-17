# Network Policies
  - Take me to [Video Tutorials](https://kodekloud.com/topic/network-policies-3/)
  
#### Trafic flowing through a webserver serving frontend to users an app server serving backend API and a database server

  ![traffic](../../images/traffic.PNG)
  
- There are two types of traffic
  - Ingress
  - Egress
  
   ![ing1](../../images/ing1.PNG)
  
   ![ing2](../../images/ing2.PNG)
  
## Network Security

  ![nsec](../../images/nsec.PNG)
  
## Network Policy

  ![npol](../../images/npol.PNG)
  
  ![npol1](../../images/npol1.PNG)
  
## Network Policy Selectors
  
  ![npolsec](../../images/npolsec.PNG)
  
## Network Policy Rules

  ![npol2](../../images/npol2.PNG)
  
## Create network policy
 
- To create a network policy
  ```
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
   name: db-policy
  spec:
    podSelector:
      matchLabels:
        role: db
    policyTypes:
    - Ingress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            role: api-pod
      ports:
      - protocol: TCP
        port: 3306
  ```
  
  - With only pod selector any pod with label as mentioned above can access the db it can be from any namespace, how to restrict that?
  - This will taken care by namespaceSelector block as shown in below example
  - Suppose outside server which is not in k8s environment want to access the db then that is possible by ipBlock block as mentioned in below example
    
```
apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
   name: db-policy
  spec:
    podSelector:
      matchLabels:
        role: db
    policyTypes:
    - Ingress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            role: api-pod
        namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: prod
      - ipBlock:
            cidr: 192.168.5.10/32     
      ports:
      - protocol: TCP
        port: 3306

```

  ```
  $ kubectl create -f policy-definition.yaml
  ```
- In above example podSelector and namespaceSelector acts as a "AND" condition means only when both meets it will allow, means in our case pod with
  label role: api-pod from prod namespace only will able to access the db pod
- In above example podSelector and ipBlock acts as a "OR" condition means any of the 2 can access the db pod
- Suppose id we create namespaceSelector with saperate block as below then any pod within the cluster can access the db pod as it is "OR" condition
```
- from:
  - podSelector:
      matchLabels:
         role: api-pod
  - namespaceSelector:
      matchLabels:
         kubernetes.io/metadata.name: prod
```

## Egress rule:
- Suppose we have a backup server where need to perform backup action from db pod then we can define egress rule
- Note by default when we create a ingress rule we do not have to add egress rule but in this condition we require dedicated egress rul.

```
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
   name: db-policy
  spec:
    podSelector:
      matchLabels:
        role: db
    policyTypes:
    - Ingress
    - Egress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            role: api-pod
      ports:
      - protocol: TCP
        port: 3306
     egress:
     - to:
       - ipBlock: 192.168.5.10/32   # It can be anything like podSelector & namespaceSelector as well as per requirement
       ports:
       - protocol: TCP
         port: 80
  ```  
  
 ![npol3](../../images/npol3.PNG)
 
 ![npol4](../../images/npol4.PNG)
  
## Note
 
 ![note1](../../images/note1.PNG)

 ## Sample test
Create a network policy to allow egress traffic from the Internal application only to the payroll-service and db-service.
Use the spec given below. You might want to enable ingress traffic to the pod to test your rules in the UI.
Also, ensure that you allow egress traffic to DNS ports TCP and UDP (port 53) to enable DNS resolution from the internal pod.
- Policy Name: internal-policy
- Policy Type: Egress
- Egress Allow: payroll
- Payroll Port: 8080
- Egress Allow: mysql
- MySQL Port: 3306

SOLUTION
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      name: internal
  policyTypes:
  - Egress
  - Ingress
  ingress:
    - {}
  egress:
  - to:
    - podSelector:
        matchLabels:
          name: mysql
    ports:
    - protocol: TCP
      port: 3306

  - to:
    - podSelector:
        matchLabels:
          name: payroll
    ports:
    - protocol: TCP
      port: 8080

  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

- Explanation:

Target Pods:
This policy applies to all pods in the default namespace with the label name: internal.

Ingress:
All incoming traffic is allowed to these pods. This is typically needed for UI-based testing during labs.

In production, you should restrict ingress to only trusted sources.

Egress:
Outbound traffic is restricted to:

Pods labeled name: mysql on TCP port 3306 (database service)
Pods labeled name: payroll on TCP port 8080 (payroll service)
Any destination on UDP/TCP port 53 (for DNS resolution, required for service discovery in Kubernetes)
DNS Access:
DNS is handled by the kube-dns service, which listens on port 53 for both UDP and TCP:

```
root@controlplane:~> kubectl get svc -n kube-system 
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   18m
root@controlplane:~>
```
#### Additional lecture on [Developing Networking Policies](https://kodekloud.com/topic/developing-network-policies/)

#### K8s Reference Docs
- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/
 
  
  
  
  
