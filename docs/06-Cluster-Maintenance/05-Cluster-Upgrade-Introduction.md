# Cluster Upgrade Introduction
  - Take me to [Video Tutorial](https://kodekloud.com/topic/cluster-upgrade-introduction/)
  
#### Is it mandatory for all of the kubernetes components to have the same versions?
- No, The components can be at different release versions.
- Since the kube-api-server is the primary component in the control plane other components should not be major version than kube-apiserver
- Controller manager and kube-scheduler can be 1 version lower than kube-apiserver Exa: If kube-api server is 1.10 then other can 1.9
- kubelet and kubeproxy can be 2 version lower than kube-apiserver. Exa If kube-api server is 1.10 then other can 1.8
- kubectl utility can be 1 version lower or equal or 1 version higher.
- Latest 3 versions are supported 
  
#### At any time, kubernetes supports only up to the recent 3 minor versions
- The recommended approach is to upgrade one minor version at a time.
  
  ![up2](../../images/up2.PNG)
  
#### Options to upgrade k8s cluster
 
  ![opt](../../images/opt.PNG)
  
## Upgrading a Cluster
- Upgrading a cluster involves 2 major steps
  
#### There are different strategies that are available to upgrade the worker nodes
- One is to upgrade all at once. But then your pods will be down and users will not be able to access the applications.
  ![stg1](../../images/stg1.PNG)
- Second one is to upgrade one node at a time. 
  ![stg2](../../images/stg2.PNG)
- Third one would be to add new nodes to the cluster
  ![stg3](../../images/stg3.PNG)
  
## kubeadm - Upgrade master node
- When the master is down during upgrade doesnt mean cluster goes down the traffic is served by the worker nodes only accessing the cluster and cluster
  management such as deploying the app and controller also wont work as master is down during brief period of time.
- kubeadm has an upgrade command that helps in upgrading clusters.
  ```
  $ kubeadm upgrade plan
  ```
  ![kube1](../../images/kube1.png)
  
- Upgrade kubeadm from v1.11 to v1.12 [ We need to upgrade first kubeadm and then upgrade the master each time for each minor version, kubeadm follow k8s version patter ]
  ```
  $ apt-get upgrade -y kubeadm=1.12.0-00
  ```
- Upgrade the cluster
  ```
  $ kubeadm upgrade apply v1.12.0
  ```
- If you run the 'kubectl get nodes' command, you will see the older version. This is because in the output of the command it is showing the versions of kubelets on each of these nodes registered with the API Server and not the version of API Server itself  
  ```
  $ kubectl get nodes
  ```
  
  ![kubeu](../../images/kubeu.PNG)
  
- Upgrade 'kubelet' on the master node
  ```
  $ apt-get upgrade kubelet=1.12.0-00
  ```
- Restart the kubelet
  ```
  $ systemctl restart kubelet
  ```
- Run 'kubectl get nodes' to verify
  ```
  $ kubectl get nodes
  ```
  
  ![kubeu1](../../images/kubeu1.PNG)

  Follow following steps for master upgrade:
  - This will give latest version available for upgrade
  ```
  kubeadm upgrade plan
  ```
  we will get current version exa: 1.33.0 , kubeadm version: 1.33.0, target version: 1.33.10
  
  suppose we want to upgrade to 1.43.0
  - Drain master node
  ```
  kubectl drain controlplane --ignore-daemonsets
  ```
  - Upgrade first kubeadm, first find out what package repo it is using
  ```
  pager /etc/apt/sources.list.d/kubernetes.list
  output: deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /
  ```
  - provide required version which we want to switch to
  ```
  vi /etc/apt/sources.list.d/kubernetes.list
  add 1.34
  output: deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /
  ```
  - Now we should see our version on which we want to upgrade to
  ```
  sudo apt update
  sudo apt-cache madison kubeadm
  Exa oputput:
   kubeadm | 1.34.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
  
  ```
  - Upgrade kubeadm: Modify version accordingly
  ```
  sudo apt-mark unhold kubeadm && \
  sudo apt-get update && sudo apt-get install -y kubeadm='1.34.0-*' && \
  sudo apt-mark hold kubeadm
  # Then check kubeadm version
  kubeadm version
  ```
  - Now Verify the upgrade plan: This command checks that your cluster can be upgraded, and fetches the versions you can upgrade to. It also shows a table with        the component config version states.
  ```
  sudo kubeadm upgrade plan
  ```
  - Choose a version to upgrade to, and run the appropriate command. For example:
  ```
  sudo kubeadm upgrade apply v1.34.0
  ```
  Once the command finishes you should see:
  [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.34.x". Enjoy!

  [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.

  - Even after upgrade you will see older version as it is showing kubelet version
  ```
  k get nodes
  output:
  controlplane ~ ➜  k get nodes
  NAME           STATUS                     ROLES           AGE   VERSION
  controlplane   Ready,SchedulingDisabled   control-plane   64m   v1.33.0
  node01         Ready                      <none>          64m   v1.33.0
  ```
  - Upgrade the kubectl and kubelet
  ```
  sudo apt-mark unhold kubelet kubectl && \
  sudo apt-get update && sudo apt-get install -y kubelet='1.34.0-*' kubectl='1.34.0-*' && \
  sudo apt-mark hold kubelet kubectl
  ```
  - Now restart the service
  ```
  sudo systemctl daemon-reload
  sudo systemctl restart kubelet

  k uncordon controlplane
  ```
  - Now the master node upgrade is successful
  ```
  controlplane ~ ➜  k get nodes
  NAME           STATUS   ROLES           AGE   VERSION
  controlplane   Ready    control-plane   68m   v1.34.0
  node01         Ready    <none>          68m   v1.33.0
  ```
  
  - For the other control plane nodes : Same as the first control plane node but use:
  ```
  sudo kubeadm upgrade node instead of sudo kubeadm upgrade apply
  ```
  
  
## kubeadm - Upgrade worker nodes
  
- From master node, run 'kubectl drain' command to move the workloads to other nodes
  ```
  $ kubectl drain node-1 --ignore-daemonsets
  ```
- SSH into node
  ```
  ssh node01
  ```
- Use any text editor you prefer to open the file that defines the Kubernetes apt repository.
  ```
  vi /etc/apt/sources.list.d/kubernetes.list
  # Update the version in the URL to the next available minor release, i.e v1.34.
  deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /
  ```
- Check if required version is available
  ```
  apt update
  apt-cache madison kubeadm
  ```
- Based on the version information displayed by apt-cache madison, it indicates that for Kubernetes version 1.34.0, the available package version is 1.34.0-1.1.     Therefore, to install kubeadm for Kubernetes v1.34.0, use the following command:
  ```
  $ apt-get install kubeadm=1.34.0-1.1
  # Upgrade the node  
    kubeadm upgrade node
  ```
- Now, upgrade the Kubelet version.
  ```
  apt-get install kubelet=1.34.0-1.1
  ```
- Restart the kubelet service :Run the following commands to refresh the systemd configuration and apply changes to the Kubelet service
  ```
  $ systemctl daemon-reload
  $ systemctl restart kubelet
  ```
- Mark the node back to schedulable
  ```
  $ kubectl uncordon node-1
  ```
  
  ![kubeu2](../../images/kubeu2.PNG)
  
- Upgrade all worker nodes in the same way

  ![kubeu3](../../images/kubeu3.PNG)
  

#### Demo Video on [Cluster Upgrade](https://kodekloud.com/topic/demo-cluster-upgrade/)

#### K8s Reference Docs
- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
- https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-upgrade/
  
