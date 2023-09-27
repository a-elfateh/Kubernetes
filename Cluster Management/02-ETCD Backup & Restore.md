# ETCD Backup & Restore 🚀 # 

### What is ETCD? ❓
etcd is a key value store. In Kubernetes, it's responsible of storing all information about the cluster.

###  What type of information etcd stores?
- information regarding nodes, pods, services, namespaces, resources limits... simple everything you use ```kubectl create``` will insert data in etcd, and all ```kubectl get``` or ```kubectl describe``` will fetch data from etcd.

### The types of etcd deployments in a Kubernetes kubeadm setup:
- Stacked Deployment: topology where etcd is stacked on top of the cluster managed by Kubeadm
- External Deployment: etcd is deployed as a systemd service in a seperate server

### Someone might ask why do we need to backup etcd in the first place? 🤔
As we mentioned, etcd stores information about the cluster, nodes, and every resource deployed in that cluster. Meaning, backuping up etcd results in backuping up our entire cluster.

### Backup process of etcd ###
#### 1- Stacked Deployment:
