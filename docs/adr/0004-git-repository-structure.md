# ADR 0004: Git repository structure

- **Status:** Accepted
- **Decision:** Keep cluster bootstrap in EasyKind and platform configuration, team requests and the reference workload in the `kiratech-challenge` repository.

## Context

The cluster must be reproducible, and teams need to request environments through Git. Reconciliation ownership should be clear so bootstrap tools and Argo CD do not fight over the same resources.

## Decision

EasyKind owns cluster creation, Calico and kube-vip bootstrap, Argo CD installation and node-level operations. The `kiratech-challenge` repository contains the root GitOps Kustomization, platform policies and charts, Envoy Gateway and its GatewayClass, request manifests under `gitops/team-requests/`, and the sample workload under `gitops/team-workloads/blue/`. Argo CD reconciles the root path. A child Application watches team requests, each request Application creates its environment from the chart, and a generated child Application reconciles the team's workload repository.

## Consequences

Bootstrap and application lifecycle remain separate. A team environment is added by committing one request manifest; deleting that request from Git causes Argo CD to prune it and its managed resources. In this reference, the sample workload lives in the same repository. In a production setup, each team can use a separate repository with its own ownership and access controls. Do not commit repository credentials to Git.
