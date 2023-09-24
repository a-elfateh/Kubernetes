# Cluster Upgrade #
Cluster Upgrade is an important aspect of any Kubernetes Administrator role. The importance relies in fixing major bugs and issues with old patches; thus having to keep up with new versions of Kubernetes is important for assuring the continuity of deployed application.
  
## Important to know
- When upgrading a cluster, in most cases the cluster will contain applications that are currently running in their respective nodes. It's imprtant as an adminstrator to develope a plan for your upgrade; as some workloads are mission ciritcal that don't affords to be down for any kind of reason.
- The following is an upgrade for a Kubeadm setup cluster, with a 1 master node and 1 worker node.

## Steps
1- View the current running Pods in the cluster, keeping in mind how the nodes are distributed accros nodes
  ```
  $ kubectl get pods -o wide
  ```

2- Drain all pods from the first node you want to upgrade, if the node contains a DaemonSet (pass the ```--ignore-daemonsets``` option), if the node contains Pods that are not a part of a Deployment (you have to through the ```--force``` option)
  ```
  $ kubectl drain $node_name
  ```

3- Inspect the nodes distribuition accros nodes again to check where the pods where avicted to
  ```
  $ kubectl get pod -o wide
  ```

4- Update your package manager's list and install the latest/desired Kubeadm & Kubelet versions
  ```
  $ sudo apt update
  $ sudo apt install kubeadm=1.2x.0-00 kubectl=1.2x.0-00
  ```

5- Check the current version of Kubeadm 
  ```
  $ kubeadm version
  ```

6- After upgrading your Kubeadm version, check that your cluster can be upgraded, and fetch the versions you can upgrade to through the ```kubeadm upgrade plan``` command
  ```
  $ kubeadm upgrade plan
  ```

7- Choose the minor-version/patch-version you want to upgrade to and upgrade your cluster to, wait for a minute or mroe till Kubeadm finishes upgrading your node
  ```
  $ kubeadm upgrade apply 1.2x.0
  ```

8- Inspect any pod of the kube-system namespace, and look for the image version to assure the upgrade process was successfull
  ```
  $ kubectl describe pod -n kube-system $pod_name
  ```

**If you ran the ``` kubectl get node ``` command, you will notice that the node version still hasn't been changed; that's because the version that is displayed is the kubelet version and there's a couple of steps to do to finish kubelet's upgrade.**

8- Reload ``` Systemd ``` unit files thorugh, and restart the kubelet service
  ```
  $ sudo systemctl daemon-reload
  $ sudo systemctl restart kubelet
  ```

9- Inspect the nodes version again, and you will notice that the node version has been change to the version you've upgraded to
  ```
  $ kubectl get nodes
  ```
10- Change the node's state from Unschedulable to Schedulable again so it can be ready for accepting pods upon creation.
  ```
  $ kubectl uncordon $node_name
  ```
11- Repeat the entire steps to upgrade the remaining nodes

## Video Demonstration

