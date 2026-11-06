# Part 5 - Troubleshoot

## Situation 1 - Worker cannot authenticate

### Most likely cause

The client certificate is expired on October 10, 2026, when the request has been sent on October 12, 2026. 
On the `worker-1`, validate certificate:
```bash
sudo openssl x509 \
  -in /var/lib/kubelet/pki/kubelet-client-current.pem \
  -noout -dates -subject -issuer
```

### How to confirm

On the worker:

```bash
timedatectl status
sudo openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem \
  -noout -subject -issuer -dates
sudo journalctl -u kubelet -n 100 --no-pager
```

Expected result:
```bash
notAfter=Oct 10 08:02:11 2026 GMT
```

You should patch the certificate with the new one, or verify if the automatic certificate rotation is enable and something else is broken.

To verify automatic certificate rotation:
```bash
sudo grep -E 'certificate-authority|client-certificate|client-key' \
  /etc/kubernetes/kubelet.conf
```

and look for `--rotate-certificates=true`.

## Situation 2 - A deleted control-plane node remains in etcd

### Most likely cause

`cp-4` was removed as a VM but its etcd member was never removed. The cluster still has four members, so quorum is three. One member is permanently gone, which means losing one more healthy member leaves only two of four members and the cluster loses quorum. Before `cp-4` was added, three members needed two for quorum and could tolerate one loss.

### How to confirm

Run these on a healthy control-plane node with the etcd certificates:

```bash
export ETCDCTL_API=3
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  endpoint health --cluster

etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  member list -w table
```

Match the `c41d...` member to `10.0.10.14` and confirm that the VM is really gone, not just temporarily unreachable. Check whether a stale `cp-4` Node object still exists with `kubectl get nodes`.

### Fix

Take and verify an etcd snapshot first. Then remove only the stale member by its full ID:

```bash
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  member remove <full-member-id>
```

Remove the stale Kubernetes Node object if it is still there, then recheck `member list` and `endpoint health --cluster`. The cluster should be back to three members and tolerate one failure. Do not remove a member just because it is temporarily unreachable; if quorum is already lost, normal membership changes will not work and the snapshot restore or etcd recovery procedure must be used instead.

## Situation 3 - A Deployment stays Pending

### Most likely cause

The pod does not fit on `worker-1` because its memory request is larger than the remaining allocatable memory. The three control-plane nodes are tainted with `node-role.kubernetes.io/control-plane:NoSchedule`, and this pod does not tolerate that taint. There is therefore no eligible node.

### How to confirm

```bash
kubectl describe pod web-6d4f9c7b8-kx2lp
kubectl describe node worker-1
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
kubectl top node
```

`describe pod` shows the scheduler's `FailedScheduling` reason. `describe node worker-1` shows allocatable memory and the sum of existing requests. Compare that with the pod's request.

### What I would answer my colleague

No, I would not remove the control-plane taint. The taint protects the capacity and stability of the API server and etcd, and keeps ordinary team workloads off the control plane. Removing it does not create memory on the worker, so the pod may still not fit or may compete with control-plane components. The correct fix is to add or resize worker capacity, reduce the pod's memory request if it is oversized, or free memory on the worker. A toleration should only be added for a specific workload that genuinely needs to run on the control plane.

## Situation 4 - A team application cannot reach its database

Since the NetworkPolicy is a security control I wouldn't delete it; deleting it would remove network isolation rather than fix the underlying configuration.

The evidence suggests a network connectivity problem affecting both DNS and database access:

- nslookup db times out: the pod cannot reach its configured DNS resolver.
- nc -zv 10.96.41.7 5432 times out: the pod cannot establish a TCP connection to the database Service.

So, the likely cause is that the environment template creates a default-deny egress polict without allowing DNS and database traffic.

The solution is to retain default-deny and explicitly allow the required connections:

- DNS egress to CoreDNS on UDP/TCP 53.
- Application-to-database traffic on TCP 5432, restricted by pod labels and namespace.
- Required ingress/egress traffic only.
