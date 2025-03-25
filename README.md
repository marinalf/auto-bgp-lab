
# Cisco Nexus Auto-BGP fabric with K8s and Cilium CNI

This lab is running on a single topology host based on Ubuntu 24.04. It leverages containerlab to run Nexus 9kv as containers, and K8s with Cilium CNI using [KinD](https://containerlab.dev/manual/kinds/k8s-kind/), making it very simple and accessible to test features like auto-bgp fabric and CNI integrations. 

Disclaimer: for enterprise features support, please contact your Isovalent Account Team

### Install Dependencies

[docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository),
[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/),
[helm](https://helm.sh/docs/intro/install/),
[kind](https://kind.sigs.k8s.io/#installation-and-usage),
[containerlab](https://containerlab.dev/install/)

Kind Install Example

```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

On your topology host, use [vrnetlab](https://containerlab.dev/manual/vrnetlab/) to convert a Nexus 9kv image (.qcow2) into a container using the following steps:

1) Download a lite version of [Nexus 9kv](https://software.cisco.com/download/home/286312239/type/282088129/release/10.5(2)) from Cisco (credentials required). This lab is using nexus9300v64-lite.10.5.2.F.qcow2.
2) Install vrnetlab with ```git clone https://github.com/hellt/vrnetlab && cd vrnetlab/n9kv``` and add the .qcow2 image inside the "n9kv" folder. 
3) Before running ```make docker-image``` inside the n9kv folder, ensure the "[Makefile](https://github.com/hellt/vrnetlab/tree/master/n9kv)" is updated with the right expression to match your image such as ```/nexus9300v64-\```.

Once conversion is complete, check ```docker images``` to verify container image has been created. This will be used in the containerlab topology along with kind type [cisco_n9kv].

### Repository Structure

Back to your home directory (cd /home/$User), create your working folder or ```git clone https://github.com/marinalf/auto-bgp-lab.git && cd auto-bgp-lab```

| File | Description |
| ------------- | ------------- |
| n9kv-k8s.clab.yml | topology file defining a spine and leaf fabric  |
| config  | folder containing auto-bgp config for all switches |
| cluster.yaml    | definition for the K8s nodes with 1 control plane and 4-worker nodes |
| cilium-values.yaml     | initial config to install Cilium with helm  |
| cilium-bgp.yaml   | definition for BGP peering and policies on Cilium CNI |
| cilium-cli.sh    | script to install Cilium CLI  |

### Topology File

Here is an excerpt for the Nexus 9kv configuration in the *.clab.yml file.

```
topology:
  defaults:
    kind: cisco_n9kv
  kinds: 
    cisco_n9kv:
      image: vrnetlab/cisco_n9kv:lite.10.5.2.F
      env:
        QEMU_MEMORY: 6144 # N9kv-lite requires minimum 6GB memory
        QEMU_SMP: 2 # N9kv-lite requires minimum 2 CPUs
        CLAB_MGMT_PASSTHROUGH: "true" # support for different management IP
  nodes:
    leaf1:
      mgmt-ipv4: 172.20.1.101
      startup-config: config/leaf1.txt
    leaf2:
      mgmt-ipv4: 172.20.1.102
      startup-config: config/leaf2.txt
```

### Deployment Steps

**K8s**
1) Build K8s cluster: ```kind create cluster --config=cluster.yaml```

**Nexus Auto-BGP Fabric**

2) Deploy topology file: ```containerlab deploy -t n9kv-k8s.clab.yml```

**Cilium CNI**

3) Add helm repo for Cilium: ```helm repo add cilium https://helm.cilium.io```
4) Install Cilium with Helm: ```helm install cilium cilium/cilium --version <your image> --namespace kube-system -f cilium-values.yaml```. 
5) Deploy Cilium CLI: ```./cilium-cli.sh```, and ensure file has the right permission [chmod +x ./cilium-cli.sh]

**NOTE**: Before applying the BGP config on Cilium CNI, update the ```cilium-bgp.yaml``` file with the auto-generated AS number from the leaf switches [credentials admin/admin]

```
ssh admin@clab-n9kv-k8s-leaf1
show ip bgp summary
```

6) Deploy BGP peering/policies on Cilium CNI: ```kubectl apply -f cilium-bgp.yaml```



### Verification & Troubleshooting Commands

* clab inspect -t n9kv-k8s.clab.yml
* kind get clusters
* kubectl version
* kubectl cluster-info --context kind-k8s
* kubectl get node k8s-control-plane -o yaml
* kubectl get nodes -o wide
* kubectl get pods -o wide
* kubectl get pod -n kube-system
* kubectl get pods -A
* kubectl config view
* docker ps
* docker images
* docker exec -it k8s-worker4 ip a show eth0
* docker exec -it k8s-control-plane sh
* helm repo list
* helm list --namespace kube-system
* cilium status
* cilium config view | grep ipv
* kubectl get ds -n kube-system cilium
* kubectl -n kube-system rollout restart ds/cilium
* cilium bgp peers
* cilium bgp routes
* cilium connectivity test


### Clean-Up

* helm uninstall cilium -n kube-system
* clab destroy -t n9kv-k8s.clab.yml
* kind delete cluster -n k8s