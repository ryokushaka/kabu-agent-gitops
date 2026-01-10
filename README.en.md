# Kabu Agent GitOps Repository

Kubernetes manifests for Kabu Agent, managed by ArgoCD.

## Structure

```
manifests/
├── argocd/           # ArgoCD Application & Project
├── kabu-agent/       # Application manifests
│   ├── backend/      # FastAPI backend
│   ├── frontend/     # React frontend
│   ├── postgres/     # PostgreSQL database
│   └── redis/        # Redis cache
├── istio/            # Istio service mesh
├── kiali/            # Kiali dashboard
├── monitoring/       # Prometheus & Grafana stack
│   ├── alertmanager.yaml
│   ├── grafana-dashboards.yaml
│   ├── grafana-values.yaml
│   └── prometheus-values.yaml
└── tracing/          # Distributed tracing (Jaeger)
```

## Setup

### 1. Update Placeholders

Before using, replace the following placeholders:

| Placeholder | Replace With | Files |
|-------------|--------------|--------|
| `OWNER` | Your GitHub username | `backend/deployment.yaml`, `argocd/*.yaml` |

**Done for ryokushaka/kabu-agent-gitops**

### 2. Access Without Domain

Since no domain is configured, access services via NodePort:

```bash
# Get NodePort for backend
kubectl -n kabu-agent get svc kabu-backend

# Access via Elastic IP:NodePort
# Example: http://106.73.5.224:30080
```

Services:
- **Backend**: `EIP:30080`
- **Frontend**: `EIP:30000`
- **ArgoCD**: `https://EIP:30443` (use ArgoCD password from EC2)
- **Grafana**: `EIP:30300`
- **Prometheus**: `EIP:30090`
- **Kiali**: `EIP:30201`

### 3. Create Sealed Secrets (After K3s Deployed)

```bash
# On your K3s server (where sealed-secrets controller runs)

# 1. Copy secrets-template.yaml and fill actual values
cp manifests/kabu-agent/secrets-template.yaml.example /tmp/secrets.yaml
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

### 4. Register with ArgoCD

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

## Monitoring Stack

### Prometheus
- Metrics collection from all services
- Custom alerting rules via Alertmanager
- Service discovery for Kubernetes pods

### Grafana
- Pre-configured dashboards for Kabu Agent
- Data source: Prometheus

### Alertmanager
- Alert routing and notification
- Slack/Email integration available

## Service Mesh (Istio)

- Traffic management
- mTLS between services
- Observability via Kiali

## Troubleshooting

### Backend Not Accessible

Check NodePort:
```bash
kubectl -n kabu-agent get svc kabu-backend
# Look for port in 30000-32767 range under PORT(S)
```

### ArgoCD Password

```bash
# Get from EC2 server
ssh ubuntu@EIP
cat /home/ubuntu/argocd-password.txt
```

### Check Pod Status

```bash
kubectl -n kabu-agent get pods
kubectl -n kabu-agent describe pod <pod-name>
kubectl -n kabu-agent logs <pod-name>
```

### Monitoring Issues

```bash
# Check monitoring namespace
kubectl -n monitoring get pods
kubectl -n monitoring logs <prometheus-pod>
```

## License

MIT License
