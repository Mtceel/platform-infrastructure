# Platform Infrastructure

Core Kubernetes configuration for the multi-tenant SaaS platform.

## Structure

- `k8s/` - Kubernetes manifests (namespaces, databases, networking)
- `terraform/` - Infrastructure as Code
- `argocd/` - ArgoCD application definitions

## Deploy

```bash
kubectl apply -f k8s/namespaces/
kubectl apply -f k8s/databases/
kubectl apply -f k8s/argocd/
```

## Services

All services are deployed via ArgoCD from their respective repos:
- MTceel/admin-dashboard
- MTceel/merchant-dashboard
- MTceel/status-page
- MTceel/platform-api
- MTceel/storefront-renderer
