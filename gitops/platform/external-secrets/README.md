# External Secrets Operator

Argo CD Application that installs the [External Secrets Operator](https://external-secrets.io/) Helm chart. The operator watches `SecretStore`, `ClusterSecretStore`, `ExternalSecret` and related resources and syncs values from an external secret manager into Kubernetes Secrets.

## Install

Apply the Argo CD Application once:

```bash
kubectl --context kind-kiratech-kind-cluster apply \
  -f gitops/platform/external-secrets/application.yaml
```

Or apply the directory or all platform add-ons:

```bash
kubectl --context kind-kiratech-kind-cluster apply \
  -k gitops/platform/external-secrets

kubectl --context kind-kiratech-kind-cluster apply \
  -k gitops/platform
```

After the Application exists, Argo CD manages the operator. The Application uses automated sync with `prune: false` and `selfHeal: true`, creates the `external-secrets` namespace, and installs the chart CRDs.

## Version

The chart is pinned to `2.14.0` from `https://charts.external-secrets.io`. Change `spec.source.targetRevision` in `application.yaml` to upgrade.

## Verify

```bash
kubectl -n argocd get application external-secrets
kubectl -n external-secrets get pods
kubectl get crds | grep external-secrets.io
```

Expected resources include:

- `external-secrets` controller
- `external-secrets-cert-controller`
- `external-secrets-webhook`
- CRDs such as `externalsecrets.external-secrets.io`, `secretstores.external-secrets.io` and `clustersecretstores.external-secrets.io`

## Next step

The operator does not connect to a secret manager by itself. Create a `SecretStore` or `ClusterSecretStore` for your provider, then create `ExternalSecret` resources that reference it. Provider credentials should come from the platform's workload identity or a Kubernetes Secret; do not commit them to this repository.

## Notes

- This directory contains the Argo CD `Application`, not the provider configuration.
- EasyKind's `Configure GitOps` menu does not create this Application yet; apply it once with `kubectl`, or add it to an app-of-apps Application.
- The chart repository is public. No Helm repository credentials are required.
