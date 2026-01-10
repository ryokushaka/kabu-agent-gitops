# Kabu Agent GitOps リポジトリ

ArgoCDで管理されるKabu AgentのKubernetesマニフェストです。

## 構成

```
manifests/
├── argocd/           # ArgoCD Application & Project
├── kabu-agent/       # アプリケーションマニフェスト
│   ├── backend/      # FastAPI バックエンド
│   ├── frontend/     # React フロントエンド
│   ├── postgres/     # PostgreSQL データベース
│   └── redis/        # Redis キャッシュ
├── istio/            # Istio サービスメッシュ
├── kiali/            # Kiali ダッシュボード
├── monitoring/       # Prometheus & Grafana スタック
│   ├── alertmanager.yaml
│   ├── grafana-dashboards.yaml
│   ├── grafana-values.yaml
│   └── prometheus-values.yaml
└── tracing/          # 分散トレーシング (Jaeger)
```

## セットアップ

### 1. プレースホルダーの更新

使用前に、以下のプレースホルダーを置換してください:

| プレースホルダー | 置換内容 | ファイル |
|-----------------|---------|----------|
| `OWNER` | GitHubユーザー名 | `backend/deployment.yaml`, `argocd/*.yaml` |

**ryokushaka/kabu-agent-gitops用に設定済み**

### 2. ドメインなしでのアクセス

ドメインが設定されていないため、NodePort経由でサービスにアクセスします:

```bash
# バックエンドのNodePortを取得
kubectl -n kabu-agent get svc kabu-backend

# Elastic IP:NodePort経由でアクセス
# 例: http://106.73.5.224:30080
```

サービス:
- **バックエンド**: `EIP:30080`
- **フロントエンド**: `EIP:30000`
- **ArgoCD**: `https://EIP:30443` (EC2からArgoCDパスワードを使用)
- **Grafana**: `EIP:30300`
- **Prometheus**: `EIP:30090`
- **Kiali**: `EIP:30201`

### 3. Sealed Secretsの作成 (K3sデプロイ後)

```bash
# K3sサーバー上で (sealed-secretsコントローラーが稼働している場所)

# 1. secrets-template.yamlをコピーして実際の値を入力
cp manifests/kabu-agent/secrets-template.yaml.example /tmp/secrets.yaml
# /tmp/secrets.yamlを実際の値で編集

# 2. シークレットをシール
kubeseal --format yaml < /tmp/secrets.yaml > manifests/kabu-agent/sealed-secrets.yaml

# 3. 平文シークレットを削除
rm /tmp/secrets.yaml

# 4. シールドシークレットをコミット
git add manifests/kabu-agent/sealed-secrets.yaml
git commit -m "chore: add sealed secrets"
git push
```

### 4. ArgoCDへの登録

```bash
# ArgoCDアプリケーションを適用
kubectl apply -f manifests/argocd/project.yaml
kubectl apply -f manifests/argocd/application.yaml
```

## CI/CDフロー

```
kabu-agent リポ          GitOps リポ              K3s クラスター
     │                        │                        │
     │ mainにpush             │                        │
     ├───────────────────────►│                        │
     │                        │                        │
     │ build-push.yml         │                        │
     │ (Dockerビルド&プッシュ) │                        │
     │                        │                        │
     │ repository_dispatch    │                        │
     ├───────────────────────►│                        │
     │                        │                        │
     │                        │ update-manifest.yml    │
     │                        │ (イメージタグ更新)      │
     │                        │                        │
     │                        │ ArgoCD変更検知         │
     │                        ├───────────────────────►│
     │                        │                        │
     │                        │                        │ デプロイ
```

## 手動同期

```bash
# ArgoCD CLI経由で強制同期
argocd app sync kabu-agent

# またはkubectl経由
kubectl -n argocd patch application kabu-agent \
  --type merge \
  -p '{"operation": {"initiatedBy": {"username": "admin"}, "sync": {}}}'
```

## モニタリングスタック

### Prometheus
- 全サービスからのメトリクス収集
- Alertmanager経由のカスタムアラートルール
- Kubernetes Podのサービスディスカバリー

### Grafana
- Kabu Agent用の事前設定済みダッシュボード
- データソース: Prometheus

### Alertmanager
- アラートルーティングと通知
- Slack/Email連携対応

## サービスメッシュ (Istio)

- トラフィック管理
- サービス間mTLS
- Kiali経由の可観測性

## トラブルシューティング

### バックエンドにアクセスできない

NodePortを確認:
```bash
kubectl -n kabu-agent get svc kabu-backend
# PORT(S)で30000-32767範囲のポートを確認
```

### ArgoCDパスワード

```bash
# EC2サーバーから取得
ssh ubuntu@EIP
cat /home/ubuntu/argocd-password.txt
```

### Pod状態の確認

```bash
kubectl -n kabu-agent get pods
kubectl -n kabu-agent describe pod <pod-name>
kubectl -n kabu-agent logs <pod-name>
```

### モニタリングの問題

```bash
# monitoringネームスペースを確認
kubectl -n monitoring get pods
kubectl -n monitoring logs <prometheus-pod>
```

## ライセンス

MIT License
