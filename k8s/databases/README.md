# 🗄️ Database & Cache Layer

## Overview

Persistent data layer for the SaaS platform:

- **PostgreSQL 15:** Main application database (5GB persistent storage)
- **Redis 7:** Caching, sessions, rate limiting (512MB memory limit)

Both run as **StatefulSets** in Kubernetes with persistent volumes.

---

## 📊 Current Setup

### PostgreSQL
- **Image:** `postgres:15-alpine`
- **Database:** `saas_platform`
- **User:** `postgres`
- **Password:** `postgres123` ⚠️ (TODO: Move to Sealed Secret)
- **Storage:** 5GB PersistentVolumeClaim
- **Connection:** `postgres.platform-services.svc.cluster.local:5432`

### Redis
- **Image:** `redis:7-alpine`
- **Memory Limit:** 512MB
- **Eviction Policy:** `allkeys-lru` (Least Recently Used)
- **Persistence:** AOF (Append Only File)
- **Storage:** 5GB PersistentVolumeClaim
- **Connection:** `redis.platform-services.svc.cluster.local:6379`

---

## 🚀 Deployment

```bash
# Apply both databases
kubectl apply -f k8s/databases/

# Check status
kubectl get statefulsets -n platform-services
kubectl get pods -n platform-services | grep -E "postgres|redis"
kubectl get pvc -n platform-services
```

---

## 🔍 Access & Management

### Connect to PostgreSQL

```bash
# Via kubectl exec
kubectl exec -it postgres-0 -n platform-services -- psql -U postgres -d saas_platform

# Via port-forward (from local machine)
kubectl port-forward -n platform-services postgres-0 5432:5432
psql -h localhost -U postgres -d saas_platform
```

### Connect to Redis

```bash
# Via kubectl exec
kubectl exec -it redis-0 -n platform-services -- redis-cli

# Via port-forward (from local machine)
kubectl port-forward -n platform-services redis-0 6379:6379
redis-cli -h localhost
```

---

## 📝 Common Operations

### PostgreSQL Operations

```bash
# Backup database
kubectl exec -n platform-services postgres-0 -- \
  pg_dump -U postgres saas_platform > backup-$(date +%Y%m%d).sql

# Restore database
cat backup-20251115.sql | \
  kubectl exec -i -n platform-services postgres-0 -- \
  psql -U postgres saas_platform

# Check database size
kubectl exec -n platform-services postgres-0 -- \
  psql -U postgres -d saas_platform -c \
  "SELECT pg_size_pretty(pg_database_size('saas_platform'));"

# List tables
kubectl exec -n platform-services postgres-0 -- \
  psql -U postgres -d saas_platform -c "\dt"

# Check connections
kubectl exec -n platform-services postgres-0 -- \
  psql -U postgres -c \
  "SELECT datname, count(*) FROM pg_stat_activity GROUP BY datname;"
```

### Redis Operations

```bash
# Check memory usage
kubectl exec -n platform-services redis-0 -- redis-cli INFO memory

# Check keys
kubectl exec -n platform-services redis-0 -- redis-cli DBSIZE

# Monitor commands
kubectl exec -n platform-services redis-0 -- redis-cli MONITOR

# Flush cache (WARNING: Deletes all data!)
kubectl exec -n platform-services redis-0 -- redis-cli FLUSHALL

# Check persistence
kubectl exec -n platform-services redis-0 -- redis-cli BGSAVE
```

---

## 🔐 Security TODO

### Move Passwords to Sealed Secrets

```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Create sealed secret for PostgreSQL
kubectl create secret generic postgres-credentials \
  --from-literal=password=postgres123 \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > postgres-sealed-secret.yaml

# Update StatefulSet to use secret
env:
- name: POSTGRES_PASSWORD
  valueFrom:
    secretKeyRef:
      name: postgres-credentials
      key: password
```

---

## 📈 Scaling

### Vertical Scaling (More Resources)

```bash
# Edit StatefulSet
kubectl edit statefulset postgres -n platform-services

# Update resources:
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"
```

### Storage Expansion

```bash
# Check current PVC
kubectl get pvc -n platform-services

# Edit PVC (if storage class supports expansion)
kubectl edit pvc postgres-storage-postgres-0 -n platform-services

# Change:
spec:
  resources:
    requests:
      storage: 10Gi  # Was 5Gi
```

### Redis Replication (Future)

For high availability, deploy Redis Sentinel:

```bash
# Use Bitnami Helm chart
helm install redis bitnami/redis \
  --set architecture=replication \
  --set sentinel.enabled=true
```

---

## 🔄 Backup & Restore

### Automated Backup Strategy

Create CronJob for daily backups:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: platform-services
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15-alpine
            command:
            - /bin/sh
            - -c
            - |
              pg_dump -h postgres -U postgres saas_platform > /backup/backup-$(date +%Y%m%d).sql
              # Upload to S3 or external storage
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

---

## 🚨 Monitoring & Alerts

### Health Checks

```bash
# PostgreSQL health
kubectl exec -n platform-services postgres-0 -- pg_isready

# Redis health
kubectl exec -n platform-services redis-0 -- redis-cli ping
```

### Metrics Endpoints

Configure exporters for Prometheus:

```bash
# PostgreSQL Exporter
helm install postgres-exporter prometheus-community/prometheus-postgres-exporter \
  --set config.datasource.host=postgres.platform-services.svc.cluster.local

# Redis Exporter
helm install redis-exporter prometheus-community/prometheus-redis-exporter \
  --set redisAddress=redis://redis.platform-services.svc.cluster.local:6379
```

---

## 🔧 Troubleshooting

### PostgreSQL Not Starting

```bash
# Check logs
kubectl logs -n platform-services postgres-0

# Common issues:
# - Insufficient storage
# - Corrupted data directory
# - Wrong permissions

# Reset data (WARNING: Deletes all data!)
kubectl delete pvc postgres-storage-postgres-0 -n platform-services
kubectl delete pod postgres-0 -n platform-services
```

### Redis Memory Issues

```bash
# Check memory usage
kubectl exec -n platform-services redis-0 -- redis-cli INFO memory

# If maxmemory reached:
# 1. Increase maxmemory limit in StatefulSet
# 2. Clear old keys: redis-cli FLUSHDB
# 3. Adjust eviction policy
```

### Connection Issues

```bash
# Test from another pod
kubectl run -it --rm debug --image=alpine --restart=Never -- sh

# Inside pod:
apk add postgresql-client
psql -h postgres.platform-services.svc.cluster.local -U postgres

# For Redis:
apk add redis
redis-cli -h redis.platform-services.svc.cluster.local
```

---

## 📊 Database Schema Management

### Migrations

Use migration tool (e.g., Flyway, Liquibase):

```bash
# Example with Node.js migrations
npm install knex pg

# Create migration
npx knex migrate:make create_users_table

# Run migrations
npx knex migrate:latest
```

### Schema Versioning

Keep schema in Git:

```sql
-- migrations/001_create_users.sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 🎯 Best Practices

1. **Never hardcode passwords** - Use Sealed Secrets or External Secrets Operator
2. **Regular backups** - Automate with CronJobs
3. **Monitor metrics** - Setup Prometheus exporters
4. **Test restores** - Verify backups work monthly
5. **Separate envs** - Different databases for dev/staging/prod
6. **Connection pooling** - Use PgBouncer for PostgreSQL
7. **Redis for cache only** - Don't use as primary database
8. **Set resource limits** - Prevent OOM kills

---

## 🔗 Connection Strings

### From within K8s cluster:

```bash
# PostgreSQL
postgresql://postgres:postgres123@postgres.platform-services.svc.cluster.local:5432/saas_platform

# Redis
redis://redis.platform-services.svc.cluster.local:6379
```

### Environment Variables (for apps):

```yaml
env:
- name: DATABASE_URL
  value: postgresql://postgres:postgres123@postgres.platform-services.svc.cluster.local:5432/saas_platform
- name: REDIS_URL
  value: redis://redis.platform-services.svc.cluster.local:6379
```

---

## 📚 Next Steps

1. **Setup Sealed Secrets** for password management
2. **Configure automated backups** with CronJobs
3. **Add Prometheus exporters** for metrics
4. **Setup Redis Sentinel** for high availability
5. **Implement connection pooling** with PgBouncer
6. **Create separate databases** for dev/staging/prod

---

**Status:** Both databases running with persistent storage ✅
