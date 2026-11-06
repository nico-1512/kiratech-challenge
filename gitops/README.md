# GitOps layout

This directory is consumed by Argo CD after the cluster has been bootstrapped with EasyKind. The bootstrap layer creates the cluster, the CNI, the node-level tools and Argo CD itself; Argo then owns the Kubernetes objects below.

## Managed by Argo CD

| Component | Path | Objects |
|---|---|---|
| `restore-demo` | `apps/restore-demo` | Namespace, ConfigMap, Deployment and Service used by the etcd restore exercise |
| `nginx-probe` | `apps/nginx-probe` | nginx Deployment, Service, PodDisruptionBudget and the continuous probe |
| `external-secrets` | `platform/external-secrets` | External Secrets Operator |
| `envoy-gateway` | `platform/envoy-gateway` | Envoy Gateway Helm release pinned to `v1.9.3` |
| `envoy-gateway-class` | `platform/envoy-gateway-class` | GatewayClass `eg` |
| `team-environments` | `platform/team-environments` | Team admission policy and request watcher |

The root Argo CD Application tracks the `gitops/` path and reconciles these workloads and platform resources. Its `prune: false` setting avoids deleting resources missing from Git; dedicated child Applications such as team workloads use pruning where deletion is part of their lifecycle.

## Part 3: team environment self-service

The implementation uses an Argo CD `Application` as the single Git request. A ValidatingAdmissionPolicy checks every `team-*` Application created in the `argocd` namespace. The request is rejected at API admission if the team name is missing or invalid, the size is not `S`, `M` or `L`, or the Git URL is missing or malformed. The Helm chart also has `values.schema.json` as a second validation layer when Argo renders it.

The team-environment chart creates the team's namespace, size-based ResourceQuota, LimitRange, namespaced developer Role and group RoleBinding, default-deny NetworkPolicies, and a child Argo CD Application for the team's repository. The allow policy permits same-namespace pod traffic and DNS queries to CoreDNS; all other cross-namespace traffic stays denied. The child Application prunes its workloads, and the request Application has Argo's resources finalizer so deleting the request cascades through the child Application and removes the environment.

The admission policy and a small Argo CD Application for the `team-requests/` directory are installed from this repository's root GitOps application. After they are Synced and Healthy, edit `teamRepository` in `team-requests/team-blue.yaml` to the team's Git repository and commit that single request resource. Argo CD applies it to the remote VM cluster.

The request manifest points its source at this repository's chart. To request another team, copy the file and change the Application name and `teamName` to the same DNS label, select `S`, `M` or `L`, and provide the team's repository URL. `teamPath` is optional and defaults to `.`; it must be a repository-relative path. The example uses `gitops/team-workloads/blue` in this repository, where an `api` Deployment resolves `db` through cluster DNS and fetches its HTTP service. The same namespace and DNS allowances make this work under the default-deny policy. Replace `teamRepository` and `teamPath` with the team's repository and workload path for a real team. See [Part 3 documentation](../docs/part3/building-team-environments.md) for the full workflow and sizing details.

Check the request and generated resources:

```bash
kubectl --context <remote-vm-context> -n argocd get applications
kubectl --context <remote-vm-context> -n team-blue get resourcequota,limitrange,role,rolebinding,networkpolicy
kubectl --context <remote-vm-context> -n team-blue logs deploy/api
```

An invalid size can be used to confirm API-side rejection:

```bash
kubectl --context <remote-vm-context> apply --dry-run=server -f /path/to/invalid-team-request.yaml
```

To remove an environment, delete its request file from `team-requests/` and commit that change. The directory Application prunes the request, and its finalizer removes the workload Application and environment. Deleting only the live Application with `kubectl` will not persist while its request remains in Git.

The management plane and workload environment are on the same remote VM cluster in this reference implementation. For a separate management cluster, change the generated workload Application's destination to the registered target cluster; the request and chart remain in the management cluster.

## What Argo CD can manage

Use Argo CD for:

- Application workloads under `apps/`, such as `restore-demo` and `nginx-probe`
- Post-bootstrap add-ons such as External Secrets Operator, Envoy Gateway, Gateway API routes and other Helm/Kustomize components
- Additional applications added later under `apps/`

`Configure GitOps` asks for a repository path and creates one Argo `Application` for it. To manage all challenge resources in one application, use `gitops`; its root Kustomization includes both workloads and platform resources. You can also enter a narrower path such as `gitops/apps/restore-demo`, `gitops/apps/nginx-probe` or `gitops/platform/external-secrets`.

## Platform add-ons

| Component | Path | What it installs |
|---|---|---|
| `external-secrets` | `platform/external-secrets` | External Secrets Operator `2.14.0` from `https://charts.external-secrets.io` |
| `envoy-gateway` | `platform/envoy-gateway` | Envoy Gateway `v1.9.3` from the OCI Helm chart |
| `envoy-gateway-class` | `platform/envoy-gateway-class` | GatewayClass `eg` for Envoy Gateway |

The root `gitops/` Application includes these add-ons through `platform/kustomization.yaml`. To install an add-on independently, apply its Argo CD Application manifest once. Envoy Gateway installs Gateway API CRDs; a separate Application creates GatewayClass `eg` after those CRDs are available.

```bash
kubectl -n argocd get applications
kubectl -n envoy-gateway-system rollout status deployment/envoy-gateway --timeout=300s
kubectl get gatewayclass eg
```

For private UI access, use `kubectl -n argocd port-forward svc/argocd-server 8080:443`. Add Gateway, HTTPRoute and TLS resources to Git before exposing services through Envoy Gateway.

## Kept in EasyKind bootstrap

These are not Argo-managed because they are host, node or CNI bootstrap concerns:

- kind cluster, Docker network, kubeconfig and the HAProxy API endpoint
- Calico installation (Argo needs pod networking before it can run)
- kube-vip and its Docker-network-specific LoadBalancer range
- Argo CD installation itself
- `etcdctl`/`etcdutl` binaries on control-plane nodes
- kubeadm/kubelet upgrades, drains and etcd backup/restore

The EasyKind installation scripts remain available for all of these.

## Registering the repository

From the EasyKind menu, run `Configure GitOps`, or:

```bash
./scripts/configure_gitops
```

The script asks for the Git URL, branch, authentication method and repository path. It registers an Argo CD repository Secret and creates one Argo `Application` for that path. The path defaults to the repository root (`.`).

There is no default repository URL. If you use this repository, enter:

```text
https://github.com/nico-1512/kiratech-challenge.git
```

For automation, set `GITOPS_REPO_URL` and optionally `GITOPS_REPO_PATH` instead of typing them at the prompts.

### Private repository authentication

Choose the authentication method at the prompt:

- **Public**: no credentials are stored.
- **Private HTTPS**: enter a Git username and a personal access token with read access to the repository contents. The token is stored as the Argo CD Secret `password` field.
- **Private SSH**: enter an SSH repository URL such as `git@github.com:owner/repo.git` and the path to the private key. The key is stored as `sshPrivateKey`. GitHub/GitLab SSH host keys are already trusted by Argo CD.

For GitHub HTTPS, create a fine-grained personal access token with **Contents: Read-only** permission for the repository. Use your GitHub username (or `x-access-token`) as the username and the token as the password. For SSH, add the matching public key as a deploy key with read access.

For a self-hosted SSH server, add its host key to Argo CD's `argocd-ssh-known-hosts-cm` ConfigMap or use the Argo CD UI under **Settings → Certificates**. The host key must be unhashed `known_hosts` data, for example from `ssh-keyscan git.example.com`.

Argo CD stores these values in a repository Secret in the `argocd` namespace:

```bash
kubectl -n argocd get secret repo-<name> -o yaml
```

Do not commit credentials to this repository. The Argo CD Secret is the credential store.

## Adding another Argo-managed application

1. Add a directory under `gitops/apps/<name>` with a `kustomization.yaml`.
2. Run `Configure GitOps` and enter `apps/<name>` as the repository path.
3. To manage the whole repository from an app-of-apps, enter `.` instead.

For a platform add-on, add an Argo CD `Application` under `gitops/platform/<name>`, add it to `platform/kustomization.yaml`, then point `Configure GitOps` at `platform` or apply it once.

Do not add Calico, Argo CD bootstrap, kube-vip bootstrap, node binaries or the Kubernetes upgrade to this directory. Those stay in the bootstrap/manual layer.
