
OS Upgrades:

Scenerio:
- If you had 1 Master node and 3 worker nodes
- Worker 1 has 3 application on 2 application running on it, blue and green, if this worker node went down, application blue -
  would be fine since its also running on another worker node however cause green is only running on 1 worker node, users will be impacted, 
  what will happen?
- If the worker node,comes up quickly the kubelet will place the application on Worker 1 again.
- However if 5 min have passed the kubelet will terminate these pods. 
- of the pods were part of a replicaset, they are recreated on another worker node.
- The time it waits for a pod to come online, is called a 'Pod eviction timeout' default value of 5 min. 
- When a node goes offline, the master node waits 5 min to consider the worker node being dead. 
- When the node comes back online after the podeviction timeout, it comes back blank with no pods on it. 
- so since the green pod was not part of a replicaset, it would automatically disapear since it past the podeviction period and not part of the replicatset.
- Since we do not know that a node will be back online within 5 min or at all.
    - You can drain the nodes to put the pods on other nodes gracefully. they are simply recreated on another node. 
    - kubectl drain node-1, the node which is being upgraded 
    - Node marked as cordorned which means it is un-scheduleble 
    - To bring it back into being active in the cluster, you can do this command "kubectl uncordon node-1
    - kubectl cordon node-1 (marks the node as cordoned, which means unscxhedulable) 

Practice test: OS Upgrade

1. Let us explore the environment first. How many nodes do you see in the cluster? =. kubectl get nodes = 2
2. How many applications do you see hosted on the cluster? = kubectl get deployments = 1
3. Which nodes are the applications hosted on? = kubectl get pod -o wide = controlplane & node1 
4. We need to take node01 out for maintenance. Empty the node of all applications and mark it unschedulable. 
   below error:
    kubectl drain node01
    node/node01 cordoned
    error: unable to drain node "node01" due to error:cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-system/kube-flannel-ds-9wmgm, kube-system/kube-proxy-n79c4, continuing command...
    There are pending nodes to be drained:
    node01
    cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-system/kube-flannel-ds-9wmgm, kube-system/kube-proxy-n79c4
    = kubectl drain node01 --ignore-daemonsets
5. What nodes are the apps on now? = controlplane
6. The maintenance tasks have been completed. Configure the node node01 to be schedulable again = kubectl uncordon node01
7. How many pods are scheduled on node01 now? = 0
8. Why are there no pods on node01? only when new pods are created, they will be scheduled
9. Why are the pods placed on the controlplane node? controlplane node does not have any taint
10. ////
11. We need to carry out a maintenance activity on node01 again. Try draining the node again using the same command as before: 
    kubectl drain node01 --ignore-daemonsets 
12. Why did the drain command fail on node01? It worked the first time! = There is 1 pod in there, which isnt part of replicaset
13. What is the name of the POD hosted on node01 that is not part of a replicaset? = 
14. What would happen to hr-app if node01 is drained forcefully? A forceful drain of the node will delete any pod that is not part of a replicaset.
15. /////
16. hr-app is a critical app and we do not want it to be removed and we do not want to schedule any more pods on node01.
    Mark node01 as unschedulable so that no new pods are scheduled on this node.
    = kubectl cordon node01

Make sure that hr-app is not affected.

kubernetes version:

example:
v1.11.3
1 = Major version
11 = minor version
3 = patch

Cluster Process:

- Each kubernetes component have their version of software
- kube-api-server
  controller-manager 
  kube-scheduler
  kubelet
  kube-proxy
  kubectl
  etcd cluster
  CoreDNS
  
- Each componenet can run on different versions not mandotory
- kube-api server is the main component where all other components talk to, they cannot be higher than the Kube-API Server
- However the kube-controller can be a version higher than kube api-server or even same leve. 
- recommended approach when upgrading is to upgrade 1 minor version at a time 
- If cluster is managed on cloud providers, for example google kubernetes engine lets you upgrade with few clicks
- if cluster is deployed kube-adm, tools can help you upgrade from stratch.
- if deployed from scratch, manually upgrade each components. 
- upgrading 2 concepts:
  - upgrade master nodes
  - then worker nodes
- whilst master nodes are being upgraded, control plane components such as API Server, sheduler and controller manager go down briefly. 
- master being down doesnt mean worker nodes are affected, all work loads continue serving request, during this time you cannot deploy new appications or -
  delete or configure existing ones.
- Since Master is down, all mgmt functions will be down and cannot access cluster using kubectl or other kube api, create new applications. 
- if pod was to fail during this time, new pods wont come up. 
- once master comes up, it should function properly again.

Strategies in upgrading cluster:
Solution 1:

- Upgrading all worker nodes at once, however all workloads are down and users can no longer access the application.
  once nodes are back up, everything should function as normal.
  
Solution 2:

- Upgrade 1 node at a time 
  - upgrade 1 node at a time
  - workloads move to other nodes
 

Solution 3:

- Add new nodes to the cluster
- These nodes with newer software more convenient when your on a cloud enviroment where you can easily provision new nodes and deprevision old ones.
- move workloads to the new node and then delete old one

Kubeadm upgrade plan command:

- kubeadm does not install or upgrade kubelet
- must also upgrade kubeadm also 

To upgrade worker node via kubadm:

1. Drain node
2. Upgrade kubeadm
3. Upgrade kubelet
4. restart kubelet service
5. kubectl uncordon

Practice test: Cluster Upgrade

1. This lab tests your skills on upgrading a kubernetes cluster. We have a production cluster with applications running on it. 
   Let us explore the setup first. What is the current version of the cluster? = kubeadm upgrade plan v1.19.0
2. How many nodes are part of this cluster? kubectl get nodes = 2
3. How many nodes can host workloads in this cluster? = 2 since no taints places on Masternode
4. How many applications are hosted on the cluster? = kubectl get deployments --all-namespaces = 1
5. What nodes are the pods hosted on? = Controlplane & node01
6. You are tasked to upgrade the cluster. User's accessing the applications must not be impacted. 
   And you cannot provision new VMs. What strategy would you use to upgrade the cluster = Upgrade 1 node at a time and move workloads to other nodes
7. What is the latest stable version available for upgrade? = Latest stable version: v1.19.16
8. We will be upgrading the master node first. Drain the master node of workloads and mark it UnSchedulable
   = kubectl drain controlplane --ignore-daemonsets
9. Upgrade the controlplane components to exact version v1.20.0, Upgrade kubeadm tool (if not already), then the master components, 
   and finally the kubelet. Practice referring to the kubernetes documentation page. 
   Note: While upgrading kubelet, if you hit dependency issue while running the apt-get upgrade kubelet command, 
   use the apt install kubelet=1.20.0-00 command instead
   
On the controlplane node, run the command run the following commands:

apt update
This will update the package lists from the software repository.

apt install kubeadm=1.20.0-00
This will install the kubeadm version 1.20

kubeadm upgrade apply v1.20.0
This will upgrade kubernetes controlplane. Note that this can take a few minutes.

apt install kubelet=1.20.0-00 This will update the kubelet with the version 1.20.

You may need to restart kubelet after it has been upgraded.
Run: systemctl restart kubelet


10. Mark the controlplane node as "Schedulable" again = kubectl uncordon controlplane 
11. Next is the worker node. Drain the worker node of the workloads and mark it UnSchedulable = kubectl drain node01 --ignore-daemonsets12.
12. Upgrade the worker node to the exact version v1.20.0 = 

On the node01 node, run the command run the following commands:

If you are on the master node, run ssh node01 to go to node01


apt update 
This will update the package lists from the software repository.

apt install kubeadm=1.20.0-00
This will install the kubeadm version 1.20

kubeadm upgrade node
This will upgrade the node01 configuration.

apt install kubelet=1.20.0-00 This will update the kubelet with the version 1.20.

You may need to restart kubelet after it has been upgraded.
Run: systemctl restart kubelet

Type exit or enter CTL + d to go back to the controlplane node.


13. Remove the restriction and mark the worker node as schedulable again.
    = kubectl uncordon node01


**Backup and Restore Method**:

- All config stored on ETCD
- Better approach in backing up is querying the kube api server saving all resources created in a cluster.
- 1 command can be used such as:
  - kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml 
- ETCD:
  - stores state on cluster
  - hosted on master node
  - ETCD comes built in with snapshot solution 
  - To restore the ETCD cluster, need to stop the kube-api service than run the etcd snapshot restore command  
  - when restore occurs. it brings up a brand new cluster and stores it as if it is new prevent new member from joining cluster.
  - restart deamon and etcd. followed by api-service
  

Practice test: Backup and restore methods:

1. We have a working kubernetes cluster with a set of applications running. Let us first explore the setup.
   How many deployments exist in the cluster? kubectl get deployments --all-namespaces = 2
   
2. What is the version of ETCD running on the cluster? kubeadm upgrade plan = 3.5.1
3. At what address can you reach the ETCD cluster from the controlplane node?
   Check the ETCD Service configuration in the ETCD POD = kubectl  describe pod etcd-controlplane -n kube-system | grep listen-client
   Use the command kubectl describe pod etcd-controlplane -n kube-system and look for --listen-client-urls = 127.0.0.1:2379
4. Where is the ETCD server certificate file located? Note this path down as you will need to use it later =
5. 
6. The master node in our cluster is planned for a regular maintenance reboot tonight. While we do not anticipate anything to go wrong, we are required to take the necessary backups. 
   Take a snapshot of the ETCD database using the built-in snapshot functionality.
   Store the backup file at location /opt/snapshot-pre-boot.db
   ```yaml
   root@controlplane:~# ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key \
   snapshot save /opt/snapshot-pre-boot.db
   Snapshot saved at /opt/snapshot-pre-boot.db
   root@controlplane:~# 
   ```
7. 
8. Wake up! We have a conference call! After the reboot the master nodes came back online, but none of our applications are accessible. 
   Check the status of the applications on the cluster. What's wrong? 
   = all of the above
9. Luckily we took a backup. Restore the original state of the cluster using the backup file. = 
=
First Restore the snapshot:
```yaml
root@controlplane:~# ETCDCTL_API=3 etcdctl  --data-dir /var/lib/etcd-from-backup \
snapshot restore /opt/snapshot-pre-boot.db

2022-03-25 09:19:27.175043 I | mvcc: restore compact to 2552
2022-03-25 09:19:27.266709 I | etcdserver/membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32

```
Note: In this case, we are restoring the snapshot to a different directory but in the same server where we took the backup (the controlplane node) As a result, the only required option for the restore command is the --data-dir.


Next, update the /etc/kubernetes/manifests/etcd.yaml:
We have now restored the etcd snapshot to a new path on the controlplane - /var/lib/etcd-from-backup, so, the only change to be made in the YAML file, is to change the hostPath for the volume called etcd-data from old directory (/var/lib/etcd) to the new directory (/var/lib/etcd-from-backup).
```yaml
  volumes:
  - hostPath:
      path: /var/lib/etcd-from-backup
      type: DirectoryOrCreate
    name: etcd-data
```
With this change, /var/lib/etcd on the container points to /var/lib/etcd-from-backup on the controlplane (which is what we want)
When this file is updated, the ETCD pod is automatically re-created as this is a static pod placed under the /etc/kubernetes/manifests directory.
Note 1: As the ETCD pod has changed it will automatically restart, and also kube-controller-manager and kube-scheduler. Wait 1-2 to mins for this pods to restart. You can run a watch "docker ps | grep etcd" command to see when the ETCD pod is restarted.
Note 2: If the etcd pod is not getting Ready 1/1, then restart it by kubectl delete pod -n kube-system etcd-controlplane and wait 1 minute.
Note 3: This is the simplest way to make sure that ETCD uses the restored data after the ETCD pod is recreated. You don't have to change anything else.
If you do change --data-dir to /var/lib/etcd-from-backup in the YAML file, make sure that the volumeMounts for etcd-data is updated as well, with the mountPath pointing to /var/lib/etcd-from-backup (THIS COMPLETE STEP IS OPTIONAL AND NEED NOT BE DONE FOR COMPLETING THE RESTORE)


