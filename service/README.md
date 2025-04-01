
## Service Announcements

For this lab, there are 2 services (red and blue) to be used for testing:

1) The service subnet was defined in the ```cluster.yaml``` file as "10.2.0.0/16,fd00:10:2::/108" 
2) The service **blue** has internal and external traffic policy set to ```Local```, so the service IP will only be advertised from the node with the backend service (replicas set to 1)
3) The service **red** does not have traffic policies set, therefore the default setting ```Cluster``` is used and the service IP will be advertised from all nodes (replicas set to 4)
4) 192.168.100.10/32 and 192.168.200.10/32 are external IPs and will follow the same pattern relying on what is defined for the ```externalTrafficPolicy``` configuration

### Example

```
$ kubectl get services -A
NAMESPACE     NAME           TYPE        CLUSTER-IP    EXTERNAL-IP                           PORT(S)                  AGE
default       kubernetes     ClusterIP   10.2.0.1      <none>                                443/TCP                  2d2h
kube-system   cilium-envoy   ClusterIP   None          <none>                                9964/TCP                 2d2h
kube-system   hubble-peer    ClusterIP   10.2.46.54    <none>                                443/TCP                  2d2h
kube-system   kube-dns       ClusterIP   10.2.0.10     <none>                                53/UDP,53/TCP,9153/TCP   2d2h
tenant-blue   service-blue   NodePort    10.2.215.58   192.168.100.10,2001:192:168:100::10   1234:32688/TCP           2d1h
tenant-red    service-red    NodePort    10.2.199.40   192.168.200.10,2001:192:168:200::10   1236:31124/TCP           2d1h

$ kubectl get pods -o wide -n tenant-red
NAME                       READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
curl-red-fbf845459-h8qhd   1/1     Running   0          32s   10.244.1.243   k8s-worker    <none>           <none>
curl-red-fbf845459-jhp2l   1/1     Running   0          32s   10.244.2.13    k8s-worker3   <none>           <none>
curl-red-fbf845459-jrbfm   1/1     Running   0          32s   10.244.3.127   k8s-worker4   <none>           <none>
curl-red-fbf845459-qnxqs   1/1     Running   0          32s   10.244.4.97    k8s-worker2   <none>           <none>

$ kubectl get pods -o wide -n tenant-blue
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
curl-blue-64b574954-v72tq   1/1     Running   0          41s   10.244.4.82   k8s-worker2   <none>           <none>
```

For the service blue scenario, spines will only see one path for the service IP (10.2.x.x) and external IP (192.168.100.10), so traffic will be localized on a single leaf where the node is hosting the service blue. For the service red scenario, spines will see advertisements from all leaf swiches for service IP (10.2.x.x) and external IP (192.168.200.10), and traffic will be load balanced across the cluster nodes.

**Route Table**

```
spine1# sh ip route bgp 
10.2.182.126/32, ubest/mbest: 1/0
    *via fe80::e59:16ff:fe00:101%default, Eth1/3, [20/0], 00:12:07, bgp-auto, external, tag 4279440147
10.2.198.222/32, ubest/mbest: 3/0
    *via fe80::e59:16ff:fe00:101%default, Eth1/3, [20/0], 00:12:11, bgp-auto, external, tag 4279440147
    *via fe80::ec0:73ff:fe00:101%default, Eth1/4, [20/0], 00:12:10, bgp-auto, external, tag 4270534441
    *via fe80::ecb:9ff:fe00:101%default, Eth1/1, [20/0], 00:12:10, bgp-auto, external, tag 4237290796
192.168.100.10/32, ubest/mbest: 1/0
    *via fe80::e59:16ff:fe00:101%default, Eth1/3, [20/0], 00:12:07, bgp-auto, external, tag 4279440147
192.168.200.10/32, ubest/mbest: 3/0
    *via fe80::e59:16ff:fe00:101%default, Eth1/3, [20/0], 00:12:11, bgp-auto, external, tag 4279440147
    *via fe80::ec0:73ff:fe00:101%default, Eth1/4, [20/0], 00:12:10, bgp-auto, external, tag 4270534441
    *via fe80::ecb:9ff:fe00:101%default, Eth1/1, [20/0], 00:12:10, bgp-auto, external, tag 4237290796
```

Leaf and spines are configured with ```bestpath as-path multipath-relax``` and ```maximum-paths``` to allow for additional paths and BGP load balancing based on leaf next hop. Additional configuration can be applied to allow for node next hop load balancing when there is an unbalanced number of nodes across different racks. 