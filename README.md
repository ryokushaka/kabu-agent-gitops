# Kabu Agent GitOps Repository

Kubernetes manifests for Kabu Agent, managed by ArgoCD.

## Structure

```
manifests/
├── argocd/           # ArgoCD Application & Project
├── kabu-agent/       # Application manifests
│   ├── backend/      # FastAPI backend
│   ├── postgres/     # PostgreSQL database
│   └── redis/        # Redis cache
└── monitoring/       # (Optional) Monitoring stack
```

## Setup

### 1. Update Placeholders

Before using, replace the following placeholders:

| Placeholder | Replace With | Files |
|-------------|--------------|-------|
| `OWNER` | Your GitHub username | `backend/deployment.yaml`, `argocd/*.yaml` |
| `IMAGE_TAG` | `latest` (initially) | `backend/deployment.yaml` |
| `DOMAIN_NAME` | Your domain | `backend/ingress.yaml` |

### 2. Create Sealed Secrets

```bash
# On your K3s server (where sealed-secrets controller runs)

# 1. Copy secrets-template.yaml and fill actual values
cp manifests/kabu-agent/secrets-template.yaml /tmp/secrets.yaml
# Edit /tmp/secrets.yaml with real values

# 2. Seal the secret
kubeseal --format yaml < /tmp/secrets.yaml > manifests/kabu-agent/sealed-secrets.yaml

# 3. Delete plaintext secret
rm /tmp/secrets.yaml

# 4. Commit sealed secret
git add manifests/kabu-agent/sealed-secrets.yaml
git commit -m "chore: add sealed secrets"
git push
```

### 3. Register with ArgoCD

```bash
# Apply ArgoCD application
kubectl apply -f manifests/argocd/project.yaml
kubectl apply -f manifests/argocd/application.yaml
```

## CI/CD Flow

```
kabu-agent repo          GitOps repo              K3s Cluster
     │                        │                        │
     │ push to main           │                        │
     ├───────────────────────►│                        │
     │                        │                        │
     │ build-push.yml         │                        │
     │ (Docker build & push)  │                        │
     │                        │                        │
     │ repository_dispatch    │                        │
     ├───────────────────────►│                        │
     │                        │                        │
     │                        │ update-manifest.yml    │
     │                        │ (Update image tag)     │
     │                        │                        │
     │                        │ ArgoCD detects change  │
     │                        ├───────────────────────►│
     │                        │                        │
     │                        │                        │ Deploy
```

## Manual Sync

```bash
# Force sync via ArgoCD CLI
argocd app sync kabu-agent

# Or via kubectl
kubectl -n argocd patch application kabu-agent \
  --type merge \
  -p '{"operation": {"initiatedBy": {"username": "admin"}, "sync": {}}}'
```
