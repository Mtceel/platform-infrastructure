# 🏗️ Platform Infrastructure

Core Kubernetes infrastructure for the SaaS Platform with GitOps automation.

## 📦 What's Included

### **Databases** (`k8s/databases/`)
- ✅ **PostgreSQL 15** - Main database (5GB persistent storage)
- ✅ **Redis 7** - Cache & sessions (512MB memory, LRU eviction)

### **ArgoCD** (`k8s/argocd/`)
- ✅ GitOps automation watching 5 GitHub repos
- ✅ Auto-sync enabled (deploys within 3 minutes)
- ✅ UI: https://argocd.fv-company.com

## 🚀 Quick Deploy

```bash
# 1. Create namespace
kubectl apply -f k8s/namespaces/

# 2. Deploy databases
kubectl apply -f k8s/databases/

# 3. Setup ArgoCD
kubectl apply -f k8s/argocd/
```

## 📊 Managed Services

ArgoCD auto-deploys from GitHub:
- admin-dashboard
- merchant-dashboard (Blue/Green)
- status-page
- platform-api
- storefront-renderer

## 🔐 Database Access

**PostgreSQL:** `postgres.platform-services.svc.cluster.local:5432`
**Redis:** `redis.platform-services.svc.cluster.local:6379`

See `k8s/databases/README.md` for details.

## 🔗 Links

- ArgoCD: https://argocd.fv-company.com
- GitHub: https://github.com/Mtceel?tab=repositories
