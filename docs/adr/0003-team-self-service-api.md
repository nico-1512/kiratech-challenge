# ADR 0003: Team self-service API

- **Status:** Accepted
- **Decision:** Use Argo CD Applications, Kubernetes ValidatingAdmissionPolicy and a Helm chart (Option B).

## Context

The platform must accept one declarative request with a team name, size and repository, reject invalid values at API admission, create the team controls and workload Application, and clean up on deletion.

## Decision

Represent each request as a `team-*` Argo CD Application in Git. A ValidatingAdmissionPolicy checks the name, size, repository URL and optional path. The request Application renders the team-environment Helm chart, which creates the namespace, quota, limits, RBAC, network policies and child workload Application. Argo CD pruning and resource finalizers provide deletion behavior.

## Consequences

This reuses Argo CD already installed in the reference cluster and keeps the request and resource templates in Git. The chart schema provides a second validation layer, while the admission policy rejects invalid requests before Argo renders them. Krateo (Option A) offers a richer platform API and portal experience, but would add a larger platform footprint than needed for this first capability. This design is intentionally small; a larger service catalog or user portal may justify a dedicated platform layer later.
