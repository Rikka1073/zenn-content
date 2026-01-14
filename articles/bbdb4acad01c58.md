---
title: "はじめてのデプロイガイド - コンテナ入門からCI/CD自動化まで"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# はじめに

今回は**コンテナに触れたことがない方**を対象に、Todo アプリをインターネットにデプロイするまでの手順を丁寧に説明します。

## React アプリをデプロイする

React のアプリはすでに作成済みのところからスタートします。
今回は Vercel を使用します。

### Vercel とは

**Vercel**は、フロントエンドアプリを簡単にデプロイできるサービスです。

### 3.1 Vercel アカウント作成

1. [Vercel](https://vercel.com) にアクセス
2. "Sign Up" をクリック
3. GitHub アカウントで連携してサインアップ

### 3.2 プロジェクトをインポート

1. Vercel ダッシュボードで「Add New...」→「Project」をクリック

![](https://storage.googleapis.com/zenn-user-upload/8a3a8bdbc450-20260104.png)

2. GitHub リポジトリ一覧から `todo-docker-app` (該当リポジトリー)を選択

3. 「Import」をクリック

![](https://storage.googleapis.com/zenn-user-upload/071d2e28e9db-20260104.png)

### 3.3 ビルド設定

Vercel が自動的に検出しますが、念のため確認：

```
Framework Preset: Vite
Build Command: npm run build
Output Directory: dist
Install Command: npm install
```

**「Deploy」をクリック**

![](https://storage.googleapis.com/zenn-user-upload/0a81d19aec6c-20260104.png)

### 3.4 デプロイ完了

数分後、デプロイが完了します：

```
✓ Your project has been deployed!

URL: https://todo-docker-app-xxxxx.vercel.app
```

ブラウザで URL を開いて、アプリが表示されることを確認しましょう。

![](https://storage.googleapis.com/zenn-user-upload/7e34edc8d0b4-20260104.png)

### Cloud Run とは

**Cloud Run** は、Google が提供するコンテナ実行サービスです。

### 4.1 前提：GCP プロジェクトの準備

#### Google Cloud Platform（GCP）アカウント作成

1. [Google Cloud Console](https://console.cloud.google.com/) にアクセス
2. Google アカウントでログイン
3. 「コンソール」をクリック
   ![](https://storage.googleapis.com/zenn-user-upload/815ffb60d310-20260107.png)
4. 「新しいプロジェクト」をクリック
   ![](https://storage.googleapis.com/zenn-user-upload/1c410b524036-20260107.png)
5. プロジェクト名: `todo-app`（任意）
6. プロジェクト ID をメモ: `todo-app-xxxxx`

#### 必要な API を有効化

Google Cloud Console で以下の API を有効化します：

1. **Cloud Run API**

   ```
   https://console.cloud.google.com/apis/library/run.googleapis.com
   ```

2. **Artifact Registry API**

   ```
   https://console.cloud.google.com/apis/library/artifactregistry.googleapis.com
   ```

3. **Cloud SQL API**（データベース用）
   ```
   https://console.cloud.google.com/apis/library/sqladmin.googleapis.com
   ```

各ページで「有効にする」をクリックしてください。

### 4.2 gcloud CLI のインストール

ローカル PC で GCP を操作するためのツールをインストールします。

#### macOS

```bash
brew install --cask google-cloud-sdk
```

#### Windows

1. [Google Cloud SDK](https://cloud.google.com/sdk/docs/install) からインストーラーをダウンロード
2. インストーラーを実行

#### 初期設定

```bash
# Google Cloud CLI 初期化
gcloud init

# 表示されるブラウザでログイン
# プロジェクトを選択: todo-app-xxxxx
# リージョンを選択: 30（asia-northeast1 - 東京）
```

### 4.3 Dockerfile の確認

バックエンド用の Dockerfile が既に存在します：

```bash
ls todo-api/Dockerfile
```

このファイルが、**Hono アプリをコンテナに詰める設計図**です。

#### Dockerfile の中身を見てみよう

```dockerfile
# Node.js 18のイメージを使用
FROM node:18-alpine AS builder

# 作業ディレクトリを設定
WORKDIR /app

# package.jsonをコピー
COPY package*.json ./

# ライブラリをインストール
RUN npm ci

# アプリのコードをコピー
COPY . .

# Prismaクライアントを生成
RUN npx prisma generate

# ビルド（本番用）
RUN npm run build

# 本番環境用のイメージ
FROM node:18-alpine

WORKDIR /app

# 本番用ライブラリのみインストール
COPY package*.json ./
RUN npm ci --production

# ビルド済みファイルをコピー
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma

# ポート8080で起動
EXPOSE 8080

# アプリを起動
CMD ["node", "dist/index.js"]
```

**これが何をしているか：**

1. Node.js 18 をベースにコンテナを作成
2. `package.json` をコピーして、ライブラリをインストール
3. アプリのコードをコピー
4. Prisma クライアントを生成（データベースアクセス用）
5. アプリをビルド
6. ポート 8080 で起動する準備
7. `node dist/index.js` でアプリを起動

### 4.4 Artifact Registry リポジトリを作成

**Artifact Registry** は、Docker イメージを GCP に保存する場所です。

#### なぜ必要？

```
あなたのパソコン:
Dockerイメージを作成
    ↓
Artifact Registry:
Dockerイメージを保存（GCPのストレージ）
    ↓
Cloud Run:
保存されたイメージを取得して起動
```

#### リポジトリ作成

```bash
gcloud artifacts repositories create todo-api \
  --repository-format=docker \
  --location=asia-northeast1 \
  --description="Docker repository for todo-api"
```

**コマンドの意味：**

- `todo-api`: リポジトリ名
- `--repository-format=docker`: Docker イメージを保存
- `--location=asia-northeast1`: 東京リージョン

#### 確認

```bash
gcloud artifacts repositories list --location=asia-northeast1
```

### 4.5 ローカルで Docker イメージをビルド

まず、ローカル PC で Docker イメージを作ってみましょう。

#### プロジェクト ID を環境変数に設定

```bash
# あなたのGCPプロジェクトIDに置き換えてください
export PROJECT_ID="todo-app-xxxxx"
```

#### Docker イメージをビルド

```bash
cd todo-api

docker build -t asia-northeast1-docker.pkg.dev/$PROJECT_ID/todo-api/todo-api:latest .
```

**コマンドの意味：**

- `docker build`: Docker イメージを作成
- `-t`: イメージに名前（タグ）を付ける
- `asia-northeast1-docker.pkg.dev/$PROJECT_ID/todo-api/todo-api:latest`:
  - `asia-northeast1-docker.pkg.dev`: Artifact Registry の URL
  - `$PROJECT_ID`: あなたの GCP プロジェクト ID
  - `todo-api`: リポジトリ名
  - `todo-api`: イメージ名
  - `latest`: タグ（最新版という意味）
- `.`: 現在のディレクトリの Dockerfile を使う

#### 完了を確認

```bash
docker images | grep todo-api
```

以下のように表示されれば OK：

```
asia-northeast1-docker.pkg.dev/todo-app-xxxxx/todo-api/todo-api   latest   abc123def456   2 minutes ago   200MB
```

### 4.6 Docker イメージを Artifact Registry にプッシュ

#### Docker 認証設定

```bash
gcloud auth configure-docker asia-northeast1-docker.pkg.dev
```

これで、Docker が Artifact Registry にアクセスできるようになります。

#### イメージをプッシュ

```bash
docker push asia-northeast1-docker.pkg.dev/$PROJECT_ID/todo-api/todo-api:latest
```

**何が起こっているか：**

```
あなたのパソコン:
Dockerイメージ（200MB）
    ↓ アップロード
Artifact Registry（GCP）:
保存完了
```

#### 確認

GCP Console で Artifact Registry を開く：

```
https://console.cloud.google.com/artifacts/docker/[PROJECT_ID]/asia-northeast1/todo-api
```

`todo-api` イメージが表示されていれば OK！

### 4.7 Cloud SQL（データベース）の準備

#### Cloud SQL インスタンスが既に存在する場合

既に Cloud SQL インスタンスがある場合は、そのまま使えます。

```bash
# インスタンス一覧を確認
gcloud sql instances list
```

#### データベース接続 URL

Cloud Run から接続するための`DATABASE_URL`は以下の形式です：

```
postgresql://ユーザー名:パスワード@/データベース名?host=/cloudsql/プロジェクトID:リージョン:インスタンス名
```

**例：**

```
postgresql://postgres:your-password@/tododb?host=/cloudsql/todo-app-xxxxx:asia-northeast1:todo-db
```

この値は後で GitHub Secrets に登録します。

### 4.8 Cloud Run にデプロイ

いよいよ、Cloud Run にデプロイします！

```bash
gcloud run deploy todo-api \
  --image asia-northeast1-docker.pkg.dev/$PROJECT_ID/todo-api/todo-api:latest \
  --region asia-northeast1 \
  --platform managed \
  --allow-unauthenticated \
  --set-env-vars "DATABASE_URL=postgresql://..." \
  --port 8080 \
  --memory 512Mi \
  --cpu 1 \
  --max-instances 10 \
  --min-instances 0
```

**コマンドの意味：**

| オプション                | 意味                                |
| ------------------------- | ----------------------------------- |
| `--image`                 | 使用する Docker イメージ            |
| `--region`                | デプロイするリージョン（東京）      |
| `--platform managed`      | フルマネージド（自動管理）          |
| `--allow-unauthenticated` | 認証なしでアクセス可能              |
| `--set-env-vars`          | 環境変数を設定（DATABASE_URL）      |
| `--port 8080`             | アプリが起動するポート              |
| `--memory 512Mi`          | メモリ 512MB 割り当て               |
| `--cpu 1`                 | CPU 1 個割り当て                    |
| `--max-instances 10`      | 最大 10 個まで自動スケール          |
| `--min-instances 0`       | アクセスがない時は 0 個（料金節約） |

#### デプロイ完了

数分後、以下のように表示されます：

```
Service [todo-api] revision [todo-api-00001-xxx] has been deployed
and is serving 100 percent of traffic.
Service URL: https://todo-api-xxxxx-an.a.run.app
```

### 4.9 動作確認

#### API にアクセス

```bash
curl https://todo-api-xxxxx-an.a.run.app
```

レスポンスが返ってくれば OK！

```json
{
  "message": "Hello from Hono!"
}
```

#### 特定のエンドポイントを確認

```bash
curl https://todo-api-xxxxx-an.a.run.app/api/todos
```

---

# ステップ 1: GCP プロジェクトの準備

### 1.1 プロジェクト作成

1. [Google Cloud Console](https://console.cloud.google.com/) にアクセス
2. 「プロジェクトを作成」をクリック
3. 「コンソール」をクリック
   ![](https://storage.googleapis.com/zenn-user-upload/815ffb60d310-20260107.png)
4. 「新しいプロジェクト」をクリック
   ![](https://storage.googleapis.com/zenn-user-upload/1c410b524036-20260107.png)
5. プロジェクト名: `todo-app`
6. プロジェクト ID をメモ: `todo-app-xxxxx`

### 1.2 API を有効化

以下の API を有効化（各リンクをクリックして「有効にする」）:

- [Cloud Run API](https://console.cloud.google.com/apis/library/run.googleapis.com)
- [Artifact Registry API](https://console.cloud.google.com/apis/library/artifactregistry.googleapis.com)
- [Cloud SQL API](https://console.cloud.google.com/apis/library/sqladmin.googleapis.com)

### 1.3 Artifact Registry リポジトリ作成

1. [Artifact Registry](https://console.cloud.google.com/artifacts) を開く
2. 「リポジトリを作成」をクリック
3. 設定:
   - 名前: `todo-api`
   - 形式: `Docker`
   - リージョン: `asia-northeast1`
4. 「作成」をクリック

### 1.4 Cloud Run サービス作成

1. [Cloud Run](https://console.cloud.google.com/run) を開く
2. 「サービスを作成」をクリック
3. 設定:
   - サービス名: `todo-api`
   - リージョン: `asia-northeast1`
   - コンテナイメージの URL: `us-docker.pkg.dev/cloudrun/container/hello`
   - 認証: `未認証の呼び出しを許可`
4. 「コンテナ、ネットワーキング、セキュリティ」→「コンテナ」:
   - コンテナポート: `8080`
   - メモリ: `512 MiB`
   - CPU: `1`
5. 「環境変数」:
   - `DATABASE_URL` = `postgresql://ユーザー名:パスワード@/データベース名?host=/cloudsql/プロジェクトID:リージョン:インスタンス名`
6. 「作成」をクリック
7. サービス URL をメモ: `https://todo-api-xxxxx-an.a.run.app`

### 1.5 サービスアカウント作成

1. [サービスアカウント](https://console.cloud.google.com/iam-admin/serviceaccounts) を開く
2. 「サービスアカウントを作成」をクリック
3. サービスアカウント名: `github-actions`
4. 権限を追加:
   - `Cloud Run 管理者`
   - `Artifact Registry 書き込み`
   - `サービス アカウント ユーザー`
5. 「完了」をクリック
6. 作成したサービスアカウントをクリック → 「キー」タブ
7. 「鍵を追加」→「新しい鍵を作成」→「JSON」
8. ダウンロードされた JSON ファイルの内容をコピー（後で使う）

---
