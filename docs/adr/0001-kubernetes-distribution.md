# ADR 0001: Kubernetes distribution

- **Status:** Accepted
- **Decision:** Use kubeadm for the target VM-based cluster; use Kind with kubeadm-built nodes for this reference implementation.

## Context

The client needs reproducible, highly available Kubernetes clusters on VMs. The challenge allows kubeadm, RKE2 or k3s. The current reference cluster runs as a multi-node Kind cluster on a VM so the exercises can be repeated without provisioning separate node VMs.

## Decision

Choose kubeadm for the target VM cluster. It is the upstream bootstrap tool, gives direct control over the Kubernetes components and fits the challenge's focus on operating the control plane, stacked etcd, upgrades and recovery. The reference uses EasyKind scripts to create the Kind nodes; Kind bootstraps those nodes with kubeadm.

## Consequences

The VM implementation must automate OS prerequisites, container runtime setup, kubeadm configuration, certificate handling and node upgrades. RKE2 would provide a more packaged distribution with useful defaults; k3s would reduce resource use for edge or smaller environments. Both are reasonable alternatives, but kubeadm best matches the operational learning goals here. Kind is a lab stand-in and does not provide the node-level failure isolation of separate VMs.
