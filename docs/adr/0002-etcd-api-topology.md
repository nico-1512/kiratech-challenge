# ADR 0002: etcd and API endpoint topology

- **Status:** Accepted
- **Decision:** Use three stacked etcd members, one on each control-plane node, and a stable API endpoint in front of all three API servers.

## Context

The cluster has three control-plane nodes and must tolerate one control-plane failure. The API endpoint must remain stable while requests are routed to healthy API servers.

## Decision

Use stacked etcd on the three control-plane nodes. A three-member quorum tolerates one member failure and avoids separate etcd hosts. In the current Kind reference, Kind provides the stable endpoint through its HAProxy load-balancer container. On VM deployments, provide the endpoint with a highly available VIP or load-balancer pair, and configure kubeadm's `controlPlaneEndpoint` to use it.

## Consequences

The current reference makes the HAProxy container a single point of failure; this is a Kind limitation. A VM deployment must make the endpoint itself highly available, monitor etcd quorum and disk latency, and back up etcd regularly. Losing two of three etcd members removes quorum and prevents normal writes until members are recovered or a restore is performed. kube-vip in the current reference allocates `LoadBalancer` service addresses; it does not own the Kubernetes API endpoint.
