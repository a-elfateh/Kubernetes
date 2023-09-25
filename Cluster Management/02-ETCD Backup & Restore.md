# ETCD Backup & Restore #

## What is ETCD?
etcd is a key value store. In Kubernetes, it's responsible of storing all information about the cluster.

###  What type of information etcd stores?
- information regarding nodes, pods, services, namespaces, resources limits... simple everything you use ```kubectl create``` will insert data in etcd, and all ```kubectl get``` or ```kubectl describe``` will fetch data from etcd.

### The types of etcd deployments in a Kubernetes kubeadm setup:
