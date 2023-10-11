# ETCD Backup & Restore 🚀 # 

### What is ETCD? ❓
etcd is a key value store. In Kubernetes, it's responsible of storing all information about the cluster.

###  What type of information etcd stores?
- information regarding nodes, pods, services, namespaces, resources limits... simple everything you use ```kubectl create``` will insert data in etcd, and all ```kubectl get``` or ```kubectl describe``` will fetch data from etcd.

### The types of etcd deployments in a Kubernetes kubeadm setup:
- Stacked Deployment: topology where etcd is stacked on top of the cluster managed by Kubeadm
- External Deployment: etcd is deployed as a systemd service in a seperate server

### Someone might ask why do we need to backup etcd in the first place? 🤔
As we mentioned, etcd stores information about the cluster, nodes, and every resource deployed in that cluster. Meaning, backing up etcd results in backing up our cluster's resources.

### Backup process of etcd ###
#### 1- Stacked Deployment:
This is when etcd shares the same node alongside the other control plane components.
Things to know regarding stacked etcd deployment's backup and restore process:
   - etcd is deployed using a static pod contoroled by Kubeadm where the yaml file is located in the ```/etc/kubernetes/manifests/```
   - ```etcdctl``` utility is used for the backup/restore procedure.
   - The backup and restore methods require TLS verification using certificates located in the ```/etc/kuberneters/pki/etcd```
   - As the etcd pod is located in the same node with the kube-apiserver, ```etcdctl``` will not require passing in the ```endpoints``` option as the kube-apiserver and etcd share the same node/server.

Steps:

1- First check the currnet cluster for all deployed resources accross the default namespaces.
```
$ kubectl get all
```

2- If your current cluster has no resources deployed in the default namespace, please throw couple of resources in there so we can use it as a validater for the backup/restore process.
```
$ kubectl create deploy testing-deployment --image=nginx --replicas=4
```

3- Check that the deployment was created successfully and that's currently running
```
$ kubectl get all
```

**Till this point in our namespace we have 1x Deployment 1x ReplicaSet 4x Pods**

4- Let's set the environment variable needed to run the etcdctl utility.
```
$ export ETCDCTL_API=3
```

5- As we mentioned before, the backup process needes TLS authentication using certificates. We will need to know which certificates etcd is currently using. We will pass in "step 6" these options 
- ```cacert```=/etc/kubernetes/pki/etcd/ca.crt
- ```cert```=/etc/kubernetes/pki/etcd/server.crt
- ```key```=/etc/kubernetes/pki/etcd/server.key

**```etcdctl``` utility -in most cases- requires defining the etcd server location using the ```endpoints``` option, as we are working on the same server where etcd is located, this won't be needed**

```
$ kubectl describe pod -n kube-system etcd | grep -iA 20 command
```
**OR**
```
$ cat /etc/kubernetes/manifests/etcd.yaml | grep -iA 20 command
```
This will show the variables that the etcd pod is using.

6- We will be taking the etcd snapshot using the ```etcdctl snaphost save``` command. 
```
$ etcdctl snapshot save backup_name.db\
--cacert /etc/kubernetes/pki/etcd/ca.crt\
--cert /etc/kubernetes/pki/etcd/server.crt\
--key /etc/kubernetes/pki/etcd/server.key
```

7- After we created the backup, let us now delete the deployment we created earlier to simulate a disaster scenario.
```
$ kubectl delete deploy testing-deployment
```

8- Verify the deletion, you should see no resources
```
$ kubectl get all
```

9- Let us now restore the backup we took.
```
$ etcdctl snapshot restore backup_name.db --data-dir /opt/etcd_restore
```
If the restore was successfull, you should see an output like this

<img width="743" alt="Screenshot 2023-09-30 at 2 58 08 PM" src="https://github.com/a-elfateh/Kubernetes/assets/61758821/b9a66d16-5b00-40a3-9d00-b9044364563a">



10- Finally, edit the static file that the etcd pod reads its configuration from. 
```
$ nano /etc/kubernetes/manifests/etcd.yaml
```

Search/scroll-down for the "Volumes" section at the end of the yaml file, change the ```etcd-data``` volume path from ```/var/lib/etcd``` to ```/opt/etcd_restore``` and save the file using CTRL+O and the pressing enter. Exit the file with CTRL+X
```
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /opt/etcd_restore
      type: DirectoryOrCreate
    name: etcd-data
```
**Please note: it will take about a minute for the changes to take effect on the etcd pod, don't rush it like me 😬**. Check to see if the deployment we created and deleted earlier is restored using the ```kubectl get all```

## Video Demonstration ##



https://github.com/a-elfateh/Kubernetes/assets/61758821/1609b19d-ef23-4bf8-bac9-891604dbd909



#### 2- External Deployment:
This is when the etcd server is located outside the controlplane node. Things to note about this deployment:
- etcd is deployed as a systemd service, where the default config file is located at ```/etc/systemd/system/etcd.service```
- The etcd server exposes by default port **2379** for the etcd service
- TLS authentication happens between any client retrieving information and the etcd server
- As the etcd server is a remote server, we will be needing to specify the ```--endpoints``` option in our commands

Steps:
1- Assuming you're currently in the control-plane node of your cluster, check to see if you have any Pods, Deployments, DeamonSets in your cluster. **If not, please create some; as we'll need them to verify our restore later**

2- Let's first locate the IP address of our external etcd server
```
$ ps -ef | grep etcd
```
You will get an output such as 
```
root        1844    1462  0 18:14 ?        00:03:01 kube-apiserver --advertise-address=192.39.14.9 --allow-privileged=true --authorization-mode=Node,RBAC
--client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd
/ca.pem --etcd-certfile=/etc/kubernetes/pki/etcd/etcd.pem --etcd-keyfile=/etc/kubernetes/pki/etcd/etcd-key.pem --etcd-servers=https://192.39.14.21:2379 --kubelet-
client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-
address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes
/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
--requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-
port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-account-signing-
key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc
/kubernetes/pki/apiserver.key
```
**Notice in the above process, an option is passed called ```--etcd-servers``` with an IP address in its value. This is what we need**

3- We can check if our connection to the etcd server is running by using the following command, where we can get the following options values from the above output:
- [etcd_server_ip] = --etcd-servers
- [ca_cert_location] = --etcd-cafile
- [cert_location] = --etcd-certfile
- [key_location] = --etcd-keyfile
```
$ curl [etcd_server_ip]:2380/version  --cacert [ca_cert_location] --cert [cert_location] --key [key_location]
```
OR
```
$ ETCDCTL_API=3 etcdctl endpoint health --endpoints [etcd_server_ip]:2379 --cacert [ca_cert_location] --cert [cert_location] --key [key_location]
```

4- Now lets use the IP address value of ```--etcd-servers``` we got from the first step and ssh to the machine hosting the etcd server
```
$ ssh user@IP_ADDRESS
```

5- Run a systemd status check of the etcd service and take a look at the resulted output
```
$ systemctl status etcd
```
**You will see an output such as the following**
```
● etcd.service - etcd key-value store
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2023-10-11 09:36:36 UTC; 1h 32min ago
     Docs: https://github.com/etcd-io/etcd
 Main PID: 822 (etcd)
    Tasks: 41 (limit: 251379)
   CGroup: /system.slice/etcd.service
           └─822 /usr/local/bin/etcd --name etcd-server --data-dir=/var/lib/etcd-data --cert-file=/etc/etcd/pki/etcd.pem --key
-file=/etc/etcd/pki/etcd-key.pem --peer-cert-file=/etc/etcd/pki/etcd.pem --peer-key-file=/etc/etcd/pki/etcd-key.pem --trusted-
ca-file=/etc/etcd/pki/ca.pem --peer-trusted-ca-file=/etc/etcd/pki/ca.pem --peer-client-cert-auth --client-cert-auth --initial-
advertise-peer-urls https://192.18.29.11:2380 --listen-peer-urls https://192.18.29.11:2380 --advertise-client-urls https://192
.18.29.11:2379 --listen-client-urls https://192.18.29.11:2379,https://127.0.0.1:2379 --initial-cluster-token etcd-cluster-1 --
initial-cluster etcd-server=https://192.18.29.11:2380 --initial-cluster-state new
```

**The loaded line shows a path to a file, this file is where systemd gets the configuration needed to load the etcd service. Will be needing the path for the ```--cacert```, ```--cert```, and ```key``` files. Press "q" to exit from the systemd status result**

6- Create a backup.db file on the current working directory of your etcd server through ```etcdctl```, passing the following options and their values from the previous step:
- ```--cacert```
- ```--cert```
- ```--key```

```
$ ETCDCTL_API=3 etcdctl snapshot save backup.db --cert /etc/etcd/pki/etcd.pem --key /etc/etcd/pki/etcd-key.pem --cacert /etc/etcd/pki/ca.pem
```

7- Return to the control-plane node
```
$ exit
```

8- Run the ```kubectl get all``` command, and delete any workload you see using the ```kubectl delete``` command

9- Return to the etcd-server using ssh

10- Restore you backup.db file using the ```etcdctl```
```
$ETCDCTL_API=3 etcdctl snapshot restore backup.db --data-dir /opt/etcd_resotre
```

11- Change the ownership and permissions of the /opt/restore file so that the etcd user configured to run the etcd service can access the restore file
```
$ chown -R etcd:etcd /opt/etcd_resotre
```
```
$ chmod -R 755 /opt/etcd_resotre
```

12- Edit the etcd service's config file located in ```/etc/systemd/system/etcd.service```
```
nano /etc/systemd/system/etcd.service
```
**Change the value of the ```--data-dir``` optionn to ```/opt/etcd_restore```. Save the file using CTRL+O and the pressing enter. Exit the file with CTRL+X**

13- Return to the control-plane node using the ```exit``` command, and wait for a minute or two so etcd can restore your cluster.

14- Check your restore
```

```
