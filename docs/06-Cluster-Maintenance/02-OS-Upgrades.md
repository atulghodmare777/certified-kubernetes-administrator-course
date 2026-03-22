# OS Upgrades
  - Take me to [Video Tutorial](https://kodekloud.com/topic/os-upgrades/)
  
In this section, we will take a look at OS upgrades.

#### If the node was down for more than 5 minutes, then the pods are terminated from that node

  ![os](../../images/os.PNG)
  
- You can purposefully **`drain`** the node of all the workloads so that the workloads are moved to other nodes.
  ```
  $ kubectl drain node-1
  ```
- The node is also cordoned or marked as unschedulable.
- When the node is back online after a maintenance, it is still unschedulable. You then need to uncordon it.
  ```
  $ kubectl uncordon node-1
  ```
- There is also another command called cordon. Cordon simply marks a node unschedulable. Unlike drain it does not terminate or move the pods on an existing node.

  ![drain](../../images/drain.PNG)

 ## While draining the node if deamonset present then we might get following error msg
  ```
  k drain node01
node/node01 already cordoned
error: unable to drain node "node01" due to error: cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-flannel/kube-flannel-ds-zvwj4, kube-system/kube-proxy-scnfw, continuing command...
There are pending nodes to be drained:
 node01
cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-flannel/kube-flannel-ds-zvwj4, kube-system/kube-proxy-scnfw
```
## During this we need to run following command
 ```
k drain node01 --ignore-daemonsets
node/node01 already cordoned
Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-zvwj4, kube-system/kube-proxy-scnfw
evicting pod default/blue-759779556-w6qcm
evicting pod default/blue-759779556-5hzcs
pod/blue-759779556-5hzcs evicted
pod/blue-759779556-w6qcm evicted
node/node01 drained

```
## What will happen to pod who do not have a controller if try to drain
```
We get following error if pod is not controlled by controller
kubectl drain node01 --ignore-daemonsets
node/node01 cordoned
error: unable to drain node "node01" due to error: cannot delete cannot delete Pods that declare no controller (use --force to override): default/hr-app, continuing command...
There are pending nodes to be drained:
 node01
cannot delete cannot delete Pods that declare no controller (use --force to override): default/hr-app
```
It wont delete the pod as pod do not have replica set or deployment
If still want to drain it then we need to use --force but then pod will be lost foreever
  
#### K8s Reference Docs
- https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/
