# In this lab

This lab provides the instructions to:

* [Overview](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/tree/main/2.%20Implement%20pod%20networking%20using%20Calico%20Enterprise%20CNI%20and%20IPAM#overview)
* [Examine pod network connectivity using Calico Enterprise CNI](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/blob/main/2.%20Implement%20pod%20networking%20using%20Calico%20Enterprise%20CNI%20and%20IPAM/README.md#examine-pod-network-connectivity-using-calico-enterprise-cni)
* [Create an aditional Calico Enterprise IPPool](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/tree/main/2.%20Implement%20pod%20networking%20using%20Calico%20Enterprise%20CNI%20and%20IPAM#create-an-aditional-calico-enterprise-ippool)


### Overview

Calico is one of the most popular open-source CNI plugins available for the Kubernetes ecosystem. Calico provides a performant, scalable, and pluggable CNI solution with advanced network administration capabilities. Like other CNI solutions, Calico CNI configures the pod IP address and updates the pod and node routing table to establish pod networking. However, there are some differences between how various CNI providers implement pod networking in a Kubernetes system. This lab covers how Calico CNI and IPAM implement pod networing and walk you through some of the advanced IPAM capabilities and use cases.



_______________________________________________________________________________________________________________________________________________________________________



### Examine pod network connectivity using Calico Enterprise CNI

As discussed in the previous lab, upstream Kubernetes by default does not provide a network interface plugin. In the previous lab, we deployed Calico Enterprise as the CNI provider, which enables pod networking. In this section, we will focus on Kubernetes pod networking by:   

* Examining what the network looks like from the perspecitve of a pod (the pod network namespace)
* Examining what the network looks like from the perspecitve of the host (the host network namespace)

1. Each pod gets its own Linux network namespace, which you can think of as giving a pod an isolated copy of the Linux networking stack. Let's start by examining what the network looks like from the pod's point of view. Find the name and location of the customer pod using the following command.

```
kubectl get pods -n yaobank -l app=customer -o wide
```
```
NAME                        READY   STATUS    RESTARTS   AGE    IP           NODE                                      NOMINATED NODE   READINESS GATES
customer-687b8d8f74-lmzqz   1/1     Running   0          129m   10.48.0.20   ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
```
Note the node on which the pod is running on (`ip-10-0-1-31.eu-west-1.compute.internal` in this example)

2. Use `kubectl` to exec into the pod so we can check the pod networking details. 

```
kubectl exec -ti -n yaobank $(kubectl get pods -n yaobank -l app=customer -o name) bash
```

3. First we will use `ip addr` to list the addresses and associated network interfaces that the pod sees.

```
ip addr
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
3: eth0@if28: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default 
    link/ether ca:56:3f:31:b7:f4 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.48.0.20/32 scope global eth0
       valid_lft forever preferred_lft forever
```

The key things to note in this output are:

* There is a `lo` loopback interface with an IP address of `127.0.0.1`. This is the standard loopback interface that every network namespace has by default. You can think of it as `localhost` for the pod itself.
* There is an `eth0` interface which has the pods actual IP address, `10.48.0.20`. Notice this matches the IP address that `kubectl get pods` returned earlier.

4. Next let's look more closely at the interfaces using `ip link`.  We will use the `-c` option, which colours the output to make it easier to read.

```
ip -c link
```
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: eth0@if28: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default 
    link/ether ca:56:3f:31:b7:f4 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

Look at the `eth0` part of the output. The key things to note are:

* The `eth0` interface is interface number 3 in the pod's network namespace.
* `eth0` is a link to the host network namespace (indicated by `link-netnsid 0`). i.e. It is the pod's side of the veth pair (virtual ethernet pair) that connects the pod to the host's networking.
* The `@if28` at the end of the interface name is the interface number of the other end of the veth pair within the hosts' network namespaces. In this example, interface number 28.  Remember this for later. We will take look at the other end of the veth pair shortly.

5. Finally, let's look at the routes the pod sees.

```
ip route
```
```
default via 169.254.1.1 dev eth0 
169.254.1.1 dev eth0  scope link
```
This shows that the pod's default route is out over the `eth0` interface. i.e. Anytime it wants to send traffic to anywhere other than itself, it will send the traffic over `eth0`.

We've finished our tour of the pod's view of the network. Let's exit out of the exec mode to return to bastion host.

```
exit
```

6. To examine the network from the node prespective, we will need to connect to the node where the customer pod is running. In our example earlier, this was `ip-10-0-1-31.eu-west-1.compute.internal`, which refers to worker2. SSH into worker2. (Please note that the your customer pod might run on a different node. SSH into the worker node where customer pod is running in your lab)

```
ssh worker2
```

7. Now we're on the node hosting the customer pod. we'll examine the other end of the veth pair. In our example output earlier, the `@if28` indicated it should be interface number 28 in the host network namespace. (Your interface numbers may be different, but you should be able to follow along the same logic.)

```
ip -c link
```

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:df:0c:da:8e:b1 brd ff:ff:ff:ff:ff:ff
    altname enp0s5
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default 
    link/ether 02:42:8f:c8:90:55 brd ff:ff:ff:ff:ff:ff
4: calia3096763ba9@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
5: cali027125f348e@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
8: cali3d74289f4a7@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
10: caliac2b08e52f5@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 3
11: calic1faebfe4cf@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 4
12: calie813bce5d54@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 5
13: caliccba8bb4a6f@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 6
16: cali5106485eace@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 7
17: cali954a1eb982a@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 8
18: calid31c0d4c940@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 9
19: cali8f727151f17@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 10
20: cali923088142d2@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 11
21: cali1457a8c4478@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 12
22: cali2fb2a259700@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 13
24: cali610b564d159@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 15
25: cali82d3343db5c@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 16
26: cali58307464414@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 14
27: calid968939d98d@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 17
28: caliaa83cb3d2b3@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 18
29: calide171474ddb@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 19
```

8. Looking at interface number 28 in this example we see `caliaa83cb3d2b3` which links to `@if3` in network namespace ID 18 (the customer pod's network namespace).  You may recall that interface 3 in the pod's network namespace was `eth0`, so this looks exactly as expected for the veth pair that connects the customer pod to the host network namespace.  
You can also see the host end of the veth pairs to other pods running on this node, all beginning with `cali`.


9. Let examine the node routing table. First let's remind ourselves of the `customer` pod's IP address.

```
kubectl get pods -n yaobank -l app=customer -o wide
````
```
NAME                        READY   STATUS    RESTARTS   AGE    IP           NODE                                      NOMINATED NODE   READINESS GATES
customer-687b8d8f74-lmzqz   1/1     Running   0          148m   10.48.0.20   ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
```

10. Now lets look at the routes on the node, where the customer pod is running. SSH into the node.

```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.31 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.31 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.31 metric 100 
10.48.0.0 dev calia3096763ba9 scope link 
blackhole 10.48.0.0/26 proto bird 
10.48.0.1 dev cali027125f348e scope link 
10.48.0.2 dev cali3d74289f4a7 scope link 
10.48.0.4 dev caliac2b08e52f5 scope link 
10.48.0.5 dev calic1faebfe4cf scope link 
10.48.0.6 dev calie813bce5d54 scope link 
10.48.0.7 dev caliccba8bb4a6f scope link 
10.48.0.8 dev cali5106485eace scope link 
10.48.0.9 dev cali954a1eb982a scope link 
10.48.0.10 dev calid31c0d4c940 scope link 
10.48.0.11 dev cali8f727151f17 scope link 
10.48.0.12 dev cali923088142d2 scope link 
10.48.0.13 dev cali1457a8c4478 scope link 
10.48.0.14 dev cali2fb2a259700 scope link 
10.48.0.16 dev cali610b564d159 scope link 
10.48.0.17 dev cali82d3343db5c scope link 
10.48.0.18 dev cali58307464414 scope link 
10.48.0.19 dev calid968939d98d scope link 
10.48.0.20 dev caliaa83cb3d2b3 scope link 
10.48.0.21 dev calide171474ddb scope link 
10.48.0.128/26 via 10.0.1.20 dev ens5 proto bird 
10.48.0.192/26 via 10.0.1.30 dev ens5 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
```

In this example output, we can see the route to the customer pod's IP `10.48.0.20` is via the `caliaa83cb3d2b3` interface, the host end of the veth pair for the customer pod. You can see similar routes for each of the IPs of the other pods hosted on this node. These routes tell Linux where to send the traffic when the traffic is destined to local pods on the node.

11. We can also see several routes labelled `proto bird`. These are routes to pods on other nodes that Calico has learned over BGP. 

To understand this better, consider this route in the example output above `10.48.0.192/26 via 10.0.1.30 dev ens5 proto bird`. It indicates pods with IP addresses falling within the `10.48.0.192/26` CIDR can be reached via `10.0.1.30` through the `ens5` network interface (the host's main interface to the rest of the network). You should see similar routes in your output for each node.

12. Calico uses route aggregation to reduce the number of routes when possible. (e.g. `/26` in this example). The `/26` corresponds to the default block size that Calico IPAM (IP Address Management) allocates on demand as nodes need pod IP addresses. (If desired, the block size can be configured in Calico IPAM settings.)  

13. You can also see the `blackhole 10.48.0.0/26 proto bird` route. The `10.48.0.0/26` corresponds to the block of IPs that Calico IPAM allocated on demand for this node. This is the block from which each of the local pods got their IP addresses. The blackhole route tells Linux that if it can't find a more specific route for an individual IP in that block then it should discard the packet (rather than sending it out through the default route to the network). You will only see traffic that hits this rule if something is trying to send traffic to a pod IP that doesn't exist, for example sending traffic to a recently deleted pod.

14. If Calico IPAM runs out of blocks to allocate to nodes, then it will use unused IPs from other nodes' blocks. These will be announced by calico as more specific routes, so traffic to pods will always find its way to the right host.

15. Exit from the worker node's ssh and get back to bastion host terminal. 

```
exit
```

_______________________________________________________________________________________________________________________________________________________________________

### Create an aditional Calico Enterprise IPPool

When a Kubernetes cluster is bootstrapped, there are two address ranges that are configured. It is very important to understand these address ranges as they can't be changed once the cluster is created.

* Pod network CIDR is the range of IP addresses Kubernetes is expecting to be assigned to the pods in the cluster.
* Service CIDR is the range of IP addresses that are used for the Cluster IPs of Kubernetes Sevices (the virtual IP that corresponds to each Kubernetes Service).

These are configured at cluster creation time (e.g. as initial kubeadm configuration).

You can find these values using the following command.

```
kubectl cluster-info dump | grep -m 2 -E "service-cluster-ip-range|cluster-cidr"
```

```
kubectl cluster-info dump | grep -m 2 -E "service-cluster-ip-range|cluster-cidr"
                            "--service-cluster-ip-range=10.49.0.0/16",
                            "--cluster-cidr=10.48.0.0/16",
```

When Calico Enterprise is deployed, a defaul IPPool is created in the cluster for each address family (IPv4-IPv6) enabled in the cluster. This cluster runs only IPv4. As a result, we will only have an IPPool for IPv4. By default, Calico creates a default IPPool for the whole cluster pod network CIDR range. However, this can be customized and a subset of pod network CIDR can be used for the default IPPool using the Installation resource at the install time. Calico Enterprise enables you to create additional IPPools post-install.\
In this section, we will:

* Create a new IPPool
* Update the yaobank deployments to receive IP addresses from the new IPPool

1. Let's start by listing the configured IPPool in this cluster and getting familiar with some of the configuration parameters in the IPPool.

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

We have extracted the relevant information here. You can see from the above output that the default IPPool range is 10.48.0.0/24, which was configured through the Installation resource in the previous lab `1. Install Calico Enterprise`.\
Note the relevant information in the manifest:

* allowedUses: specifies if this IPPool can be used for tunnel interfaces, workload interfaces, or both.
* blockSize: used by Calico IPAM to efficiently assign IPAM Blocks to node and advertise them between different nodes.
* cidr: specifies the IP range to be used for this IPPool.
* ipipMode/vxlanMode: used to enable or disable ipip and vxlan overlay. options are never, always and crosssubnet.
* natOutgoing: specifies if SNAT should happen when pods try to connect to destination outside the cluster. `natOutgoing` must be set to `true` when using overlay networking mode for external connectivity.
* nodeSelector: selects the nodes that Calico IPAM should assign addresses from this pool to.


**Note:** You can also get IPPool information using `calicotctl` instead of `kubectl` in the previous command. If you use Openshift, you can replace `kubectl` with `oc`.

In this cluster, Calico Enteprise has been configured to allocate IP addresses for pods from the `10.48.0.0/24` CIDR, which is a subset of the `10.48.0.0/16` configured through the Kubernetes APIServer.

We have the following address ranges configured in this cluster.

| CIDR         |  Purpose                                                  |
|--------------|-----------------------------------------------------------|
| 10.48.0.0/16 | Kubernetes Pod Network (via kubeadm `--pod-network-cidr`) |
| 10.48.0.0/24 | Calico - Initial default IPPool                           |
| 10.49.0.0/16 | Kubernetes Service Network (via kubeadm `--service-cidr`) |


2. Let's create a new IPPool by applying the following manifest.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: pool2-ipv4-ippool
spec:
  blockSize: 26
  cidr: 10.48.128.0/24
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
EOF

```
You should receive an output similar to the following.

```
ippool.projectcalico.org/pool2-ipv4-ippool created
```

3. Check the IPPools that exist in the cluster.

```
calicoctl get ippools
```
```
NAME                  CIDR             SELECTOR   
default-ipv4-ippool   10.48.0.0/24     all()      
pool2-ipv4-ippool     10.48.128.0/24   all()
```


4. There is a new version of the yaobank application that is configured for specific IP address treatment. We have configured the yaobank manifest with the necessary annotation to use the newly created IPPool. Note that annotations are configured only for summary and database deployment. Customer deployment can receive its IP address from any IPPool. Examine the annotation section of the deployments below and get yourself familiar with the configurations. 

5. Before deploying the new version of yaobank application, let's delete the old version to avoid any conflicts. 

```
kubectl delete namespace yaobank
```
You should receive an output similar to the following. This command might take about 1-2 minutes to complete. Please wait util the namespace and all of the resources assocaited within it are deleted.

```
namespace "yaobank" deleted
```
6. Now implement the new version of the app.

```
kubectl apply -f -<<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: yaobank
  labels:
    istio-injection: disabled

---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: yaobank
  labels:
    app: database
spec:
  ports:
  - port: 2379
    name: http
  selector:
    app: database

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: database
  namespace: yaobank
  labels:
    app: yaobank

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: yaobank
spec:
  selector:
    matchLabels:
      app: database
      version: v1
  replicas: 1
  template:
    metadata:
      labels:
        app: database
        version: v1
      annotations:
        "cni.projectcalico.org/ipv4pools": "[\"pool2-ipv4-ippool\"]"
    spec:
      serviceAccountName: database
      containers:
      - name: database
        image: calico/yaobank-database:certification
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 2379
        command: ["etcd"]
        args:
          - "-advertise-client-urls"
          - "http://database:2379"
          - "-listen-client-urls"
          - "http://0.0.0.0:2379"

---
apiVersion: v1
kind: Service
metadata:
  name: summary
  namespace: yaobank
  labels:
    app: summary
spec:
  ports:
  - port: 80
    name: http
  selector:
    app: summary
    
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: summary
  namespace: yaobank
  labels:
    app: yaobank
    database: reader
    
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: summary
  namespace: yaobank
spec:
  replicas: 2
  selector:
    matchLabels:
      app: summary
      version: v1
  template:
    metadata:
      labels:
        app: summary
        version: v1
      annotations:
        "cni.projectcalico.org/ipv4pools": "[\"pool2-ipv4-ippool\"]"
    spec:
      serviceAccountName: summary
      containers:
      - name: summary
        image: calico/yaobank-summary:certification
        imagePullPolicy: Always
        ports:
        - containerPort: 80
 
---
apiVersion: v1
kind: Service
metadata:
  name: customer
  namespace: yaobank
  labels:
    app: customer
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30180
    name: http
  selector:
    app: customer
    
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: customer
  namespace: yaobank
  labels:
    app: yaobank
    summary: reader
    
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: customer
  namespace: yaobank
spec:
  replicas: 1
  selector:
    matchLabels:
      app: customer
      version: v1
  template:
    metadata:
      labels:
        app: customer
        version: v1
    spec:
      serviceAccountName: customer
      containers:
      - name: customer
        image: calico/yaobank-customer:certification
        imagePullPolicy: Always
        ports:
        - containerPort: 80
---
EOF

```

You should receive an output similar to the following.

```
namespace/yaobank created
service/database created
serviceaccount/database created
deployment.apps/database created
service/summary created
serviceaccount/summary created
deployment.apps/summary created
service/customer created
serviceaccount/customer created
deployment.apps/customer created
```

7. Let's check on the Pods' ip address assignments.

```
kubectl get pod -n yaobank -o wide
```

```
NAME                        READY   STATUS    RESTARTS   AGE   IP              NODE                                      NOMINATED NODE   READINESS GATES
customer-687b8d8f74-tcclp   1/1     Running   0          16s   10.48.0.43      ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
database-75ccfdc84f-hqr64   1/1     Running   0          16s   10.48.128.0     ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
summary-7d78c9976b-sxv4k    1/1     Running   0          16s   10.48.128.192   ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
summary-7d78c9976b-wjdd5    1/1     Running   0          16s   10.48.128.1     ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>

```

You can see that Pod IP address assignment is aligned with the IPAM configurations defined in manifest above, assigning pods of a deployment to the correct IPPool. Calico IPAM provides the flexibility of assigning IPPools to namespaces, deployment, deamonset, etc. You can also implement topology based IP address assignment in which racks or nodes in specific racks receive their IP addresses from one or more specific IPPools. For more information, visit the following link.

https://projectcalico.docs.tigera.io/networking/assign-ip-addresses-topology

8. To check the IPAM allocations in the cluster, run the following command.

```
calicoctl ipam show

```

```
+----------+----------------+-----------+------------+-----------+
| GROUPING |      CIDR      | IPS TOTAL | IPS IN USE | IPS FREE  |
+----------+----------------+-----------+------------+-----------+
| IP Pool  | 10.48.0.0/24   |       256 | 36 (14%)   | 220 (86%) |
| IP Pool  | 10.48.128.0/24 |       256 | 3 (1%)     | 253 (99%) |
+----------+----------------+-----------+------------+-----------+
```

9. Use `calicoctl` help to learn about subcommands available to `calicoctl ipam`. Please note that Tigera APIServer enables you to use Kubernetes native command line utility `kubectl` to manage most of the aspects of Calico Enterprise (CE) resources. However, there are some aspects of CE that can't be managed by `kubectl` and `calicoctl` needs to be used. Tere is no equivalent capability through `kubectl` to find the information that is provided below through `calicoctl ipam`.

```
calicoctl ipam -h

```
```
Set the Calico datastore access information in the environment variables or
supply details in a config file.

Usage:
  calicoctl ipam <command> [<args>...]

    check            Check the integrity of the IPAM datastructures.
    release          Release a Calico assigned IP address.
    show             Show details of a Calico configuration,
                     assigned IP address, or of overall IP usage.
    split            Split the IP pool specified by the CIDR into
                     the specified number of smaller IPPools.
    configure        Configure IPAM

Options:
  -h --help      Show this screen.

Description:
  IP address management commands for Calico.

  See 'calicoctl ipam <command> --help' to read about a specific subcommand.
```

> **Congratulations! You have completed `2. Implement pod networking using Calico Enterprise CNI and IPAM` lab.**
