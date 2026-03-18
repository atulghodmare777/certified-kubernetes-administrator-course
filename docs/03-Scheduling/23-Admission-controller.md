When an kubectl utility request any resource modification it by default authenticate and authorise the user using the role concept.
But it have limitations such as it cannot restrict the user for following things:
exa: not pulling image from public, resource should have always metadata present, resource should not run with root user etc.
These can be achived using admission controller.
By default multiple admission controllers are enabled
exa: DefaultStorageClass, always pull image, EventRateLimit, NamespaceExists etc
But we have other admission controllers as well which are in disabled stage. Exa: NamespaceAutoProvisioning
We can find enabled admission controller by following command:
kube-apiserver -h | grep enable-admission-plugins
If running in kubeadm set then use following command:
k exec kube-api-controlplane -n kube-system -- kube-api-server -h | grep enable-admission-plugins
