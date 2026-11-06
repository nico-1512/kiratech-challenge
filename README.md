# Kiratech P4 Challenge

Reference implementation of the Kiratech Cloud Native Engineer P4 challenge: a Kubernetes cluster on a remote VM, built with [EasyKind](https://github.com/nico-1512/easykind), and a GitOps platform on top of it with a self-service team environment API.

## What is included

- 3 control-plane nodes and 1 worker (kind on a remote VM)
- kubeadm, stacked etcd, Calico CNI and kube-vip
- Argo CD managing the cluster from Git
- Team environments created from a single Argo CD `Application` request
- Admission validation for team name, size and repository URL
- Default-deny NetworkPolicies with only the required DNS and in-namespace traffic
- Documentation for all five parts of the challenge

## Repository layout

| Path | Contents |
| --- | --- |
| `docs/part1` | Cluster build, HA API endpoint, etcd and CNI choices |
| `docs/part2` | Failure tolerance, etcd backup/restore and upgrade |
| `docs/part3` | Team environment self-service API |
| `docs/part4` | Note on AI assistant use |
| `docs/part5` | Troubleshooting answers |
| `docs/adr` | Architecture decision records |
| `docs/architecture` | C4 container and deployment diagrams |
| `gitops/` | Argo CD-managed workloads and platform components |
| `Challenge_P4.pdf` | Original challenge description |

## Documentation

- [Part 1 - Build the cluster](docs/part1/building-the-cluster.md)
- [Part 2 - Operate the control plane](docs/part2/operating-control-plane.md)
- [Part 3 - Build the platform control plane](docs/part3/building-team-environments.md)
- [Part 4 - Work with AI assistants](docs/part4/ai-assistant-use.md)
- [Part 5 - Troubleshoot](docs/part5/troubleshooting.md)
- [Architecture diagrams](docs/architecture/README.md)
- [GitOps layout](gitops/README.md)