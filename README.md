# 4-3-k8sapp

ローカルKubernetes（minikube）でFastAPI + MySQLを動かす

---

## 目的

k8sって何？を学ぶ
ローカルでk8sを動かして
コンテナオーケストレーションの基礎を体験する

---

## 構成図

minikube（ローカルk8sクラスター）
└─ Namespace: 4-3-k8sapp
├─ Deployment: api（FastAPI）
│   └─ Pod x1
├─ Service: api-service（NodePort: 30080）
├─ Deployment: db（MySQL）
│   └─ Pod x1（replicas=1）
├─ Service: db-service（ClusterIP: 3306）
├─ ConfigMap: db-config
└─ Secret: db-secret

---

## ECS vs EC2 vs k8s の比較

| 項目 | ECS | EC2 | k8s |
|------|-----|-----|-----|
| 実行環境 | AWS | AWS | どこでも |
| スケールアップ | AWSコンソール/Terraform | 大変 | 1コマンド |
| デプロイ | Dockerビルド→ECR→ECS | SSH→ファイルコピー | kubectl apply |
| ポータビリティ | AWSのみ | AWSのみ | AWS/GCP/Azure/ローカル |
| 学習コスト | 中 | 低 | 高 |

---

## 使用技術

### アプリケーション
| 技術 | 用途 |
|------|------|
| Python + FastAPI | REST APIサーバー |
| SQLAlchemy | DBの操作（ORM） |
| MySQL 8.0 | データベース |

### インフラ
| ツール | 用途 |
|--------|------|
| minikube | ローカルk8sクラスター |
| kubectl | k8sの操作コマンド |

---

## k8sの基本概念

### Pod

最小のデプロイ単位
└─ 1つ以上のコンテナを含む
└─ ECSのタスクに相当

### Deployment

Podを管理するリソース
└─ レプリカ数・イメージを管理
└─ ECSのサービスに相当

### Service

Podへのネットワークアクセスを管理
種類
├─ ClusterIP → クラスター内のみ（DB用）
└─ NodePort  → 外部からアクセス可能（API用）

### ConfigMap

機密でない設定値を管理
└─ DB名・ユーザー名など
└─ AWSのParameter Storeに相当

### Secret

機密情報を管理
└─ パスワード・APIキーなど
└─ AWSのSecrets Managerに相当


### Namespace

リソースを論理的に分離する仕組み
└─ 例：開発環境・本番環境を分ける

---

## ファイル構成

4-3-k8sapp/
├── backend/
│   ├── app/
│   │   ├── main.py          → アプリの入口・テーブル自動作成
│   │   ├── database.py      → DB接続設定
│   │   ├── models.py        → DBテーブル定義
│   │   ├── schemas.py       → APIの入出力
│   │   └── routers/
│   │       └── items.py     → 商品APIエンドポイント
│   ├── Dockerfile
│   └── requirements.txt
└── k8s/
├── namespace.yaml        → Namespace定義
├── db-config.yaml        → ConfigMap（DB設定）
├── db-secret.yaml        → Secret（DBパスワード）
├── db-deployment.yaml    → MySQL Deployment
├── db-service.yaml       → MySQL Service（ClusterIP）
├── api-deployment.yaml   → FastAPI Deployment
└── api-service.yaml      → FastAPI Service（NodePort）

---

## 環境の構築手順

### Step1: minikubeをインストール

```bash
brew install minikube
brew install kubectl
```

### Step2: minikubeを起動

```bash
minikube start
```

### Step3: minikubeのDockerを使う

```bash
eval $(minikube docker-env)
```

### Step4: Dockerイメージをビルド

```bash
docker build -t k8sapp-api:latest ./backend
```

### Step5: k8sリソースをデプロイ

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/db-config.yaml
kubectl apply -f k8s/db-secret.yaml
kubectl apply -f k8s/db-deployment.yaml
kubectl apply -f k8s/db-service.yaml
kubectl apply -f k8s/api-deployment.yaml
kubectl apply -f k8s/api-service.yaml
```

### Step6: 起動確認

```bash
kubectl get all -n 4-3-k8sapp
```

### Step7: アクセス

```bash
minikube service api-service -n 4-3-k8sapp
```

ブラウザで以下にアクセス：
- `/health` → ヘルスチェック
- `/api/items/` → 商品一覧
- `/docs` → Swagger UI

---

## よく使うkubectlコマンド

```bash
# リソース一覧
kubectl get all -n 4-3-k8sapp

# Podのログ確認
kubectl logs -n 4-3-k8sapp -l app=api

# Podの詳細確認
kubectl describe pod -n 4-3-k8sapp -l app=api

# スケールアップ
kubectl scale deployment api -n 4-3-k8sapp --replicas=3

# スケールダウン
kubectl scale deployment api -n 4-3-k8sapp --replicas=1

# Podに入る
kubectl exec -it -n 4-3-k8sapp <pod-name> -- /bin/bash

# リソースを削除
kubectl delete -f k8s/

# Namespaceごと削除
kubectl delete namespace 4-3-k8sapp
```

---

## 動作確認

```bash
# ヘルスチェック
curl http://127.0.0.1:<PORT>/health

# 商品作成
curl -X POST http://127.0.0.1:<PORT>/api/items/ \
  -H "Content-Type: application/json" \
  -d '{"name": "テスト商品", "description": "説明", "price": 1000}'

# 商品一覧
curl http://127.0.0.1:<PORT>/api/items/
```

---

## トラブルシューティング

### PodがCrashLoopBackOffになる場合

```bash
# ログを確認
kubectl logs -n 4-3-k8sapp <pod-name>

# DBの起動を待ってから再起動
kubectl rollout restart deployment api -n 4-3-k8sapp
```

### イメージが見つからない場合

```bash
# minikubeのDockerを使う設定を忘れずに
eval $(minikube docker-env)

# 再ビルド
docker build -t k8sapp-api:latest ./backend
```

### minikubeのサービスにアクセスできない場合

```bash
# トンネルを使う
minikube service api-service -n 4-3-k8sapp

# または別ターミナルでポートフォワード
kubectl port-forward -n 4-3-k8sapp svc/api-service 8080:80
```

---

## 環境の削除手順

```bash
# k8sリソースを削除
kubectl delete namespace 4-3-k8sapp

# minikubeを停止
minikube stop

# minikubeを削除（完全リセット）
minikube delete
```

