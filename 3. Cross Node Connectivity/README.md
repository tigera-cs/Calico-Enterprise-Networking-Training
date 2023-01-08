
# In this lab

This lab provides the instructions to:

* [Overview](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/tree/main/3.%20Cross%20Node%20Connectivity#overview)
* [Examine pod network connectivity across cluster nodes using CE VXLAN mode](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/tree/main/3.%20Cross%20Node%20Connectivity#examine-pod-network-connectivity-across-cluster-nodes-using-ce-vxlan-mode)
* [Configure an externally routable IPPool](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/tree/main/3.%20Cross%20Node%20Connectivity#configure-an-externally-routable-ippool)
* [Configure Calico Enterprise BGP Peering to connect with an upsteam router outside the cluster](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/tree/main/3.%20Cross%20Node%20Connectivity#configure-calico-enterprise-bgp-peering-to-connect-with-an-upsteam-router-outside-the-cluster)
* [Configure a namespace to use an externally routable IP addresses](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/tree/main/3.%20Cross%20Node%20Connectivity#configure-a-namespace-to-use-an-externally-routable-ip-addresses)

### Overview

When it comes to pod networking, a pod either needs to talk to other pods that are running on the same node or different nodes. If the communicating pods are collocated on the same node, it is called intrahost networking. In this scenario, pods are connected together through a layer 2 or layer 3 device on the host depending on the CNI solution used. However, pods traffic stays within the same node no matter if the pods are connected using a layer 2 or layer 3 device.

If the communicating pods are located on the different nodes, it is called interhost networking. In this case, pods communicate with each other using their IP addresses without any nat. In this scenario, network must be configured such that pods located on different nodes can communicate with each other. Depending on the CNI provider, network can be configured in layer 2, layer 3, or overlay networking mode for the pods to communicate with each. Calico uses the same logic for intra-host pod traffic and Inter-host pod traffic. All traffic between the pods must follow a L3 routed path.

Calico has got a modular architecture and supports various networking deployment options so that you can select the best networking approach for your specific environment and needs. 

Calico can operate in two networking modes:

* Non-Overlay or flat mode uses bgp and is typically used in on-premise deployments
* Overlay network is a network that is layered on top of another network. In the context of Kubernetes, an overlay network can be used to handle pod-to-pod traffic between nodes on top of an underlying network that is not aware of pod IP addresses. Calico supports two different overlay technologies: IPIP and VXLAN


_______________________________________________________________________________________________________________________________________________________________________

### Examine pod network connectivity across cluster nodes using CE VXLAN mode

VXLAN is one of the two supported overlay technologies in Calico Enterprise CNI implementation. VXLAN implementation in Calico Enterprise can use bird (BGP agent) or Felix as the control plane to distribute routing information. Calico Enterprise by default deploys VXLAN overlay using Felix. In this section, we will learn about Calico Enterprise VXLAN implementation by examining the relevant configurations in the lab.

1. Networking mode used in the cluster is a configuration specified in the IPPool resource. Let's examine the default IPPool. The two configuration parameters `ipipMode: Never` and `vxlanMode: Never` tells us that this cluster is configured to use native bgp to exchange routing information.

```
kubectl get ippools default-ipv4-ippool -o yaml
```

```
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  creationTimestamp: "2022-12-29T22:21:11Z"
  name: default-ipv4-ippool
  resourceVersion: "1722"
  uid: 3f053fa7-0d94-4d6d-94e6-e6582d80c0b0
spec:
  allowedUses:
  - Workload
  - Tunnel
  blockSize: 26
  cidr: 10.48.0.0/24
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Never
```

2. Let's ssh into one of the cluster nodes and examine the current networking configurations of the node.

```
ssh control1
```

3. Check the network interface on the node and make sure there is no VXLAN interface.

```
ip -c link
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:79:d5:90:06:0b brd ff:ff:ff:ff:ff:ff
    altname enp0s5
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:ba:ae:65:54 brd ff:ff:ff:ff:ff:ff
4: cali1bff47cba53@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
10: calic628ed26868@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
11: calic7ceee6b788@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
12: cali4b871599db0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 3
13: caliae0f159e5b3@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 4
14: cali2135bcd7981@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 5
```

4. Let's examine the node routing table. We do not see any routes related to VXLAN in the following routing table (You can compare this output with the output of the node routing table when VXLAN is enabled below). We also see that bird is used to exchange routing information on this node. One example of such route is `10.48.0.192/26 via 10.0.1.30 dev ens5 proto bird `. 

```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.20 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.20 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.20 metric 100 
10.48.0.0/26 via 10.0.1.31 dev ens5 proto bird 
10.48.0.128 dev cali1bff47cba53 scope link 
blackhole 10.48.0.128/26 proto bird 
10.48.0.130 dev calic628ed26868 scope link 
10.48.0.131 dev calic7ceee6b788 scope link 
10.48.0.132 dev cali4b871599db0 scope link 
10.48.0.133 dev caliae0f159e5b3 scope link 
10.48.0.134 dev cali2135bcd7981 scope link 
10.48.0.192/26 via 10.0.1.30 dev ens5 proto bird 
10.48.128.0/26 via 10.0.1.31 dev ens5 proto bird 
10.48.128.192/26 via 10.0.1.30 dev ens5 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
```
5. Exit the ssh mode from control1.

```
exit
```

6. Use the following command to configure the default IPPool to use vxlanMode by setting `vxlanMode: Always`. Then save and exit.

```
kubectl edit ippools default-ipv4-ippool
```

You should see an output similar to the following.

```
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  creationTimestamp: "2022-12-29T22:21:11Z"
  name: default-ipv4-ippool
  resourceVersion: "232027"
  uid: 3f053fa7-0d94-4d6d-94e6-e6582d80c0b0
spec:
  allowedUses:
  - Workload
  - Tunnel
  blockSize: 26
  cidr: 10.48.0.0/24
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Always
```

7. Let's check on the configured interfaces on the node resource. We can see that all the nodes have got a VXLAN interface. This interface was added after we enabled VXLAN networking in the IPPool resource. calico-node watches the IPPool resource and as soon as the resource is configured to use VXLAN, it will configure the VXLAN device on the nodes. Make note of each node's VXLAN interface IP address. We will use them in the next steps.

```
kubectl get nodes -o yaml | grep -B1 -i vxlan
```
```
      projectcalico.org/IPv4Address: 10.0.1.20/24
      projectcalico.org/IPv4VXLANTunnelAddr: 10.48.0.135
--
      projectcalico.org/IPv4Address: 10.0.1.30/24
      projectcalico.org/IPv4VXLANTunnelAddr: 10.48.0.206
--
      projectcalico.org/IPv4Address: 10.0.1.31/24
      projectcalico.org/IPv4VXLANTunnelAddr: 10.48.0.46
```


8. ssh into control1 again.

```
ssh control1
```

9. Check the network interface on the node again. This time you should see a vxlan interface named `vxlan.calico`. The last interface in the following output.

```
ip -c link
```

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:79:d5:90:06:0b brd ff:ff:ff:ff:ff:ff
    altname enp0s5
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:ba:ae:65:54 brd ff:ff:ff:ff:ff:ff
4: cali1bff47cba53@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
10: calic628ed26868@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
11: calic7ceee6b788@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
12: cali4b871599db0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 3
13: caliae0f159e5b3@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 4
14: cali2135bcd7981@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 5
17: vxlan.calico: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8951 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/ether 66:c9:ee:b1:9b:47 brd ff:ff:ff:ff:ff:ff
```

10. Let's examine the node routing table again. We can see routes related to VXLAN overlay networking as indicted by `10.48.0.192/26 via 10.48.0.206 dev vxlan.calico onlink ` and routes related to native bgp networking as indicated by `10.48.128.0/26 via 10.0.1.31 dev ens5 proto bird `. The reason for having both VXLAN and native bgp networking in this cluster is that we have two IPPools one of which is configured to use VXLAN and the other is configured to use native bgp.

```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.20 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.20 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.20 metric 100 
10.48.0.0/26 via 10.48.0.46 dev vxlan.calico onlink 
10.48.0.128 dev cali1bff47cba53 scope link 
blackhole 10.48.0.128/26 proto 80 
10.48.0.130 dev calic628ed26868 scope link 
10.48.0.131 dev calic7ceee6b788 scope link 
10.48.0.132 dev cali4b871599db0 scope link 
10.48.0.133 dev caliae0f159e5b3 scope link 
10.48.0.134 dev cali2135bcd7981 scope link 
10.48.0.192/26 via 10.48.0.206 dev vxlan.calico onlink 
10.48.128.0/26 via 10.0.1.31 dev ens5 proto bird 
10.48.128.192/26 via 10.0.1.30 dev ens5 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
```
11. Felix on each node is watching the node objects in the datastore to learn about the VXLAN tunnel endpoints on remote nodes as well as to learn their IP addresses and MAC addresses. Felix will then program the local node with the following information on each node in the cluster:

* A static ARP entry, visible with `ip neigh show`
* Static Bridge FDB entry, visible with `bridge fdb show`

Run the following command to see the relevant `VXLAN` ARP entries. Note the followings:
* `10.48.0.206` is the IP address and `66:14:9d:f4:2c:1f` is the mac address of `vxlan.calico` on worker1
* `10.48.0.46` is the IP address and `66:4b:ff:e1:ef:01` is the mac address of `vxlan.calico` on worker2

```
ip neigh show
```
```
10.48.0.206 dev vxlan.calico lladdr 66:14:9d:f4:2c:1f PERMANENT
10.0.1.10 dev ens5 lladdr 02:88:a0:32:73:4d REACHABLE
10.48.0.46 dev vxlan.calico lladdr 66:4b:ff:e1:ef:01 PERMANENT
10.0.1.1 dev ens5 lladdr 02:04:7a:86:55:c1 REACHABLE
10.48.0.134 dev cali2135bcd7981 lladdr fa:67:13:92:26:c8 REACHABLE
10.48.0.130 dev calic628ed26868 lladdr 52:12:d1:4c:04:20 REACHABLE
10.48.0.133 dev caliae0f159e5b3 lladdr a6:0b:a3:07:ca:a3 REACHABLE
10.0.1.30 dev ens5 lladdr 02:85:fc:c5:96:85 REACHABLE
10.48.0.131 dev calic7ceee6b788 lladdr d6:c0:bc:a2:8f:49 DELAY
10.0.1.31 dev ens5 lladdr 02:df:0c:da:8e:b1 REACHABLE
10.48.0.132 dev cali4b871599db0 lladdr 76:17:69:d0:15:04 REACHABLE
```

12. Run the following command to see the relevant bridge forward dabase enteries. Note the last two enteries that are relevant to the VXLAN networking in our scenario.
* `66:14:9d:f4:2c:1f` is the mac address of `vxlan.calico` on worker1 and is accessible via `10.0.1.30`, which is the fabric facing network interface of worker1
* `66:4b:ff:e1:ef:01` is the mac address of `vxlan.calico` on worker2 and is accessible via `10.0.1.31`, which is the fabric facing network interface of worker2

```
bridge fdb show
```
```
01:00:5e:00:00:01 dev ens5 self permanent
01:80:c2:00:00:00 dev ens5 self permanent
01:80:c2:00:00:03 dev ens5 self permanent
01:80:c2:00:00:0e dev ens5 self permanent
33:33:00:00:00:01 dev ens5 self permanent
33:33:ff:90:06:0b dev ens5 self permanent
33:33:00:00:00:01 dev docker0 self permanent
01:00:5e:00:00:6a dev docker0 self permanent
33:33:00:00:00:6a dev docker0 self permanent
01:00:5e:00:00:01 dev docker0 self permanent
02:42:ba:ae:65:54 dev docker0 vlan 1 master docker0 permanent
02:42:ba:ae:65:54 dev docker0 master docker0 permanent
33:33:00:00:00:01 dev cali1bff47cba53 self permanent
01:00:5e:00:00:01 dev cali1bff47cba53 self permanent
33:33:ff:ee:ee:ee dev cali1bff47cba53 self permanent
33:33:00:00:00:01 dev calic628ed26868 self permanent
01:00:5e:00:00:01 dev calic628ed26868 self permanent
33:33:ff:ee:ee:ee dev calic628ed26868 self permanent
33:33:00:00:00:01 dev calic7ceee6b788 self permanent
01:00:5e:00:00:01 dev calic7ceee6b788 self permanent
33:33:ff:ee:ee:ee dev calic7ceee6b788 self permanent
33:33:00:00:00:01 dev cali4b871599db0 self permanent
01:00:5e:00:00:01 dev cali4b871599db0 self permanent
33:33:ff:ee:ee:ee dev cali4b871599db0 self permanent
33:33:00:00:00:01 dev caliae0f159e5b3 self permanent
01:00:5e:00:00:01 dev caliae0f159e5b3 self permanent
33:33:ff:ee:ee:ee dev caliae0f159e5b3 self permanent
33:33:00:00:00:01 dev cali2135bcd7981 self permanent
01:00:5e:00:00:01 dev cali2135bcd7981 self permanent
33:33:ff:ee:ee:ee dev cali2135bcd7981 self permanent
66:14:9d:f4:2c:1f dev vxlan.calico dst 10.0.1.30 self permanent
66:4b:ff:e1:ef:01 dev vxlan.calico dst 10.0.1.31 self permanent
```
13. Finally, Felix on each node learns about the other nodes routing information through the information (IPAM blocks and pod IP address) stored in the datastore and updates its node's routing table.

14. Exit the ssh mode from control1.

```
exit
```

15. Reset the default IPPool configuration to native bgp again by setting `vxlanMode: Never`. Then save and exit.


```
kubectl edit ippools default-ipv4-ippool
```

You should see an output similar to the following.

```
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  creationTimestamp: "2022-12-29T22:21:11Z"
  name: default-ipv4-ippool
  resourceVersion: "245626"
  uid: 3f053fa7-0d94-4d6d-94e6-e6582d80c0b0
spec:
  allowedUses:
  - Workload
  - Tunnel
  blockSize: 26
  cidr: 10.48.0.0/24
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: Never
```

_______________________________________________________________________________________________________________________________________________________________________

### Configure an externally routable IPPool

Calico provides a sophisticated and powerful IPAM solution, which enables you to allocate and manage IP addresses for a variety of use cases and requirements.

One of the use cases of Calico IPPool is to distinguish between different ranges of IP addresses that have different routablity scopes. If you are operating at a large scale, then IP addresses are precious resources. You might want to have a range of IPs that is only routable within the cluster, and another range of IPs that is routable across your enterprise. In that case, you can choose which pods get IPs from which range depending on whether workloads from outside of the cluster need direct access to the pods or not.

In this section, we'll simulate this use case by creating an additional IPPool to represent the externally routable pool.  (And we've already configured the underlying network to not allow routing of the existing IPPool outside of the cluster.)

1. Let's start by creating a new IPPool for `10.48.2.0/24` that is externally routable.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: external-pool
spec:
  cidr: 10.48.2.0/24
  blockSize: 29
  ipipMode: Never
  natOutgoing: true
EOF

```
2. List the currentlly configured IPPools in the cluster.

```
calicoctl get ippools
```
```
NAME                  CIDR             SELECTOR   
default-ipv4-ippool   10.48.0.0/24     all()      
external-pool         10.48.2.0/24     all()      
pool2-ipv4-ippool     10.48.128.0/24   all() 
```

3. We now have the followings:

| CIDR          |  Purpose                                                  |
|---------------|-----------------------------------------------------------|
| 10.48.0.0/16  | Kubernetes Pod Network (via kubeadm `--pod-network-cidr`) |
| 10.48.0.0/24  | Calico - Initial default IPPool                           |
| 10.48.128.0/24| Calico - Seconday IPPool                                  |
| 10.48.2.0/24  | Calico - External IPPool (externally routable)            |
| 10.49.0.0/16  | Kubernetes Service Network (via kubeadm `--service-cidr`) |


_______________________________________________________________________________________________________________________________________________________________________


### Configure Calico Enterprise BGP Peering to connect with an upsteam router outside the cluster

1. Let's start by examining Calico BGP peering status on one of the nodes. There are different methods to find BGP status information, but all these methods require access to `calicoctl` in some form. The reason for this is that bird requires privileged access to the local bird socket to provide status information. In this excerise, we are using `calicoctl` configured as a binary on the node.

SSH into worker1.

```
ssh worker1
```

2. Follow the steps provided in [1. Install Calico Enterprise](https://github.com/Pooriya-a/CalicoEnterprise-Networking-Training/blob/main/1.%20Install%20Calico%20Enterprise/README.md#install-calico-enterprise-command-line-utility-calicoctl) to install `calicoctl` as a binary on a single host.

3. Once `calicoctl` is installed, check the BGP connection status from `worker1` to other nodes (BGPPeers) in the cluster.

```
sudo calicoctl node status
```
```
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+------------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |   SINCE    |    INFO     |
+--------------+-------------------+-------+------------+-------------+
| 10.0.1.20    | node-to-node mesh | up    | 2022-12-29 | Established |
| 10.0.1.31    | node-to-node mesh | up    | 2022-12-29 | Established |
+--------------+-------------------+-------+------------+-------------+

IPv6 BGP status
No IPv6 peers found.
```
4. The above output shows that currently Calico on `worker1` is only peering with the other nodes `control1` and `worker2` in the cluster and is not peering with any router outside of the cluster. Exit `worker1`

```
exit
```


5. Now, let's simulate BGP peering to a router outside of the cluster by peering to `bastion` node. We've already set up `bastion` node to act as a router and it is ready to accept new BGP peering.

If you are interested to see the standalone bird configurations on `bastion` node, run the following command from the `bastion` node.

```
sudo cat /etc/bird/bird.conf
```

6. Let's add a new BGPPeer by running the following command. Get yourself familiar with the GBPPeer resource. `peerIP` is the IP address of the peering router, which is the `bastion` node in this case. `asNumber` is the AS number of the peering router.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-64512
spec:
  peerIP: 10.0.1.10
  asNumber: 64512
EOF

```
You should receive an output similar to the following.

```
bgppeer.projectcalico.org/bgppeer-global-64512 created
```

7. Let's examine the BGP peering from `worker1` node again by doing SSH into the node.

```
ssh worker1
```

8. Check the status of Calico on the node.

```
sudo calicoctl node status
```
```
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 10.0.1.20    | node-to-node mesh | up    | 17:44:47 | Established |
| 10.0.1.31    | node-to-node mesh | up    | 17:44:46 | Established |
| 10.0.1.10    | global            | up    | 20:35:04 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

The output above shows that Calico is now peered with the bastion node (`10.0.1.10`). This means Calico can share routes to and learn routes from `bastion` node. Note that you should receive a similar output if you run the same command on the other cluster nodes.

In a real-world on-prem deployment, you would typically configure Calico nodes within a rack to peer with the ToRs (Top of Rack) routers, and the ToRs are then connected to the rest of the enterprise or data center network. In this way, if desired, pods can be reached from anywhere in then network. You could even go as far as giving some pods public IP address and have them accessible from the Internet if you wanted to.

9. We're done with adding the peers. Exit from `worker1` to return back to bastion node.

```
exit
```

_______________________________________________________________________________________________________________________________________________________________________


### Configure a namespace to use an externally routable IP addresses

Calico supports annotations on both namespaces and pods that can be used to control which IPPool or even which IP address a pod will receive its address from when it is created. In this example, we're going to create a namespace to host externally routable Pods.

1. Let's create a namespace with the required Calico IPAM annotations. Examine the namespace configurations below. Note how Calico IPAM annotation `cni.projectcalico.org/ipv4pools` is used with the name of IPPool `external-pool` to allocate IP addresses from IPPool `external-pool` to pods deployed in this namespace.

```
kubectl apply -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    cni.projectcalico.org/ipv4pools: '["external-pool"]'
  name: external-ns
EOF

```

You should receive an output similar to the following.

```
namespace/external-ns created
```

2. Before deploying nginx to test the routing for pods deployed in `external-pool` IPPool from outside the cluster, let's examine the routing table of `bastion` node. You should receive an output similar to the following. At this point, `bastion` node should have no routing info for `external-pool` IPPool as there has not been any pods receiving their IP addresses from that IPPool.

```
ip route
```

```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100
```


3. Deploy a nginx example pod in the `external-ns` namespace by running the following command.

```
kubectl apply -f -<<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: external-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent
EOF

```

You should receive an output similar to the following.

```
deployment.apps/nginx created
networkpolicy.networking.k8s.io/nginx created
```

4. Check `bastion` node's routing table again. You should have a new routing table entry.

```
ip route
```

```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.48.2.216/29 via 10.0.1.31 dev ens5 proto bird
```

5. Check the IP address that was assigned to our nginx pod.

```
kubectl get pods -n external-ns -o wide
```
```
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE                                      NOMINATED NODE   READINESS GATES
nginx-5ff46477b5-tjrl5   1/1     Running   0          92s   10.48.2.216   ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
```
The output above shows that the nginx pod has an IP address from the externally routable IP Pool.

6. Let's try connectivity to the nginx pod from the bastion node. Please make sure to replace the IP address of nginx pod from your lab in the following command.

```
curl 10.48.2.216

```

This should have succeeded showing that the nginx pod is directly routable on the broader network.

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

7. If you would like to see IP allocation stats from Calico-IPAM, run the following command.

```
calicoctl ipam show

```

There is one IP address in use from the range `10.48.2.0/24`, which is for our nginx pod.

```
+----------+----------------+-----------+------------+------------+
| GROUPING |      CIDR      | IPS TOTAL | IPS IN USE |  IPS FREE  |
+----------+----------------+-----------+------------+------------+
| IP Pool  | 10.48.0.0/24   |       256 | 36 (14%)   | 220 (86%)  |
| IP Pool  | 10.48.2.0/24   |       256 | 1 (0%)     | 255 (100%) |
| IP Pool  | 10.48.128.0/24 |       256 | 3 (1%)     | 253 (99%)  |
+----------+----------------+-----------+------------+------------+
```

> **Congratulations! You have completed `3. Cross Node Connectivity` lab.**

