# RUNBOOK-002: Argo CD Self-Healing and Drift Remediation

## Objective

Validate that Argo CD automatically detects and remediates
configuration drift between the Git-defined desired state and
the live Kubernetes cluster.

## Application

- Argo CD Application: `demo-app-dev`
- Environment: `dev`
- Namespace: `demo-app`
- Git repository: `azure-devsecops-platform-gitops`
- Git path: `apps/demo-app/overlays/dev`
- Deployment: `demo-app`

## Desired State

The GitOps manifest defines:

- Replicas: `2`
- Automated sync: enabled
- Self-healing: enabled
- Pruning: enabled

The live Argo CD configuration confirmed:

```text
automated:
  prune: true
  selfHeal: true