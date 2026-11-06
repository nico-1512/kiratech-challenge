# Part 1 - Build the cluster

This document describes how the reference cluster is built and why the control plane is arranged this way. The cluster is a 3 control-plane + 1 worker kind cluster running on a remote VM. No local kind cluster is involved. The build is scripted with [EasyKind](https://github.com/nico-1512/easykind) using `easykind/assets/config-cluster.yaml`, Calico is installed as the CNI, and kube-vip is installed for `LoadBalancer` services.

## 1a. How the API endpoint is made highly available

The cluster runs three active `kube-apiserver` instances, one on each control-plane node. There is no active/passive API server: all three serve requests. Clients need one stable address, so the API is published through a single endpoint that load-balances across the three instances and removes unhealthy backends.

### Stable endpoint in kind

In this kind cluster the endpoint is provided by kind itself. When a kind cluster has more than one control-plane node, kind creates an external load balancer container (HAProxy) and writes its address into the kubeconfig.

Check the endpoint used by `kubectl`:

```bash
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
```

Example output:
```
https://127.0.0.1:36543
```

Check the load balancer container:

```bash
docker ps --filter name=external-load-balancer --format '{{.Names}} {{.Image}}'
```

![external-load-balancer](../src/part1/external-load-balancer.png)

### kube-vip

EasyKind installs the kube-vip cloud provider and DaemonSet with `easykind/scripts/kube_vip`. The DaemonSet is generated with:

```bash
docker run --rm --network host ghcr.io/kube-vip/kube-vip:<version> \
  manifest daemonset --services --inCluster --arp --interface eth0
```

The kube-vip cloud provider watches `Service` objects of type `LoadBalancer` and allocates an IP from the `kubevip` ConfigMap (`range-global`). The DaemonSet runs on the nodes, elects a leader per service and advertises the VIP with ARP on `eth0`. If the node holding the VIP fails, another kube-vip instance takes the VIP over.

Verify the installation:

```bash
kubectl -n kube-system get ds kube-vip-ds
kubectl -n kube-system get cm kubevip -o yaml
```

Example output:
```
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-vip-ds   4         4         4       4            4           <none>          5m
```

In the current kind setup this covers `LoadBalancer` services, not the API endpoint, because the script uses `--services`. The API endpoint is the kind HAProxy container described above.

### On real VMs

On real VMs the same kube-vip is deployed in **control-plane mode** (`--controlplane`) with a reserved VIP. kubeadm is configured with `controlPlaneEndpoint: https://<VIP>:6443`, and kube-vip runs on all control-plane nodes. One instance advertises the VIP (ARP on L2, BGP on L3), and failover is a VIP move. There is no dedicated load-balancer VM or container.

kind's HAProxy container is a single point of failure for the API. This is a limitation of kind, not of the three-control-plane design. On real VMs kube-vip owns the VIP directly on the control-plane nodes, so losing the VIP holder only moves the address to another control-plane node.

## 1b. Where etcd runs and why

### Stacked etcd

etcd is **stacked**: every control-plane node runs its own etcd member, and each `kube-apiserver` talks to the etcd member on its own node.

Query the etcd cluster from a control-plane node:

```bash
docker exec kind-control-plane sh -c \
  "ETCDCTL_API=3 etcdctl \
   --endpoints=https://127.0.0.1:2379 \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key \
   member list -w table"
```
![etc-cluster-query](../src/part1/etc-cluster-query.png)

### Why stacked

- It is the kubeadm default and follows directly from the required topology: 3 control-plane nodes give a 3-member etcd cluster, which tolerates the loss of one member (quorum is 2 of 3).
- No extra VMs are needed for a separate etcd cluster.
- The API server communicates with the local etcd member, which keeps latency low.
- There is one lifecycle to operate: the control-plane nodes and etcd are upgraded and backed up as one unit.
- It is the simplest correct choice for this cluster size; external etcd becomes worthwhile at larger scale or when etcd must be scaled and maintained independently.

### Trade-offs

- The control plane and etcd share the same failure domain and the same CPU, memory and disk. A control-plane node loss also removes an etcd member.
- Losing one control-plane node is tolerated; losing two leaves etcd without quorum and the control plane cannot accept writes.
- Heavy API load or etcd disk latency can affect the control-plane nodes directly.

### In kind

The etcd members run inside the control-plane node containers as static pods managed by kubelet, with data under `/var/lib/etcd` and certificates under `/etc/kubernetes/pki/etcd`. `install_etcdctl` places `etcdctl` and `etcdutl` on every control-plane node so snapshots and restores (Part 2b) can be performed from the nodes themselves.

## 1c. Which CNI was chosen and why

### Chosen CNI: Calico

The cluster uses **Calico**. The kind config disables the default CNI and sets the pod subnet to match Calico's default IPv4 pool:

```yaml
networking:
  disableDefaultCNI: true
  podSubnet: "192.168.0.0/16"
```

EasyKind installs Calico after the cluster is created with `easykind/scripts/install_calico`. The script applies the upstream Calico manifest for the latest release (or `CALICO_VERSION`), waits for `calico-node` and `calico-kube-controllers`, and then waits for all nodes to become Ready. If the cluster still runs kindnet, the script refuses to install and requires the cluster to be recreated from this config.

Verify the CNI:

```bash
kubectl -n kube-system get ds calico-node
kubectl -n kube-system get deploy calico-kube-controllers
kubectl -n kube-system get pods -l k8s-app=calico-node -o wide
```

![calico-cni](../src/part1/calico-cni.png)

### Why Calico

- It enforces Kubernetes `NetworkPolicy`, which Part 3 requires: a default-deny policy is only useful if the data plane actually enforces it.
- It is a mature, widely deployed CNI with a straightforward upstream manifest that fits the scripted EasyKind workflow; no Helm chart or operator is required for this reference cluster.
- It provides both pod networking and policy, so there is no need to run a second CNI only for policy.
- It is also usable on real VM clusters and supports routed/BGP and overlay modes when the cluster moves off kind.
- Its default IPv4 pool is `192.168.0.0/16`, matching the kind `podSubnet`, so the manifest needs no extra patch.

### Why not kindnet

kindnet is the kind default and is sufficient for basic connectivity, but it does not enforce `NetworkPolicy`. A default-deny policy would be accepted by the API and silently ignored by the data plane, which would make the Part 3 isolation requirement false. Cilium was a valid alternative (eBPF, Hubble); Calico was chosen for its mature policy support and the simple manifest-based installation.

## What would be different on real VMs

- Nodes are separate VMs instead of Docker containers, so the cluster survives a failure of the Docker host and nodes have their own kernel, disks and network interfaces.
- The API VIP is owned by kube-vip in control-plane mode (ARP or BGP), or by an HAProxy + keepalived pair. There is no kind-managed HAProxy container and no single load-balancer SPOF.
- Calico runs on the VM network instead of the Docker bridge. On real VMs it must be configured for routed/BGP or overlay networking, and the pod CIDR must not overlap with the datacentre network.
- etcd runs on the control-plane VM disks (or on a separate external etcd cluster), not inside containers on one host.
- Host ports 80/443 are no longer mapped from one VM to a container; ingress is exposed through a `LoadBalancer` service or a dedicated ingress VIP.
- Upgrades are done in place with kubeadm (one node at a time) instead of recreating kind nodes from a newer node image.
