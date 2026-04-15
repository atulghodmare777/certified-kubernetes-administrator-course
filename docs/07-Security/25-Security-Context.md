# Security Context
  - Take me to [Video Tutorial](https://kodekloud.com/topic/security-contexts-2/)
  
In this section, we will take a look at security context
```
docker run ubuntu sleep 3600
```
- Host and container share same kernel
- But container is deployed in a saperate namespace
- Hence container can only view the processes present in its namespace not other namespace or not the host processes
- Id we run the cmd in the container > ps aux > we get a process id for sleep 3600 process exa: xx
- But if we run ps aux in the host then for same process sleep 3600 we will get different process id exa: xy
- Thats how docker isolates processes in the system
- Docker host have multiple users but container have by default root user
- Is we dont want root user for the container then we can provide the user as present in below command or we can provide in the docker file > USER 1001
  
## Container Security
 ```
 $ docker run --user=1001 ubuntu sleep 3600 > this will create container with non root user which is 1001
 ```

 - What if we run the container with root user will it same as root user in docker host
 - Can the root user in container perform same action as root user in docker host
 - Docker implements the security features by default to contain the root user of container with limited permissions.
 - We can see the host root user capabilities at location > /usr/include/linux/capability.h
 - By default docker run with limited scope of capabilities.
 - If we want to modify this behaviour and provide additional capabilities then we do that by following cmd:
```
 $ docker run -cap-add MAC_ADMIN ubuntu > this will add the container previlages
```

- We can drop previlages as well by using following command:
  
```
$ docker run -cap-drop KILL ubuntu
```

- If want to provide container all the previlages as host then run following cmd:
```
$ docker run --privilaged ubuntu
```

 ![csec](../../images/csec.PNG)
 
## Kubernetes Security
- You may choose to configure the security settings at a container level or at a pod level.

 ![ksec](../../images/ksec.PNG)

## Security Context
- To add security context on the pod add a field called **`securityContext`** under the spec section.
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: web-pod
  spec:
    securityContext:
      runAsUser: 1000
    containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
  ```
  ![sxc1](../../images/sxc1.PNG)
  
- To set the same context at the container level, then move the whole section under container section.
  
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: web-pod
  spec:
    containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
  ```
  ![sxc2](../../images/sxc2.PNG)
  
- To add capabilities use the **`capabilities`** option
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: web-pod
  spec:
    containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000
        capabilities: 
          add: ["MAC_ADMIN"]
  ```
  ![cap](../../images/cap.PNG)

  ### Multiple context present in the pod
  - If multiple security context present in the pod exa:
```
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  securityContext:
    runAsUser: 1001
  containers:
  -  image: ubuntu
     name: web
     command: ["sleep", "5000"]
     securityContext:
      runAsUser: 1002

  -  image: ubuntu
     name: sidecar
     command: ["sleep", "5000"]
```
- In this web container will run with 1002 user & sidecar container will run at 1001 user as container level user will take precedence
- If want to provide user as root then we have to add zero(0) as its value then it will take it as root user
- If want to add the capabilities then we have to add at the container level only as at pod level it is not supported
  
```
securityContext:
  runAsUser: 0
  capabilities:
     add: ["SYS_TIME", "NET_ADMIN"]
```
  
  
### K8s Reference Docs
- https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
