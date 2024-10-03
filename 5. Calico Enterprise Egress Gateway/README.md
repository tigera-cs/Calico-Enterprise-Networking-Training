# In this lab

This lab provides the instructions to:

* [Overview](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/tree/main/5.%20Calico%20Enterprise%20Egress%20Gateway#overview)
* [Implement Calico Enterprise Egress Gateway](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/tree/main/5.%20Calico%20Enterprise%20Egress%20Gateway#implement-calico-enterprise-egress-gateway)



### Overview

In Kubernetes, Ingress traffic refers to any traffic that is initiated from outside the cluster to the services running inside the Kubernetes cluster. Egress traffic is just the opposite, any traffic that is initiated from the pods from within the cluster to the IP endpoints that are located outside the cluster. Kubernetes provides a native ingress resource to manage and control Ingress traffic. However, there is no native Kubernetes Egress resource. So when pods need to connect to an endpoint outside the cluster, they do so using their own IP addresses by default. Considering that an application could be implemented through one or more pods and the fact that pods in Kubernetes are ephemeral, it is almost impossible to identify and control the Kubernetes egress traffic from outside the cluster as the IP addresses are constantly changing. Egress gateways (EGWs) are pods that act as gateways for traffic leaving the cluster from certain client pods. The primary function of egress gateway is to configure the egress gateway client to have a particular and persistent source IP address when connecting to services outside the Kubernetes cluster.


After finshing this lab, you should gain a good understanding of how to deploy Calico Enterprise Egress Gateway and establish a permanent identity for the traffic that is leaving the cluster.

______________________________________________________________________________________________________________________________________________________________________

### Implement Calico Enterprise Egress Gateway


1. Enable egress gateway support by patching FelixConfiguration to support egress gateway both per namespace and per pod.

```
kubectl patch felixconfiguration.p default --type='merge' -p '{"spec":{"egressIPSupport":"EnabledPerNamespaceOrPerPod"}}'
    
```
2. Egress gateways require the Policy Sync API to be enabled in `felixconfiguration` to implement symmetric routing. Run the following command to enable this configuration cluster-wide.

```
kubectl patch felixconfiguration.p default --type='merge' -p '{"spec":{"policySyncPathPrefix":"/var/run/nodeagent"}}'
    
```

3. Egress gateways use the IPPool resource for a particular application when it connects outside of the cluster. Run the following command to create the egress gateway IPPool.

```
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: egress-ippool-1
spec:
  cidr: 10.10.10.0/30
  blockSize: 31
  nodeSelector: "!all()"
EOF

```

4. Let's create a namespace and application, which will be using egress gateway to connect to resources outside the cluster.

```
kubectl apply -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: app1
  labels:
    tenant: tenant1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-deployment
  namespace: app1
  labels:
    app: app1
    tenant: tenant1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
      tenant: tenant1
  template:
    metadata:
      labels:
        app: app1
        tenant: tenant1
    spec:
      containers:
      - name: app1
        image: praqma/network-multitool
        env:
        - name: HTTP_PORT
          value: "1180"
        - name: HTTPS_PORT
          value: "11443"
        ports:
        - containerPort: 1180
          name: http-port
        - containerPort: 11443
          name: https-port
        resources:
          requests:
            cpu: "1m"
            memory: "20Mi"
          limits:
            cpu: "10m"
            memory: "20Mi"
EOF

```

5. Make sure the pods are running.

```
kubectl get pods -n app1

```

6. Egress gateway image needs to be downloaded onto the the nodes where egress gateway pods are deployed. We need to identify the pullsecret that is needed for pulling Calico Enterprise images and copy it into the namespace where you plan to create your egress gateways. Calico Enterprise by default uses a secret named `tigera-pull-secret`. Run the following command to copy `tigera-pull-secret` from `calico-system` namespace to the `default` namespace.

```
kubectl get secret tigera-pull-secret --namespace=calico-system -o yaml | grep -v '^[[:space:]]*namespace:[[:space:]]*calico-system' | kubectl apply --namespace=default -f -

```

7. Deploy the Egress gateway by browsing to the following link, copy the egress-gateway deployment, and deploy it in the cluster. Make sure the version of egress-gateway image matches the Calico Enterprise version deployed in the cluster.

https://docs.tigera.io/networking/egress/egress-gateway-on-prem#deploy-a-group-of-egress-gateways 

**Note:** Since Tigera continously updates this manifest with the new features and configurations paramters, this manifest should be downloaded from the Tigera docs site. Following is a sample manifest from the docs site.

Note the following configurations:

* `egress-code: red` is the lable that we will use to annotate the egress gateway client pod or namespace to use the egress gateway as it egress proxy.
* `cni.projectcalico.org/ipv4pools: "[\"10.10.10.0/31\"]"` is the IPPool that was deployed to provide IP address to egress gateway pods. 


```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: egress-gateway
  namespace: default
  labels:
    egress-code: red
spec:
  replicas: 1
  selector:
    matchLabels:
      egress-code: red
  template:
    metadata:
      annotations:
        cni.projectcalico.org/ipv4pools: "[\"10.10.10.0/30\"]"
      labels:
        egress-code: red
    spec:
      imagePullSecrets:
      - name: tigera-pull-secret
      nodeSelector:
        kubernetes.io/os: linux
      initContainers:
      - name: egress-gateway-init
        command: ["/init-gateway.sh"]
        image: quay.io/tigera/egress-gateway:v3.15.5
        env:
         # Use downward API to tell the pod its own IP address.
        - name: EGRESS_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        # Optional: Used to specify IP tables backend (legacy or nft).
        # For example on RHEL9, nft can be used.
    # As of v3.15.3, the egress gateway will attempt to autodetect
    # the correct value preferring `nft` if available.
        #- name: IPTABLES_BACKEND
        #  value: "legacy"
        securityContext:
          privileged: true
      containers:
      - name: egress-gateway
        command: ["/start-gateway.sh"]
        image: quay.io/tigera/egress-gateway:v3.15.5
        env:
        # Optional: comma-delimited list of IP addresses to send ICMP pings to; if all probes fail, the egress
        # gateway will report non-ready.
        - name: ICMP_PROBE_IPS
          value: ""
         # Only used if ICMP_PROBE_IPS is non-empty: interval to send probes.
        - name: ICMP_PROBE_INTERVAL
          value: "5s"
         # Only used if ICMP_PROBE_IPS is non-empty: timeout before reporting non-ready if there are no successful
         # ICMP probes.
        - name: ICMP_PROBE_TIMEOUT
          value: "15s"
         # Optional comma-delimited list of HTTP URLs to send periodic probes to; if all probes fail, the egress
         # gateway will report non-ready.
        - name: HTTP_PROBE_URLS
          value: ""
        # Only used if HTTP_PROBE_URL is non-empty: interval to send probes.
        - name: HTTP_PROBE_INTERVAL
          value: "10s"
        # Only used if HTTP_PROBE_URL is non-empty: timeout before reporting non-ready if there are no successful
        # HTTP probes.
        - name: HTTP_PROBE_TIMEOUT
          value: "30s"
        # Port that the egress gateway serves its health reports.  Must match the readiness probe and health
        # port defined below.
        - name: HEALTH_PORT
          value: "8080"
        - name: EGRESS_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
        volumeMounts:
        - mountPath: /var/run
          name: policysync
        ports:
        - name: health
          containerPort: 8080
        readinessProbe:
          httpGet:
            path: /readiness
            port: 8080
          initialDelaySeconds: 3
          periodSeconds: 3
      terminationGracePeriodSeconds: 0
      volumes:
      - csi:
          driver: csi.tigera.io
        name: policysync
EOF

```
8. Make sure that the egress gateway pod is running.

```
kubectl get pods -o wide

```

```
NAME                               READY   STATUS    RESTARTS   AGE    IP            NODE                                      NOMINATED NODE   READINESS GATES
egress-gateway-74c977bb77-5bxsm    1/1     Running   0          133m   10.10.10.0    ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
nginx-deployment-9456bbbf9-2vkfx   1/1     Running   0          23h    10.48.0.208   ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
nginx-deployment-9456bbbf9-6g6gl   1/1     Running   0          23h    10.48.0.38    ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
nginx-deployment-9456bbbf9-rkl9s   1/1     Running   0          23h    10.48.0.37    ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
```

9. We will now need to configure our egress gateway client (app1 application) to use the egress gateway. We could configure specific pods/deployments in app1 namespace to use the egress gateway by annotating the pods/deployment or all the pods/deployments in app1 namespace by annotating the namespace. Since we only have a single app in app1 namespace, we will configure the annotation on the namespace.

```
kubectl annotate ns app1 egress.projectcalico.org/selector='egress-code == "red"'

```
10. By default, the above selector can only match egress gateways in the same namespace. To select gateways in a different namespace, specify a namespaceSelector annotation as well. Since our egress gateway pod is running in `default` namespace and our egress gateway client is running in `app1` namespace, we need to run the following command.

```
kubectl annotate ns app1 egress.projectcalico.org/namespaceSelector="projectcalico.org/name == 'default'"

```

11. Deploy the needed BGPConfiguration and BGPPeer so we route our traffic to the `bastion` host through the egress gateway.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-64512
spec:
  peerIP: 10.0.1.10
  asNumber: 64512
---
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceClusterIPs:
  - cidr: 10.49.0.0/16
  communities:
  - name: bgp-large-community
    value: 64512:120
  prefixAdvertisements:
  - cidr: 10.10.10.0/31
    communities:
    - bgp-large-community
    - 64512:120
EOF

```

12. `Bastion` host is simulating our upstream router and we should have BGP sessions established to all the cluster nodes using the above configurations. Run the following command on the `bastion` node to validate the bgp sessions from the `bastion` node to the cluster nodes.

```
sudo birdc show protocols
```

```
BIRD 1.6.8 ready.
name     proto    table    state  since       info
direct1  Direct   master   up     23:42:33    
kernel1  Kernel   master   up     23:42:33    
device1  Device   master   up     23:42:33    
control1 BGP      master   up     23:42:35    Established   
worker1  BGP      master   up     23:42:36    Established   
worker2  BGP      master   up     23:42:35    Established
```
We must also set the bird config on the bastion to accept advertisements of the egress gateway network.

```
sudo vi /etc/bird/bird.conf
```
In this file add the 10.10.10.0/31 network here:

```
# Import filter
filter rt_import {
                        if (net ~ 10.48.2.0/24) then accept;
                        if (net ~ 10.49.0.0/16) then accept;
                        if (net ~ 10.50.0.0/24) then accept;
                        if (net ~ 10.10.10.0/30) then accept; <-------
                        reject;
        }
```
And restart bird

```
sudo systemctl restart bird
```
13. Check the routes on the `bastion` node. You should see that the edge gateway pod is reachable through the worker node where it has been deployed.

```
ip route
```
```
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.10.10.0/31 via 10.0.1.31 dev ens5 proto bird 
10.48.2.216/29 via 10.0.1.31 dev ens5 proto bird 
10.49.0.0/16 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1
```

14. Let verify our egress gateway configurations in action and ensure that `app1` IP address are proxied by egress gateway when connecting to services outside the cluster. Open a second terminal to your lab instance. On that terminal, start netcat on the `bastion` host to listen to a specific port.

```
netcat -nvlkp 7777

```

15. On the original terminal window, exec into any of the pods in the app1 namespace.

```
APP1_POD=$(kubectl get pod -n app1 --no-headers -o name | head -1) && echo $APP1_POD

```
```
kubectl exec -ti $APP1_POD -n app1 -- sh
```

16. Try to connect to the port in the bastion host.

```
nc -zv 10.0.1.10 7777

```

17. Type `exit` to exit out the pod terminal.

```
exit
```

18. Go to the terminal that you ran the following netcat server on the `bastion` node. You should see an output saying you connected from the IP of the egress gateway pod to the netcat server.

```
tigera@bastion:~$ netcat -nvlkp 7777
Listening on 0.0.0.0 7777
Connection received on 10.10.10.0 39457tigera@bastion:~$ netcat -nvlkp 7777
Listening on 0.0.0.0 7777
Connection received on 10.10.10.0 39457
```

19. Stop the netcat listener process in the `bastion` host with `^C`.

20. Let's check on the special routing that Felix has implemented to proxy the `app1` traffic through the egress gateway. Check where the egress gateway client pods are running. 

```
kubectl get pods -n app1 -o wide

```

```
NAME                               READY   STATUS    RESTARTS   AGE     IP            NODE                                      NOMINATED NODE   READINESS GATES
app1-deployment-6ccc9654cb-c8ztq   1/1     Running   0          3h50m   10.48.0.60    ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
app1-deployment-6ccc9654cb-wjnfl   1/1     Running   0          3h50m   10.48.0.210   ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
```

21. Log into the worker node where the egress gateway client pod `app1` is running. In this case, there are two pods and each one is running on a different worker node. Choose a node where the `app1` is running on and ssh into that node. Note `app1` pod IP address running on worker2. It is `10.48.0.60` in this case.

```
ssh worker2
```

22. Observe the routing policy that is programmed for the the `app1` workload POD IP address `10.48.0.60`.

```
ip rule
```

```
0:      from all lookup local
100:    from 10.48.0.60 fwmark 0x80000/0x80000 lookup 250
32766:  from all lookup main
32767:  from all lookup default
```

23. Confirm that the policy is choosing the egress gateway as the next hop for any source traffic from `app1`.
#### Note: ensure to use the correct table number with the following command. In this case, the table number is 250.

```
ip route show table 250

```

```
default via 10.10.10.0 dev egress.calico onlink 
```

24. Starting Calico Enterprise 3.15, egress gateway supports using HTTP and ICMP probes for readiness health checking. If all probes of one type `ICMP/HTTP` fail, egress gateway pod failing the probe goes non-Ready. Let's configure our egress-gateway deployment to ping bastion node `10.0.1.10` for readiness check. Should this ping fail, egress gateway pod should go non-ready. Use the following command and set the value for `ICMP_PROBE_IPS` to `10.0.1.10`. Then save and exit. Egress gateway pod should restart because of this change.

```
kubectl edit deployments.apps egress-gateway

```

![patch-egress-gateway-deployment](img/1.egress-gateway-deployment-patch.JPG)

25. Make sure that the egress gateway pod is running by running the following command.

```
kubectl get pods -o wide   
```

26. Let's now implement a security policy that blocks ICMP traffic from egress gateway pod to `10.0.1.10`, but allows every other traffic.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: default.egress-gateway-allow-everything-deny-ping-to-bastion
  namespace: default
spec:
  tier: default
  selector: egress-code == "red"
  serviceAccountSelector: ''
  egress:
    - action: Deny
      protocol: ICMP
      source: {}
      destination:
        nets:
          - 10.0.1.10/32
    - action: Allow
      source: {}
      destination: {}
  types:
    - Egress
EOF

```

27. `ICMP_PROBE_TIMEOUT` environment variable is configured to `15s` by default. It should take about 15-20s after implementing the above policy for the egress gateway pod to show as non-ready as depiected below.

```
Every 2.0s: kubectl get pods -o wide                                                                                                                                                                                        bastion: Wed Jan  4 01:42:09 2023

NAME                               READY   STATUS    RESTARTS   AGE     IP            NODE                                      NOMINATED NODE   READINESS GATES
egress-gateway-5fb89cf946-snznd    0/1     Running   0          3m25s   10.10.10.0    ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
nginx-deployment-9456bbbf9-2vkfx   1/1     Running   0          25h     10.48.0.208   ip-10-0-1-30.eu-west-1.compute.internal   <none>           <none>
nginx-deployment-9456bbbf9-6g6gl   1/1     Running   0          25h     10.48.0.38    ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>
nginx-deployment-9456bbbf9-rkl9s   1/1     Running   0          25h     10.48.0.37    ip-10-0-1-31.eu-west-1.compute.internal   <none>           <none>****
```

28. Follow the same connectivity steps that were done from step 14-18 above. You should still be able to make connections from `app1` to `10.0.1.10` as shown below and the connections should appear to be coming from egress gateway pod. The reason is that egress gateway daemon is still running in the container and just the ICMP probe has failed 

```
Listening on 0.0.0.0 7777
Connection received on 10.10.10.0 35615
Connection received on 10.10.10.0 39185
Connection received on 10.10.10.0 36269
Connection received on 10.10.10.0 42397
Connection received on 10.10.10.0 44511
Connection received on 10.10.10.0 42293
```

29. Clean up the resources that were deployed for the purpose of this lab.

```
kubectl delete ns app1

```
```
kubectl delete networkpolicies.pro egress-gateway-allow-everything-deny-ping-to-bastion

```
```
kubectl delete deployments egress-gateway

```

> **Congratulations! You have completed `5. Calico Enterprise Egress Gateway` lab.**
