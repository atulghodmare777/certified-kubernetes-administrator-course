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
 
#### Additional lecture on [Developing Networking Policies](https://kodekloud.com/topic/developing-network-policies/)

#### K8s Reference Docs
- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/
 
  
  
  
  
