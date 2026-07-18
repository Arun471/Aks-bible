# AKS Project Context

This file gives Claude Code persistent context about our AKS environment.
Fill in the placeholders, then keep it updated as the cluster evolves.

## Cluster Overview
- **Subscription**: <subscription-name-or-id>
- **Resource group**: <rg-name>
- **Cluster name**: <cluster-name>
- **Region**: <e.g. eastus>
- **Kubernetes version**: <e.g. 1.31>
- **Environment**: <dev / staging / prod>
- **Node pools**: <e.g. system pool (2x Standard_D4s_v5), user pool (autoscale 3-10x Standard_D8s_v5, spot)>

## Networking
- **Network model**: <Azure CNI Overlay / Azure CNI / Kubenet>
- **Ingress**: <e.g. AGIC / App Gateway, NOT nginx>
- **API server access**: <public / private, authorized IP ranges if any>
- **DNS**: <e.g. Azure DNS zone name>

## Identity & Security
- **Auth**: <Entra ID + Azure RBAC / local accounts>
- **Workload identity**: <enabled/disabled, which workloads use it>
- **Azure Policy**: <baseline / restrictive / none>
- **Key Vault integration**: <CSI driver name if used>

## Observability
- **Monitoring**: <Managed Prometheus + Grafana / Container Insights>
- **Logs**: <Log Analytics workspace name>
- **Alerting**: <where alerts go — Slack channel, PagerDuty, etc.>

## Deployment
- **CI/CD**: <GitHub Actions / Azure DevOps / ArgoCD, pipeline name or repo>
- **Package manager**: <Helm charts location / Kustomize>
- **Rollout strategy**: <blue-green / canary / rolling>

## Known Gotchas
<Running list — add to this whenever you solve something non-obvious>
- Example: "Node pool X uses spot instances — expect occasional evictions, don't page on those alone."
- Example: "Ingress health checks fail if pod doesn't respond within 2s, not the k8s default."

## Common Commands / Runbooks
```
# Get cluster credentials
az aks get-credentials -g <rg-name> -n <cluster-name>

# Check node health
kubectl get nodes -o wide

# Recent events across namespaces
kubectl get events -A --sort-by='.lastTimestamp'
```

## Working Agreements for Claude
- Default to `readonly` MCP access — never run write/delete operations without explicit confirmation.
- Before suggesting `kubectl apply` or `az aks` mutating commands, explain what will change and why.
- When troubleshooting, start with `aks_detector` / diagnostic tools before guessing.
- Flag anything that touches prod differently from dev/staging.
