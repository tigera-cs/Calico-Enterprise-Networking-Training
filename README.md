# Kubernetes Networking Training
This is the hands-on lab guide for Calico Enterprise Networking Training.

## Lab setup

The following diagram depicts the lab setup used for the entire training.

![image](img/1.Lab-Topology.JPG)



Followings is a short description of each node in the lab.

* **Control1** is Kubernetes master node and runs Kubernetes control plane pods.
* **Worker1** is a Kubernetes worker node.
* **Worker2** is a Kubernetes worker node.
* **Bastion** is a node outside the Kubernetes cluster and is used to run kubectl/calicoctl commands. This node also runs a standalone instance of bird and is used for bgppeering in the relevant lab modules fucntioning as an upstream router. Finally, since this node is outside the cluster, it is used to simulate external connectivity when needed.

Followings are the CIDRs used to build the training lab environment.

* 10.0.1.0/24 host CIDR
* 10.48.0.0/16 Kubernetes Pod Network (via kubeadm --pod-network-cidr)
* 10.49.0.0/16 Kubernetes Service CIDR (via kubeadm --service-cidr)

## SSH access to the nodes

To ssh into the lab instances, type the following commands. The associated IP addresses are included in the lab setup diagram.

* ssh control1 
* ssh worker1
* ssh worker2


## Lab modules

[1. Install Calico Enterprise](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/tree/main/1.%20Install%20Calico%20Enterprise)

[2. Implement pod networking using Calico Enterprise CNI and IPAM](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/tree/main/2.%20Implement%20pod%20networking%20using%20Calico%20Enterprise%20CNI%20and%20IPAM)

[3. Cross Node Connectivity](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/tree/main/3.%20Cross%20Node%20Connectivity)

[4. Kubernetes Services and CE Service Advertisement](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/tree/main/4.%20Kubernetes%20Services%20and%20CE%20Service%20Advertisement)

[5. Calico Enterprise Egress Gateway](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/tree/main/5.%20Calico%20Enterprise%20Egress%20Gateway)

[6. Implement Calico Enterprise eBPF](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/tree/main/6.%20Implement%20Calico%20Enterprise%20eBPF)
