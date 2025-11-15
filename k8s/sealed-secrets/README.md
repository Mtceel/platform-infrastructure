# 🔐 Sealed Secrets - Encrypted Secrets Management

## Overview

Sealed Secrets provides a way to safely store encrypted secrets in Git. The secrets are encrypted using the cluster's public key and can only be decrypted by the Sealed Secrets controller running in your Kubernetes cluster.

## Architecture

```
┌─────────────────┐
│   Developer     │
│  (kubeseal CLI) │
└────────┬────────┘
         │ 1. Create secret
         │ 2. Encrypt with cluster public key
         ▼
┌─────────────────┐
│  Git Repository │
│ (Encrypted YAML)│
└────────┬────────┘
         │ 3. Commit encrypted secret
         │ 4. Push to GitHub
         ▼
┌─────────────────┐
│     ArgoCD      │
│  (GitOps Sync)  │
└────────┬────────┘
         │ 5. Deploy SealedSecret to cluster
         ▼
┌─────────────────────────────────┐
│  Sealed Secrets Controller       │
│  (kube-system namespace)         │
│  • Decrypts with private key     │
│  • Creates regular K8s Secret    │
└────────┬────────────────────────┘
         │ 6. Create decrypted Secret
         ▼
┌─────────────────┐
│  Application    │
│  (Uses Secret)  │
└─────────────────┘
```

## Current Sealed Secrets

### 1. PostgreSQL Credentials
**File:** `postgres-sealed-secret.yaml`

Contains encrypted PostgreSQL password for the database StatefulSet.

**Usage in deployment:**
```yaml
env:
- name: POSTGRES_PASSWORD
  valueFrom:
    secretKeyRef:
      name: postgres-credentials
      key: postgres-password
```

### 2. JWT Secret
**File:** `jwt-sealed-secret.yaml`

Contains encrypted JWT secret for API token signing.

**Usage in deployment:**
```yaml
env:
- name: JWT_SECRET
  valueFrom:
    secretKeyRef:
      name: jwt-secret
      key: jwt-secret
```

### 3. Docker Hub Credentials
**File:** `docker-sealed-secret.yaml`

Contains encrypted Docker Hub username and password for private image pulls.

**Usage in deployment:**
```yaml
imagePullSecrets:
- name: docker-credentials
```

## Deployment

Apply all sealed secrets to the cluster:

```bash
kubectl apply -f k8s/sealed-secrets/
```

The Sealed Secrets controller will automatically decrypt them and create regular Kubernetes secrets.

## Verify Secrets

Check if secrets were created successfully:

```bash
# List all secrets in platform-services namespace
kubectl get secrets -n platform-services

# View secret details (base64 encoded values)
kubectl get secret postgres-credentials -n platform-services -o yaml

# Decode secret value (example)
kubectl get secret postgres-credentials -n platform-services -o jsonpath='{.data.postgres-password}' | base64 -d
```

## How to Create New Sealed Secrets

### Step 1: Install kubeseal CLI

```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.26.0/kubeseal-0.26.0-linux-amd64.tar.gz
tar -xzf kubeseal-0.26.0-linux-amd64.tar.gz
sudo mv kubeseal /usr/local/bin/
```

### Step 2: Create a regular Kubernetes secret (dry-run)

```bash
kubectl create secret generic my-secret \
  --namespace=platform-services \
  --from-literal=my-key=my-value \
  --dry-run=client -o yaml > /tmp/my-secret.yaml
```

### Step 3: Seal the secret

```bash
kubeseal -f /tmp/my-secret.yaml -o yaml > k8s/sealed-secrets/my-sealed-secret.yaml
```

### Step 4: Commit to Git

```bash
git add k8s/sealed-secrets/my-sealed-secret.yaml
git commit -m "Add sealed secret for X"
git push
```

### Step 5: ArgoCD auto-deploys it

ArgoCD will detect the change and deploy the sealed secret to your cluster.

## Security Benefits

✅ **Safe to commit to Git** - Secrets are encrypted with cluster's public key  
✅ **No plaintext secrets** - Original secret is never stored  
✅ **Cluster-specific** - Can only be decrypted by your cluster  
✅ **Automatic decryption** - Controller handles decryption automatically  
✅ **GitOps compatible** - Works seamlessly with ArgoCD  
✅ **Audit trail** - Git history shows when secrets were changed  

## Rotation Strategy

To rotate a secret:

1. Generate new secret value
2. Create new sealed secret with same name
3. Commit and push to Git
4. ArgoCD updates the secret
5. Restart pods that use the secret

Example:
```bash
# Generate new password
NEW_PASSWORD=$(openssl rand -base64 32)

# Create and seal new secret
kubectl create secret generic postgres-credentials \
  --namespace=platform-services \
  --from-literal=postgres-password=$NEW_PASSWORD \
  --dry-run=client -o yaml | kubeseal -o yaml > postgres-sealed-secret.yaml

# Commit and push
git add postgres-sealed-secret.yaml
git commit -m "Rotate PostgreSQL password"
git push

# Restart pods to use new secret
kubectl rollout restart statefulset postgres -n platform-services
```

## Backup & Disaster Recovery

The Sealed Secrets controller uses a private key to decrypt secrets. **Back up this key!**

```bash
# Backup the private key
kubectl get secret -n kube-system sealed-secrets-key -o yaml > sealed-secrets-master-key.yaml

# Store this file SECURELY (not in Git!)
# If you lose this key, you'll need to re-seal all secrets
```

## Troubleshooting

### Secret not decrypting

Check controller logs:
```bash
kubectl logs -n kube-system -l name=sealed-secrets-controller
```

### Re-seal all secrets

If you restored a cluster or lost the private key:
```bash
# Delete old sealed secrets
kubectl delete sealedsecrets -n platform-services --all

# Re-create and re-seal all secrets
# (You'll need the original plaintext values)
```

## Reference

- Sealed Secrets GitHub: https://github.com/bitnami-labs/sealed-secrets
- Documentation: https://sealed-secrets.netlify.app/
- Controller version: v0.26.0
