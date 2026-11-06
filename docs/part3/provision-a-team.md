# Provision a team

1. Copy `gitops/team-requests/team-blue.yaml` to `gitops/team-requests/team-<name>.yaml`.
2. Set `metadata.name` and `teamName` to the same DNS-safe name.
3. Choose size `S`, `M` or `L`.
4. Set `teamRepository` and `teamPath` to the team's workload in Git. For this example, use this repository and `gitops/team-workloads/blue`.
5. Commit the file. Argo CD creates the team namespace and deploys its workloads.

Check the result with:

```bash
kubectl -n argocd get applications
kubectl get namespace team-<name>
```

To remove the team, delete its request file from Git and commit the change.
