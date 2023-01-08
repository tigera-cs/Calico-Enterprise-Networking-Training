# In this lab

This lab provides the instructions to:

* [Overview](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/blob/main/6.%20Implement%20Calico%20Enterprise%20eBPF/README.md#overview)
* [Enable Calico Enterprise eBPF](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/blob/main/6.%20Implement%20Calico%20Enterprise%20eBPF/README.md#enable-calico-enterprise-ebpf)


### Overview

Calico Enterprise has got a pluggable dataplane, which enables Calico Enterprise to easily implement and adopt various dataplanes. Currently Calico Enterprise  supports four different dataplanes. Standard Linux iptables dataplane, eBPF dataplane, Windows dataplane, and VPP dataplane. eBPF is an alternative dataplane for iptables in Linux. Instead of using Linux standard dataplane, which uses iptables and conntrack, the dataplane uses a set of bpf programs.
eBPF has lower overhead and better performance compared to iptables at large scales.

After finshing this lab, you learn how to enable Calico Enterprise eBPF along with some troubleshooting guidance.

______________________________________________________________________________________________________________________________________________________________________

### Enable Calico Enterprise eBPF

1. Calico Enterprise eBPF requires specific Linux kernel versions based on the Linux operation system in use in the cluster. SSH into all the cluster nodes and ensure they use supported kernel. For more information about the supported Linux kernel versions, check on the Calico Enterprise eBPF documentation provide below.

https://docs.tigera.io/maintenance/ebpf/

Run the following commands on each cluster node.

```
uname -rv
```

You should receive a similar output.

```
5.15.0-1026-aws #30~20.04.2-Ubuntu SMP Fri Nov 25 14:53:22 UTC 2022
```

Verify that the BPF filesystem is mounted by running the following command.

```
mount | grep "/sys/fs/bpf"
```

You should receive a similar output.

```
bpf on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700)
```

2. In eBPF mode, Calico Enterprise implements Kubernetes service networking directly rather than relying on kube-proxy. This means that, like kube-proxy, Calico Enterprise needs to directly connect to the Kubernetes APIServer rather than via the APIServer’s ClusterIP. Refer to the following documentation for more information on this.

https://docs.tigera.io/maintenance/ebpf/enabling-ebpf#configure-calico-enterprise-to-talk-directly-to-the-api-server

3. First, make a note of the address of the APIServer. Run the following command to find the APIServer IP address.

```
kubectl get endpoints kubernetes -o wide

```

You should receive a similar output with a single IP address and port number.

```
NAME         ENDPOINTS        AGE
kubernetes   10.0.1.20:6443   6d1h
```
 
4. Then, create the following config map in the tigera-operator namespace using the host and port number determined above.

```
kubectl apply -f -<<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: kubernetes-services-endpoint
  namespace: tigera-operator
data:
  KUBERNETES_SERVICE_HOST: "10.0.1.20"
  KUBERNETES_SERVICE_PORT: "6443"
EOF

```

5. It can take up to 60s for kubelet to pick up the ConfigMap. tigera operator will then do a rolling update of the relevant pods in the calico-system namespace to pass on the change. calico-node, calico-typha, and calico-kubecontroller pods should be restarted. Run the following command to validate. Make sure all the pods are back into fully Running state after the restart.
 
```
watch kubectl get pods -n calico-system

```

**Note:** If you do not see the pods restart then it’s possible that the ConfigMap was not picked up (sometimes Kubernetes is slow to propagate ConfigMap ). Try restarting the tigera-operator.

6. In eBPF mode Calico Enterprise replaces kube-proxy and there is no need to run kube-proxy in the cluster. Run the following command to disable kube-proxy.

```
kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'

``` 
 
7. To enable eBPF mode, change the spec.calicoNetwork.linuxDataplane parameter in the operator’s Installation resource to "BPF". You also need to remove the hostPorts setting because hostPorts are not supported in BPF mode.

```
kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"BPF", "hostPorts":null}}}'

```` 

8. Direct Server Return (DSR) mode is disabled by default. To enable it, set the BPFExternalServiceMode Felix configuration parameter to "DSR". Run the following command to enable DSR. For more information on DSR, visit the following link.

https://docs.tigera.io/maintenance/ebpf/enabling-ebpf#try-out-dsr-mode

```
kubectl patch felixconfigurations default --patch='{"spec": {"bpfExternalServiceMode": "DSR"}}'

```

9. To switch back from DSR to tunneled mode, set the configuration parameter to "Tunnel".

```
kubectl patch felixconfigurations default --patch='{"spec": {"bpfExternalServiceMode": "Tunnel"}}'

```
    
10. Check calico-node pods logs to verify eBPF is enabled. 

```
for i in $(kubectl get pods -n calico-system -o name | grep calico-node) ; do kubectl logs $i -n calico-system | grep -w "BPF enabled" ; done

```

You should see an output similar to the following.

```
2023-01-08 05:28:36.405 [INFO][56302] felix/int_dataplane.go 542: BPF enabled, configuring iptables layer to clean up kube-proxy's rules.
2023-01-08 05:28:36.696 [INFO][56302] felix/int_dataplane.go 973: BPF enabled, starting BPF endpoint manager and map manager.
2023-01-08 05:28:36.814 [INFO][56302] felix/int_dataplane.go 2544: BPF enabled, disabling unprivileged BPF usage.
2023-01-08 05:28:52.520 [INFO][57188] felix/int_dataplane.go 542: BPF enabled, configuring iptables layer to clean up kube-proxy's rules.
2023-01-08 05:28:52.648 [INFO][57188] felix/int_dataplane.go 973: BPF enabled, starting BPF endpoint manager and map manager.
2023-01-08 05:28:52.707 [INFO][57188] felix/int_dataplane.go 2544: BPF enabled, disabling unprivileged BPF usage.
2023-01-08 05:29:44.193 [INFO][54837] felix/int_dataplane.go 542: BPF enabled, configuring iptables layer to clean up kube-proxy's rules.
2023-01-08 05:29:44.747 [INFO][54837] felix/int_dataplane.go 973: BPF enabled, starting BPF endpoint manager and map manager.
2023-01-08 05:29:44.966 [INFO][54837] felix/int_dataplane.go 2544: BPF enabled, disabling unprivileged BPF usage.
2023-01-08 05:29:18.138 [INFO][56524] felix/int_dataplane.go 542: BPF enabled, configuring iptables layer to clean up kube-proxy's rules.
2023-01-08 05:29:18.364 [INFO][56524] felix/int_dataplane.go 973: BPF enabled, starting BPF endpoint manager and map manager.
2023-01-08 05:29:18.532 [INFO][56524] felix/int_dataplane.go 2544: BPF enabled, disabling unprivileged BPF usage.
```
-----

11. Let's validate if eBPF is enforced. The validation can be done directly from the cluster node by SSHing into the node or via the calico-node pod. Let's do the validation via `control1` node and the `calico-node` pod running on `control1`

**Validation from cluster node**

ssh into `control1` node.

```
ssh control1

```
Run the following command.

```
tc -s qdisc show | grep clsact -A 2

```
You should see an output similar to the following.

```
qdisc clsact ffff: dev ens5 parent ffff:fff1 
 Sent 63121309 bytes 136851 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
qdisc clsact ffff: dev cali4b8da7b4d32 parent ffff:fff1 
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
qdisc clsact ffff: dev calia8fdb5c22c7 parent ffff:fff1 
 Sent 6021246 bytes 4974 pkt (dropped 4, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
qdisc clsact ffff: dev cali28487a3174b parent ffff:fff1 
 Sent 241539 bytes 1760 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
qdisc clsact ffff: dev cali511b65d1b41 parent ffff:fff1 
 Sent 86681 bytes 521 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
qdisc clsact ffff: dev cali42e0d8fec43 parent ffff:fff1 
 Sent 1170139 bytes 5487 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
qdisc clsact ffff: dev cali78bac677beb parent ffff:fff1 
 Sent 620754 bytes 1656 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
qdisc clsact ffff: dev cali1c0cad31d33 parent ffff:fff1 
 Sent 7570673 bytes 19820 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
qdisc clsact ffff: dev cali0c3fb9ffe63 parent ffff:fff1 
 Sent 651039 bytes 1702 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
```

 **Validation from calico-node**

Run the following command to do the validation from calico-node running on `control1`

```
CALICO_NODE=$(kubectl get pods -n calico-system -l k8s-app=calico-node -o wide | grep 10.0.1.20 | awk {'print $1'}) && echo $CALICO_NODE
kubectl exec -ti $CALICO_NODE -n calico-system -- tc -s qdisc show | grep clsact -A 2

```
You should see an output similar to the following.

```
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), mount-bpffs (init), install-cni (init)
qdisc clsact ffff: dev ens5 parent ffff:fff1 
 Sent 133366690 bytes 304342 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
  qdisc clsact ffff: dev cali4b8da7b4d32 parent ffff:fff1 
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
  qdisc clsact ffff: dev calia8fdb5c22c7 parent ffff:fff1 
 Sent 11878043 bytes 10657 pkt (dropped 4, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
  qdisc clsact ffff: dev cali28487a3174b parent ffff:fff1 
 Sent 596152 bytes 4286 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
  qdisc clsact ffff: dev cali511b65d1b41 parent ffff:fff1 
 Sent 221125 bytes 1296 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
  qdisc clsact ffff: dev cali42e0d8fec43 parent ffff:fff1 
 Sent 2806182 bytes 13258 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
  qdisc clsact ffff: dev cali78bac677beb parent ffff:fff1 
 Sent 1390238 bytes 3733 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
  qdisc clsact ffff: dev cali1c0cad31d33 parent ffff:fff1 
 Sent 11984945 bytes 33805 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
--
  qdisc clsact ffff: dev cali0c3fb9ffe63 parent ffff:fff1 
 Sent 1416612 bytes 4086 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
```

12. Calico Enterprise provides bpftool to interact with Calico Enterpris bpf and collect relevant information. bpftool is available from calico-node pods. Let's use bpftool to view the configurations implemented by bpf. We will run these commands from calico-node pod runnin on `control1`.

```
CALICO_NODE=$(kubectl get pods -n calico-system -l k8s-app=calico-node -o wide | grep 10.0.1.20 | awk {'print $1'}) && echo $CALICO_NODE

```

```
kubectl exec -ti $CALICO_NODE -n calico-system -- calico-node -bpf help

```
You should see an output similar to the following.

```
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), mount-bpffs (init), install-cni (init)
INFO[0000] FIPS mode enabled=false                       source="main.go:137"
tool for interrogating Calico BPF state

Usage:
  calico-bpf [command]

Available Commands:
  arp          Manipulates arp
  completion   Generate the autocompletion script for the specified shell
  connect-time Manipulates connect-time load balancing programs
  conntrack    Manipulates connection tracking
  counters     Show and reset counters
  help         Help about any command
  ifstate      Manipulates ifstate
  ipsets       Manipulates ipsets
  nat          Manipulates network address translation (nat)
  policy       Dump policy attached to interface
  routes       Manipulates routes
  version      Prints the version and exits

Flags:
      --config string      config file (default is $HOME/.calico-bpf.yaml)
  -h, --help               help for calico-bpf
      --log-level string   Set log level (default "warn")
  -t, --toggle             Help message for toggle

Use "calico-bpf [command] --help" for more information about a command.
```


# Understanding routes in eBPF: #

BPF program route packets before reaching routing Linux tables however it BPF is relaying on Linux routing table:

**Linux routing table**

    ubuntu@ip-10-0-1-20:~$ip route show
    default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.20 metric 100
    10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.20
    10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.20 metric 100
    10.48.7.64/26 via 10.0.1.30 dev tunl0 proto bird onlink
    blackhole 10.48.32.192/26 proto bird
    10.48.32.195 dev cali2896b5b2df4 scope link
    10.48.32.196 dev calif903a7f3bc2 scope link
    10.48.32.197 dev calia7a92b8d91c scope link
    10.48.32.198 dev cali086ab313818 scope link
    10.48.32.199 dev cali2e939172c25 scope link
    10.48.32.200 dev caliac215715fe6 scope link
    10.48.32.201 dev cali490acbd79f7 scope link
    10.48.162.128/26 via 10.0.1.31 dev tunl0 proto bird onlink
    172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
 



**BPF routing table**

    ubuntu@ip-10-0-1-20:~$ kubectl exec -n calico-system calico-node-84r2t -- calico-node -bpf routes dump
    2021-05-10 10:59:28.353 [INFO][2445] confd/maps.go 313: Loaded map file descriptor. fd=0x7 name="/sys/fs/bpf/tc/globals/cali_v4_routes"
       10.0.1.20/32: local host
       10.0.1.30/32: remote host
       10.0.1.31/32: remote host
       10.48.0.0/16: remote in-pool nat-out
      10.48.7.64/26: remote workload in-pool nat-out nh 10.0.1.30
    10.48.32.192/32: local host
    10.48.32.195/32: local workload in-pool nat-out idx 11
    10.48.32.196/32: local workload in-pool nat-out idx 12
    10.48.32.197/32: local workload in-pool nat-out idx 13
    10.48.32.198/32: local workload in-pool nat-out idx 14
    10.48.32.199/32: local workload in-pool nat-out idx 15
    10.48.32.200/32: local workload in-pool nat-out idx 16
    10.48.32.201/32: local workload in-pool nat-out idx 19
    10.48.162.128/26: remote workload in-pool nat-out nh 10.0.1.31
      172.17.0.1/32: local host



- `10.0.1.20/32: local host` = ip is a node ip
- `10.0.1.30/32: remote host` = ip is a remote node ip
- `10.48.162.128/26: remote workload in-pool nat-out nh 10.0.1.31` = remote pod network on node -> 10.0.1.31
- `10.255.241.69/32: local workload in-pool nat-out idx 77` = local pod 



# BPF nat rules: #

    ubuntu@ip-10-0-1-20:~$ kubectl get svc -n yaobank
    NAME       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)  AGE
    customer   ClusterIP   10.49.82.5    <none>        80/TCP   140m
    database   ClusterIP   10.49.10.100  <none>        2379/TCP 140m
    summary    ClusterIP   10.49.154.85  <none>        80/TCP   140m
    ubuntu@ip-10-0-1-20:~$ kubectl get svc -n tigera-manager
    NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
    tigera-manager           ClusterIP   10.49.168.197   <none>        9443/TCP         2d20h
    tigera-manager-external  NodePort    10.49.205.93    <none>        9443:30003/TCP   2d20h
    


    ubuntu@ip-10-0-1-20:~$ kubectl exec -n calico-system calico-node-84r2t -- calico-node -bpf nat dump
    2021-05-10 11:05:45.649 [INFO][3708] confd/maps.go 313: Loaded map file descriptor. fd=0x7 name="/sys/fs/bpf/tc/globals/cali_v4_nat_fe2"
    2021-05-10 11:05:45.650 [INFO][3708] confd/maps.go 313: Loaded map file descriptor. fd=0x8 name="/sys/fs/bpf/tc/globals/cali_v4_nat_be"
    10.49.10.100 port 2379 proto 6 id 21 count 1 local 0
    	21:0 10.48.7.86:2379
    ...
    10.49.205.93 port 9443 proto 6 id 12 count 1 local 1
    	12:0 10.48.32.200:9443
    ...
    10.49.82.5 port 80 proto 6 id 18 count 1 local 0
    	18:0 10.48.162.144:8000
    10.49.168.197 port 9443 proto 6 id 14 count 1 local 1
    	14:0 10.48.32.200:9443
    ...
    10.0.1.20 port 30003 proto 6 id 12 count 1 local 1
    	12:0 10.48.32.200:9443
    ...
    10.49.154.85 port 80 proto 6 id 4 count 1 local 0
    	4:0  10.48.7.87:8000
    ...

***Each field explained:***

    [service ip] [service port] [protocol number] [id] [count] [local]
    [id:countid] [pod ip:port]

> [service ip] - kubernetes clusterIP/externalIp/nodeIP
> 
> [service port] -  kubernetes - Service port
> 
> [protocol number] - Protocol number used 
> 
> [id] - identification number (unique)
> 
> [count] - number of pods on this service
> 
> [local] - how many pods are local on current node
> 
> [id:countid] - “identification number”:”each pod ip have assigned id”
> 
> [pod ip:port]


# BPF ipsests: #

    ubuntu@ip-10-0-1-20:~$ kubectl exec -n calico-system calico-node-84r2t -- calico-node -bpf ipsets dump
    2021-05-10 11:10:12.086 [INFO][4611] confd/maps.go 313: Loaded map file descriptor. fd=0x7 name="/sys/fs/bpf/tc/globals/cali_v4_ip_sets"
    IP set 0x100000001
       10.49.0.10:53 (proto 17)
    
    IP set 0xa48e493196cddbb
       10.0.1.20/32
    
    IP set 0x1451213aebb2fe8a
       10.48.7.69/32
    
    IP set 0x1e69a881cefa300e
       10.48.162.140/32
    


# BPF connection tracking: #

    ubuntu@ip-10-0-1-20:~$ kubectl exec -n calico-system calico-node-84r2t -- calico-node -bpf conntrack dump | less
    ConntrackKey{proto=6 10.0.1.20:6443 <-> 10.0.100.241:35510} -> Entry{Type:0, Created:253843922820121, LastSeen:253844042814512, Flags: <none> Data: {A2B:{Bytes:140 Packets:2 Seqno:2871352450 SynSeen:true AckSeen:true FinSeen:true RstSeen:false Whitelisted:false Opener:false Ifindex:0} B2A:{Bytes:272 Packets:4 Seqno:2473514777 SynSeen:true AckSeen:true FinSeen:true RstSeen:false Whitelisted:true Opener:true Ifindex:2} OrigDst:0.0.0.0 OrigPort:0 TunIP:0.0.0.0}} Age: 1.178198172s Active ago 1.058203781s CLOSED
    
    ConntrackKey{proto=6 10.48.32.200:9443 <-> 10.0.100.241:32628} -> Entry{Type:2, Created:253829881145823, LastSeen:253829910183842, Flags: <none> Data: {A2B:{Bytes:140 Packets:2 Seqno:2722366807 SynSeen:true AckSeen:true FinSeen:true RstSeen:false Whitelisted:true Opener:false Ifindex:0} B2A:{Bytes:272 Packets:4 Seqno:3020788598 SynSeen:true AckSeen:true FinSeen:true RstSeen:false Whitelisted:true Opener:true Ifindex:2} OrigDst:10.0.1.20 OrigPort:30003 TunIP:0.0.0.0}} Age: 15.220220233s Active ago 15.191182214s CLOSED
    
    ConntrackKey{proto=17 10.48.7.66:53 <-> 10.48.32.195:46851} -> Entry{Type:0, Created:253796864945916, LastSeen:253796865695820, Flags:16 Data: {A2B:{Bytes:504 Packets:2 Seqno:0 SynSeen:false AckSeen:false FinSeen:false RstSeen:false Whitelisted:true Opener:false Ifindex:0} B2A:{Bytes:278 Packets:2 Seqno:0 SynSeen:false AckSeen:false FinSeen:false RstSeen:false Whitelisted:true Opener:true Ifindex:11} OrigDst:0.0.0.0 OrigPort:0 TunIP:0.0.0.0}} Age: 48.237002075s Active ago 48.236252171s

    

# Change BPF in debug mode #

1. Ensure `calicoctl` is installed. Follow the steps from previous labs.
 
2. Set Felix `bpfLogLevel` to Debug (Info not so useful at this stage)

	Enables a firehose of logs to the BPF trace pipe

3. To tail the log: `sudo tc exec bpf debug`

	Every decision point for every packet is logged!

- Get felixconfigruation:
-

    ubuntu@ip-10-0-1-20:~$ calicoctl get felixconfigurations default -o yaml
    apiVersion: projectcalico.org/v3
    kind: FelixConfiguration
    metadata:
      creationTimestamp: "2021-03-11T18:52:26Z"
      name: default
      resourceVersion: "11336796"
      uid: 0e57caaa-6a04-48cb-8bb6-3c4e4ca44c4a
    spec:
      bpfEnabled: true
      bpfExternalServiceMode: DSR
      bpfKubeProxyIptablesCleanupEnabled: false
      flowLogsCollectProcessInfo: true
      logSeverityScreen: Info
      prometheusMetricsEnabled: true
      reportingInterval: 0s


- Patch felixconfiguration:
-

    ubuntu@ip-10-0-1-20:~$ calicoctl patch felixconfiguration default --patch='{"spec": {"bpfLogLevel": "Debug"}}'

    Successfully patched 1 'FelixConfiguration' resource

- Check felixconfiguration to ensure the BPF is in debug mode 
-

    ubuntu@ip-10-0-1-20:~$ calicoctl get felixconfigurations default -o yaml
    apiVersion: projectcalico.org/v3
    kind: FelixConfiguration
    metadata:
      creationTimestamp: "2021-03-11T18:52:26Z"
      name: default
      resourceVersion: "11336796"
      uid: 0e57caaa-6a04-48cb-8bb6-3c4e4ca44c4a
    spec:
      bpfEnabled: true
      bpfExternalServiceMode: DSR
      bpfKubeProxyIptablesCleanupEnabled: false
      bpfLogLevel: Debug
      flowLogsCollectProcessInfo: true
      logSeverityScreen: Info
      prometheusMetricsEnabled: true
      reportingInterval: 0s

Run tc in debug mode: `ubuntu@ip-10-0-1-20:~$sudo tc exec bpf debug`

`<...>-84582 [000] .Ns1  6851.690474: 0: ens192---E: Final result=ALLOW (-1). Program execution time: 7366ns`                                                     

> [84582] - PID that triggered NAPI poll (often useful)
> 
> [000] - CPU number
> 
>[6851.690474] - Timestamp
>
>[0:] - Calico program ID then 
>[ens192] - Interface caliXXX, ethXX
>[---E:] - E=egress, I=ingress, C=connect-time
> 
>[Final result=ALLOW (-1). Program execution time: 7366ns] - Message
> 


# Reversing to iptables datapath #

To revert to standard Linux networking:

Reverse the changes to the operator’s Installation:

    tigera@bastion:~$ kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"Iptables"}}}'
	or
	ubuntu@ip-10-0-1-20:~$ kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"linuxDataplane":"Iptables"}}}'
If you disabled kube-proxy, re-enable it (for example, by removing the node selector added above).

    tigera@bastion:~$ kubectl patch ds -n kube-system kube-proxy --type merge -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": null}}}}}'
	or
	ubuntu@ip-10-0-1-20 :~$ kubectl patch ds -n kube-system kube-proxy --type merge -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": null}}}}}'
Monitor existing workloads to make sure they re-establish any connections disrupted by the switch.

After you see the kube-poxy pods are back please check the following controller node.

    ubuntu@ip-10-0-1-20:~$ kubectl get pods -n kube-system -o wide | grep proxy
    kube-proxy-w8hl6   1/1 Running   0  4m5s	10.0.1.30   ip-10-0-1-30   <none>   <none>
    kube-proxy-zgv8f   1/1 Running   0  4m5s	10.0.1.31   ip-10-0-1-31   <none>   <none>
    kube-proxy-zj8qn   1/1 Running   0  4m5s    10.0.1.20   ip-10-0-1-20   <none>   <none>

    ubuntu@ip-10-0-1-20:~$ tc -s qdisc show | grep clsact -A 2
    ubuntu@ip-10-0-1-20:~$
