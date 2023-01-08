# In this lab

This lab provides the instructions to:

* [Overview](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/blob/main/6.%20Implement%20Calico%20Enterprise%20eBPF/README.md#overview)
* [Enable Calico Enterprise eBPF](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/blob/main/6.%20Implement%20Calico%20Enterprise%20eBPF/README.md#enable-calico-enterprise-ebpf)
* [Troubleshoot Calico Enterprise eBPF](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/blob/main/6.%20Implement%20Calico%20Enterprise%20eBPF/README.md#troubleshoot-calico-enterprise-ebpf)
* [ Revert back to iptables dataplane](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/blob/main/6.%20Implement%20Calico%20Enterprise%20eBPF/README.md#revert-back-to-iptables-dataplane)


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

12. Calico Enterprise provides bpftool to interact with Calico Enterpris eBPF and collect relevant information. bpftool is available from calico-node pods. Let's use bpftool to view the configurations implemented by eBPF. We will run these commands from calico-node pod runnin on `control1`.

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


13. Calico Enterprise eBPF routes packets before reaching Linux routing tables. However, Calico Enterprise eBPF always uses Linux routing table to route the first packet and take advantage of Linux kernel security features, RPF, etc. Calico Enterprise eBPF records this information as a conntrack entry in the conntrack map and skip all the processing for the subsequent flows.

14. SSH into `control1` and show the routing table.

```
ssh control1

```
```
ip route show

```

You should see an output similar to the following.

```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.20 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.20 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.20 metric 100 
10.48.0.0/26 via 10.0.1.31 dev ens5 proto bird 
10.48.0.128 dev cali4b8da7b4d32 scope link 
blackhole 10.48.0.128/26 proto bird 
10.48.0.132 dev calia8fdb5c22c7 scope link 
10.48.0.133 dev cali28487a3174b scope link 
10.48.0.134 dev cali511b65d1b41 scope link 
10.48.0.135 dev cali42e0d8fec43 scope link 
10.48.0.136 dev cali78bac677beb scope link 
10.48.0.137 dev cali1c0cad31d33 scope link 
10.48.0.138 dev cali0c3fb9ffe63 scope link 
10.48.0.192/26 via 10.0.1.30 dev ens5 proto bird 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
```

15. Exit the ssh session.

```
exit

```
16. Now, let's check the eBPF routing table on calico-node running on `control1`.

```
kubectl exec -ti $CALICO_NODE -n calico-system -- calico-node -bpf routes dump

```

You should see an output similar to the following.

```
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), mount-bpffs (init), install-cni (init)
INFO[0000] FIPS mode enabled=false                       source="main.go:137"
   10.0.1.20/32: local host
   10.0.1.30/32: remote host
   10.0.1.31/32: remote host
   10.48.0.0/24: remote in-pool nat-out
   10.48.0.0/26: remote workload in-pool nat-out nh 10.0.1.31
 10.48.0.128/32: local workload in-pool nat-out idx 4
 10.48.0.132/32: local workload in-pool nat-out idx 12
 10.48.0.133/32: local workload in-pool nat-out idx 13
 10.48.0.134/32: local workload in-pool nat-out idx 14
 10.48.0.135/32: local workload in-pool nat-out idx 15
 10.48.0.136/32: local workload in-pool nat-out idx 18
 10.48.0.137/32: local workload in-pool nat-out idx 19
 10.48.0.138/32: local workload in-pool nat-out idx 20
 10.48.0.192/26: remote workload in-pool nat-out nh 10.0.1.30
  172.17.0.1/32: local host
```

17. Note the following about the above output.

* `10.0.1.20/32: local host` = ip is the local node ip
* `10.0.1.30/32: remote host` = ip is a remote node ip
* `10.48.162.128/26: remote workload in-pool nat-out nh 10.0.1.31` = remote pod network on node -> 10.0.1.31
* `10.255.241.69/32: local workload in-pool nat-out idx 77` = local pod 


18. As discussed before, in eBPF mode, Calico Enterprise implements Kubernetes service networking directly rather than relying on kube-proxy. Let's check the service implementation details through eBPF.

19. Run the following commands to list yaobank `summary` service and endpoints. This service has two endpoints. We will see how Calico Enterprise eBPF implements this service next.

```
kubectl get svc -n yaobank summary

```

```
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
summary   ClusterIP   10.49.18.111   <none>        80/TCP    6m4s
```

```
kubectl get endpoints -n yaobank summary

```
```
NAME      ENDPOINTS                       AGE
summary   10.48.0.210:80,10.48.0.212:80   9m58s
```

20. Run the following command to list the `summary` service implemented through eBPF. Make sure to replace service IP `10.49.18.111` with the service IP in your lab instance.

```
kubectl exec -ti $CALICO_NODE -n calico-system -- calico-node -bpf nat dump | grep -A2 10.49.18.111

```

You should see an output similar to the following. 

```
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), mount-bpffs (init), install-cni (init)
10.49.18.111 port 80 proto 6 id 23 count 2 local 0
        23:0     10.48.0.210:80
        23:1     10.48.0.212:80
```

21. Find below the explanation of each field in the previous output.

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


22. `ipsets` are key in implementing security policies. To list the `ipsets` in use on a node, run the following command.

```
kubectl exec -ti $CALICO_NODE -n calico-system -- calico-node -bpf ipsets dump

```
You should see an output similar to the following. However, the list can be different depending on the available ipsets on the node.

```
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), mount-bpffs (init), install-cni (init)
INFO[0000] FIPS mode enabled=false                       source="main.go:137"
IP set 0x100000001
   10.49.0.10:53 (proto 17)

IP set 0xd26fa31f8d497f4
   10.48.0.135/32
   10.48.0.208/32

IP set 0x3daf74135d5cb3e9
   10.0.1.20/32

IP set 0x6b040aa4ed99b698
   10.48.0.194/32

IP set 0x6e77fb5b61e41fed
   10.48.0.2/32
   10.48.0.5/32

IP set 0x7edc0e52b1c7a2ac
   10.48.0.11/32
   10.48.0.199/32

IP set 0x9a85ddf3b2bb9365
   10.0.1.20:6443 (proto 6)

IP set 0xa94ca2b47cd0c227
   10.48.0.197/32

IP set 0xae04d0ae61bb92ee
   10.48.0.137:5443 (proto 6)
   10.48.0.137:8080 (proto 6)
   10.48.0.14:5443 (proto 6)
   10.48.0.14:8080 (proto 6)

IP set 0xca2fa4f94dc261c4
   10.48.0.201/32

IP set 0xdfdd106f994fdb54
   10.48.0.197/32
```

23. To list the conntrack map, run the following command.

```
kubectl exec -ti $CALICO_NODE -n calico-system -- calico-node -bpf conntrack dump

```
You should see an output similar to the following. The following output is truncated.

```
Defaulted container "calico-node" out of: calico-node, flexvol-driver (init), mount-bpffs (init), install-cni (init)
INFO[0000] FIPS mode enabled=false                       source="main.go:137"
ConntrackKey{proto=17 10.48.0.2:53 <-> 10.48.0.135:36450} -> Entry{Type:0, Created:13280597154143, LastSeen:13280597444227, Flags: 0x110 B-A Data: {A2B:{Bytes:225 Packets:1 Seqno:0 SynSeen:false AckSeen:false FinSeen:false RstSeen:false Whitelisted:true Opener:false Ifindex:2} B2A:{Bytes:132 Packets:1 Seqno:0 SynSeen:false AckSeen:false FinSeen:false RstSeen:false Whitelisted:true Opener:true Ifindex:15} OrigDst:0.0.0.0 OrigSrc:0.0.0.0 OrigPort:0 OrigSPort:0 TunIP:0.0.0.0}} Age: 24.318178817s Active ago 24.317888733s
ConntrackKey{proto=6 10.0.1.20:56738 <-> 169.254.169.254:80} -> Entry{Type:0, Created:13297805187841, LastSeen:13297805741251, Flags: <none> Data: {A2B:{Bytes:616 Packets:5 Seqno:1044308989 SynSeen:true AckSeen:true FinSeen:true RstSeen:false Whitelisted:false Opener:true Ifindex:0} B2A:{Bytes:585 Packets:5 Seqno:2967668612 SynSeen:true AckSeen:true FinSeen:true RstSeen:false Whitelisted:true Opener:false Ifindex:2} OrigDst:0.0.0.0 OrigSrc:0.0.0.0 OrigPort:0 OrigSPort:0 TunIP:0.0.0.0}} Age: 7.110195217s Active ago 7.109641807s CLOSED
ConntrackKey{proto=17 10.48.0.5:53 <-> 10.48.0.132:13963} -> Entry{Type:0, Created:13256085471314, LastSeen:13256086882425, Flags: 0x110 B-A Data: {A2B:{Bytes:677 Packets:3 Seqno:0 SynSeen:false AckSeen:false FinSeen:false RstSeen:false Whitelisted:true Opener:false Ifindex:2} B2A:{Bytes:407 Packets:3 Seqno:0 SynSeen:false AckSeen:false FinSeen:false RstSeen:false Whitelisted:true Opener:true Ifindex:12} OrigDst:0.0.0.0 OrigSrc:0.0.0.0 OrigPort:0 OrigSPort:0 TunIP:0.0.0.0}} Age: 48.829949311s Active ago 48.8285382s
```

_______________________________________________________________________________________________________________________________________________________________________

### Troubleshoot Calico Enterprise eBPF

Troubleshooting eBPF at time could require enabling `debug` logging. In this section, we will learn how to enable `debug` logging for eBPF.

1. Set Felix `bpfLogLevel` to `Debug`. Note that enabling debug logging causes a firehose of logs to the BPF trace pipe and should only be activated once requsted by Tigera support team. Run the following command to enable debug logging.

```
kubectl patch felixconfiguration default --patch='{"spec": {"bpfLogLevel": "Debug"}}'

```

2. To view the logs, we need to first connect to a node. SSH into `worker1`.

```
ssh worker1

```

3. Run the following command to view the logs. Press `ctrl+c` once done. Note that every decision point for every packet is logged!

```
sudo tc exec bpf debug

```
You should see firehose of output similar to the following.

`<...>-84582 [000] .Ns1  6851.690474: 0: ens192---E: Final result=ALLOW (-1). Program execution time: 7366ns`                                                     

4. Following is a short description of the output above.

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


### Revert back to iptables dataplane

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
