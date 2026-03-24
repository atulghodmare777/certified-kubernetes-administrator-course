In this section, we will take a look at backup and restore methods

## Backup Candidates
 
 ![bc](../../images/bc.PNG)
 
## Resource Configuration
- Imperative way
  
  ![rci](../../images/rci.PNG)

- Declarative Way (Preferred approach)
  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: myapp-pod
    labels:
      app: myapp
      type: front-end
  spec:
    containers:
    - name: nginx-container
      image: nginx
  ```
 ![rcd](../../images/rcd.PNG)
 
- A good practice is to store resource configurations on source code repositories like github.

  ![rcd1](../../images/rcd1.PNG)

## Backup - Resource Configs

  ```
  $ kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml (only for few resource groups)
  ```

- There are many other resource groups that must be considered. There are tools like **`ARK`** or now called **`Velero`** by Heptio that can do this for you.

  ![brc](../../images/brc.PNG)
  
## Backup - ETCD
- So, instead of backing up resources as before, you may choose to backup the ETCD cluster itself. 
  
  ![be](../../images/be.PNG)
  
- You can take a snapshot of the etcd database by using **`etcdctl`** utility snapshot save command.
  ```
  $ ETCDCTL_API=3 etcdctl snapshot save snapshot.db
  ```
  ```
  $  ETCDCTL_API=3 etcdctl snapshot status snapshot.db
  ```
  ![be1](../../images/be1.PNG)
  
## Restore - ETCD
- To restore etcd from the backup at later in time. First stop kube-apiserver service
  ```
  $ service kube-apiserver stop
  or
  $ mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
    sleep 30
  ```
- Run the etcdctl snapshot restore command
  ```
  etcdctl snapshot restore snapshot.db --data-dir /var/lib/etcd-from-backup
  ```
  It will create a new directory /var/lib/etcd-from-backup
- Update the etcd service: Update the new directory in the service
  ```
  vi /etc/kubernetes/manifests/etcd.yaml
  Modify the volumes section as follows:
  From:

  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd                    # OLD directory
      type: DirectoryOrCreate
    name: etcd-data
  To:

  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd-from-backup        # NEW restored directory
      type: DirectoryOrCreate
    name: etcd-data
  ```
  - Upon saving this file, the etcd pod will restart automatically due to static pod behavior.
  
- Restart kubeapi server
  ```
  mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
  ```
  Wait for 60 seconds to allow the kube-apiserver to start.
  
 - Restart other control plane components
   ```
   # Restart kube-controller-manager
   mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
   sleep 20
   mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/

   # Restart kube-scheduler
   mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
   sleep 20
   mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/

   # Restart kubelet
   systemctl restart kubelet
  
  ![er](../../images/er.PNG)

- Monitor the restart process
  ```
  watch crictl ps
  ```
  Key indicators to observe:
  All components should show STATUS = Running
  The entire process should take approximately 2-3 minutes.

-Verify the restore
```
# Check all resources across all namespaces
kubectl get deployments,services --all-namespaces

# Verify specific resources if needed
kubectl get pods --all-namespaces
kubectl get nodes
```
You should now observe all the resources that existed at the time the snapshot was taken.

#### With all etcdctl commands running above specify the cert,key,cacert and endpoint for authentication as below then only it will work.
```
$ ETCDCTL_API=3 etcdctl \
  snapshot save /tmp/snapshot.db \
  --endpoints=https://[127.0.0.1]:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/etcd-server.crt \
  --key=/etc/kubernetes/pki/etcd/etcd-server.key
```

  ![erest](../../images/erest.PNG)


## Working with ETCDCTL and ETCDUTL
WORKING WITH ETCDCTL & ETCDUTL
etcdctl is a command line client for etcd.
In all our Kubernetes hands-on labs, the ETCD key-value database is deployed as a static pod on the master. The version used is v3.

To make use of etcdctl for tasks such as backup, verify it is running on API version 3.x:
```
etcdctl version
```

Example:

controlplane ~ ➜  etcdctl version
etcdctl version: 3.5.16
API version: 3.5
Backing Up ETCD
Using etcdctl (Snapshot-based Backup)

- To take a snapshot from a running etcd server, use:
```
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot.db
```

Required Options
--endpoints points to the etcd server (default: localhost:2379)

--cacert path to the CA cert

--cert path to the client cert

--key path to the client key

## Using etcdutl (File-based Backup)

For offline file-level backup of the data directory:

```
etcdutl backup \
  --data-dir /var/lib/etcd \
  --backup-dir /backup/etcd-backup
```
This copies the etcd backend database and WAL files to the target location.

# Checking Snapshot Status

You can inspect the metadata of a snapshot file using:

```
etcdctl snapshot status /backup/etcd-snapshot.db \
  --write-out=table
```
This shows details like size, revision, hash, total keys, etc. It is helpful to verify snapshot integrity before restore.

# Restoring ETCD

Using etcdutl

To restore a snapshot to a new data directory:
```
etcdutl snapshot restore /backup/etcd-snapshot.db \
  --data-dir /var/lib/etcd-restored
```
To use a backup made with etcdutl backup, simply copy the backup contents back into /var/lib/etcd and restart etcd.

# Notes
etcdctl snapshot save is used for creating .db snapshots from live etcd clusters.

etcdctl snapshot status provides metadata information about the snapshot file.

etcdutl snapshot restore is used to restore a .db snapshot file.

etcdutl backup performs a raw file-level copy of etcd’s data and WAL files without needing etcd to be running.
  
#### K8s Reference Docs
- https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/


 
