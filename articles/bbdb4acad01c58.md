---
title: "TodoアプリをDocker + Cloud Run + Vercelでデプロイする"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# はじめに

今回は**コンテナに触れたことがない方**を対象に、Todo アプリをインターネットにデプロイするまでの手順を丁寧に説明します。

---

# ステップ1: GCPプロジェクトの準備

## 1.1 プロジェクト作成

1. [Google Cloud Console](https://console.cloud.google.com/) にアクセス
2. 「プロジェクトを作成」をクリック
3. 「コンソール」をクリック
   ![](https://storage.googleapis.com/zenn-user-upload/815ffb60d310-20260107.png)
4. 「新しいプロジェクト」をクリック
   ![](https://storage.googleapis.com/zenn-user-upload/1c410b524036-20260107.png)
5. プロジェクト名: `todo-app`
6. プロジェクト ID をメモ: `todo-app-xxxxx`

![](https://storage.googleapis.com/zenn-user-upload/8096e1afc24e-20260114.png)

## 1.2 APIを有効化

以下の API を有効化（各リンクをクリックして「有効にする」）:

- [Cloud Run API](https://console.cloud.google.com/apis/library/run.googleapis.com)
  ![](https://storage.googleapis.com/zenn-user-upload/8181033cfd74-20260114.png)

- [Artifact Registry API](https://console.cloud.google.com/apis/library/artifactregistry.googleapis.com)
  ![](https://storage.googleapis.com/zenn-user-upload/98f2f4859060-20260114.png)

- [Cloud SQL API](https://console.cloud.google.com/apis/library/sqladmin.googleapis.com)

## 1.3 Artifact Registryリポジトリ作成

1. [Artifact Registry](https://console.cloud.google.com/artifacts) を開く
2. 「リポジトリを作成」をクリック
3. 設定:
   - 名前: `todo-api`
   - 形式: `Docker`
   - リージョン: `asia-northeast1`

   ![](https://storage.googleapis.com/zenn-user-upload/646c3b1d2722-20260114.png)

4. 「作成」をクリック

![](https://storage.googleapis.com/zenn-user-upload/34629701ce99-20260114.png)

---

# ステップ2: Neonでデータベースを作成

**Neon**は、無料で使えるサーバーレスPostgreSQLサービスです。Cloud SQLより安く、セットアップも簡単です。

## 2.1 Neonアカウント作成

1. [Neon](https://neon.tech) にアクセス
2. 「Sign Up」をクリック
3. GitHubアカウントでサインアップ（推奨）または、メールアドレスでサインアップ

## 2.2 プロジェクト作成

1. ダッシュボードで「Create a project」をクリック
2. 設定:
   - **Project name**: `todo-app`
   - **Region**: `AWS / Asia Pacific (Tokyo) ap-northeast-1`（東京リージョン推奨）
   - **PostgreSQL version**: `17`（最新版でOK）
3. 「Create Project」をクリック

![](https://storage.googleapis.com/zenn-user-upload/2d1aa8bb06b6-20260121.png)

## 2.3 接続文字列を取得

プロジェクト作成後、**Connection Details**が表示されます。

1. 「Connection string」タブを選択
   ![](https://storage.googleapis.com/zenn-user-upload/07255ab85397-20260121.png)

2. 接続文字列をコピー:

   ```
   postgresql://your-username:your-password@ep-xxx-xxx.ap-northeast-1.aws.neon.tech/tododb?sslmode=require
   ```

   ![](https://storage.googleapis.com/zenn-user-upload/317489ab2cb5-20260121.png)

3. この接続文字列を**安全な場所にメモ**（後でGitHub Secretsに登録します）

## 2.4 データベースの確認（オプション）

Neonダッシュボードの「SQL Editor」で接続を確認できます:

```sql
-- テーブル一覧を表示
\dt

-- データベース情報を表示
SELECT version();
```

![](https://storage.googleapis.com/zenn-user-upload/0d37b08b34d1-20260121.png)

## 2.5 Prismaマイグレーションについて

現時点ではNeonにデータベースを作成しただけで、**テーブルは作成されていません**。

Todoアプリで必要なテーブル（`Todo`テーブルなど）は、**GitHub Actionsが自動的に作成**します。
具体的には、ステップ8で作成するバックエンド用ワークフローに、以下の処理が含まれています：

```yaml
- name: Run database migrations
  run: |
    npx prisma migrate deploy
```

このコマンドが、Prisma のスキーマ定義に基づいてテーブルを自動作成します。

---

# ステップ3: Dockerイメージをビルドしてプッシュ

**前提条件**: gcloud CLI がインストールされていること（[インストール手順](https://cloud.google.com/sdk/docs/install)）

## 3.1 gcloud認証とプロジェクト設定

```bash
# gcloudにログイン
gcloud auth login
```

![](https://storage.googleapis.com/zenn-user-upload/efcd8096dab3-20260117.png)
![](https://storage.googleapis.com/zenn-user-upload/73bec93b052a-20260117.png)
![](https://storage.googleapis.com/zenn-user-upload/b5fe4e8e5311-20260117.png)

```bash
# プロジェクトを設定（プロジェクトIDを置き換えてください）
gcloud config set project todo-app-xxxxx

# Docker認証を設定
gcloud auth configure-docker asia-northeast1-docker.pkg.dev
```

## 3.2 Dockerイメージをビルド

```bash
# プロジェクトルートに移動
cd /path/to/todo-docker-app

# todo-apiディレクトリに移動
cd todo-api

# Dockerイメージをビルド（プロジェクトIDを置き換えてください）
docker build -t asia-northeast1-docker.pkg.dev/todo-app-xxxxx/todo-api/todo-api:latest .
```

**注意**: ビルド中に`DATABASE_URL`環境変数が必要というエラーが出る場合がありますが、Dockerfile でダミー値が設定されているので正常にビルドされます。

## 3.3 Artifact Registryにプッシュ

```bash
# イメージをプッシュ（プロジェクトIDを置き換えてください）
docker push asia-northeast1-docker.pkg.dev/todo-app-xxxxx/todo-api/todo-api:latest
```

## 3.4 プッシュ確認

1. [Artifact Registry](https://console.cloud.google.com/artifacts) を開く
2. `todo-api` リポジトリをクリック
   ![](https://storage.googleapis.com/zenn-user-upload/6aab329b174a-20260118.png)
3. `todo-api` イメージが表示されていれば OK
   ![](https://storage.googleapis.com/zenn-user-upload/82447551acfd-20260118.png)

---

# ステップ4: Cloud Runサービス作成

1. [Cloud Run](https://console.cloud.google.com/run) を開く
2. 「サービスを作成」をクリック
   ![](https://storage.googleapis.com/zenn-user-upload/cdfb248e32c9-20260114.png)
3. 設定:
   - サービス名: `todo-api`
   - リージョン: `asia-northeast1`
   - コンテナイメージの URL: `asia-northeast1-docker.pkg.dev/todo-app-xxxxx/todo-api/todo-api:latest`
   - 認証: `公開アクセスを許可する`
     ![](https://storage.googleapis.com/zenn-user-upload/c67578b4f9e2-20260114.png)

4. 「コンテナ、ネットワーキング、セキュリティ」→「コンテナ」:
   - コンテナポート: `8080`
   - メモリ: `512 MiB`
   - CPU: `1`
   <!-- 5. 「環境変数」:
   - `DATABASE_URL` = `postgresql://ユーザー名:パスワード@/データベース名?host=/cloudsql/プロジェクトID:リージョン:インスタンス名` -->
     ![](https://storage.googleapis.com/zenn-user-upload/984e62def876-20260114.png)
5. 「作成」をクリック
6. サービス URL をメモ: `https://todo-api-xxxxx-an.a.run.app`

完了後以下の画面が出れば成功

![](https://storage.googleapis.com/zenn-user-upload/e0efc21f214c-20260114.png)

## 4.1 APIにアクセスして確認

```bash
curl https://todo-api-xxxxx-an.a.run.app
```

レスポンスが返ってくれば OK！

```json
{
  "message": "Hello from Hono!"
}
```

特定のエンドポイントを確認:

```bash
curl https://todo-api-xxxxx-an.a.run.app/api/todos
```

---

# ステップ5: サービスアカウント作成

1. [サービスアカウント](https://console.cloud.google.com/iam-admin/serviceaccounts) を開く
2. 「サービスアカウントを作成」をクリック
   ![](https://storage.googleapis.com/zenn-user-upload/727bd44f908b-20260114.png)
3. サービスアカウント名: `github-actions`
   ![](https://storage.googleapis.com/zenn-user-upload/f8ea516c8ebb-20260114.png)
4. 権限を追加:
   - `Cloud Run 管理者`
   - `Artifact Registry 書き込み`
   - `サービス アカウント ユーザー`
     ![](https://storage.googleapis.com/zenn-user-upload/b988919a6ffe-20260114.png)
5. 「完了」をクリック
6. 作成したサービスアカウントをクリック → 「キー」タブ
   ![](https://storage.googleapis.com/zenn-user-upload/7a4e4cfacef9-20260114.png)
7. 「鍵を追加」→「新しい鍵を作成」→「JSON」
   ![](https://storage.googleapis.com/zenn-user-upload/7a4e4cfacef9-20260114.png)
   ![](https://storage.googleapis.com/zenn-user-upload/d5825cfbe4ef-20260114.png)
   ![](https://storage.googleapis.com/zenn-user-upload/9533adf124b4-20260114.png)
   ![](https://storage.googleapis.com/zenn-user-upload/de2a8099acd8-20260114.png)
8. ダウンロードされた JSON ファイルの内容をコピー（後で使う）
   ![](https://storage.googleapis.com/zenn-user-upload/bef1c2edd7f4-20260114.png)

---

# ステップ6: Vercelでフロントエンドをデプロイ

React のアプリはすでに作成済みのところからスタートします。
今回は Vercel を使用します。

## 6.1 Vercelアカウント作成

**Vercel**は、フロントエンドアプリを簡単にデプロイできるサービスです。

1. [Vercel](https://vercel.com) にアクセス
2. "Sign Up" をクリック
3. GitHub アカウントで連携してサインアップ

## 6.2 プロジェクトをインポート

1. Vercel ダッシュボードで「Add New...」→「Project」をクリック

![](https://storage.googleapis.com/zenn-user-upload/8a3a8bdbc450-20260104.png)

2. GitHub リポジトリ一覧から `todo-docker-app` (該当リポジトリー)を選択

3. 「Import」をクリック

![](https://storage.googleapis.com/zenn-user-upload/071d2e28e9db-20260104.png)

## 6.3 ビルド設定

Vercel が自動的に検出しますが、念のため確認：

```
Framework Preset: Vite
Build Command: npm run build
Output Directory: dist
Install Command: npm install
```

**「Deploy」をクリック**

![](https://storage.googleapis.com/zenn-user-upload/0a81d19aec6c-20260104.png)

## 6.4 デプロイ完了

数分後、デプロイが完了します：

```
✓ Your project has been deployed!

URL: https://todo-docker-app-xxxxx.vercel.app
```

ブラウザで URL を開いて、アプリが表示されることを確認しましょう。

![](https://storage.googleapis.com/zenn-user-upload/7e34edc8d0b4-20260104.png)

## 6.5 環境変数を設定

**重要**: 現時点では、フロントエンドが開発環境（localhost）のAPIにアクセスしようとするため、正しく動作しません。

本番環境のCloud Run APIに接続できるように、環境変数を設定します。

### 環境変数の追加

1. Vercel Dashboard → プロジェクト `todo-docker-app-tutorial` → **Settings** → **Environment Variables**
   ![](https://storage.googleapis.com/zenn-user-upload/adb16dc1311a-20260123.png)
2. 以下を追加:
   - **Key**: `VITE_API_URL`
   - **Value**: `https://todo-api-xxxxx-an.a.run.app`（**ステップ4.1で取得したCloud Run URL**）
   - **Environments**: `Production`, `Preview`, `Development` すべてにチェック
3. **Save** をクリック
   ![](https://storage.googleapis.com/zenn-user-upload/34607a241dc2-20260123.png)

### 再デプロイ

環境変数を追加しただけでは反映されないため、再デプロイが必要です。

**Vercel Dashboardから再デプロイ**

1. **Deployments** タブを開く
2. 最新のデプロイの右側にある「...」メニューをクリック
   ![](https://storage.googleapis.com/zenn-user-upload/b173e279233a-20260123.png)

3. **Redeploy** をクリック
   ![](https://storage.googleapis.com/zenn-user-upload/d2d8ef243dc3-20260123.png)

### 動作確認

再デプロイ完了後、以下を確認します：

1. https://todo-docker-app-xxxxx.vercel.app を開く
2. ブラウザの開発者ツール（F12キー）を開く
3. **Network** タブを選択
4. TODOを追加してみる
5. `https://todo-api-xxxxx-an.a.run.app/todos` へのリクエストが表示され、ステータスコード `200 OK` が返ってくればOK

**成功例:**

```
POST https://todo-api-xxxxx-an.a.run.app/todos  200 OK
```

**失敗例（環境変数が未設定の場合）:**

```
POST http://localhost:3000/todos  net::ERR_FAILED
```

---

# ステップ7: GitHub Secretsを設定

1. GitHub リポジトリ → Settings → Secrets and variables → Actions
2. 「New repository secret」をクリック
3. 以下を 1 つずつ登録:

## GCP関連

| Name                      | Value                                        |
| ------------------------- | -------------------------------------------- |
| `GCP_PROJECT_ID`          | `todo-app-xxxxx`（プロジェクト ID）          |
| `GCP_SERVICE_ACCOUNT_KEY` | JSON ファイルの内容全体                      |
| `GCP_REGION`              | `asia-northeast1`                            |
| `CLOUD_RUN_SERVICE_NAME`  | `todo-api`                                   |
| `DATABASE_URL`            | `postgresql://...`（Neon 接続文字列）        |
| `CORS_ORIGIN`             | `https://your-app.vercel.app`（VercelのURL） |

## Vercel関連

| Name                | Value           |
| ------------------- | --------------- |
| `VERCEL_TOKEN`      | Vercel トークン |
| `VERCEL_ORG_ID`     | Organization ID |
| `VERCEL_PROJECT_ID` | Project ID      |

---

# ステップ8: GitHub Actionsワークフローを作成

## 8.1 フロントエンド用

`.github/workflows/frontend-deploy.yml` を作成:

```yaml
name: Deploy Frontend to Vercel

on:
  push:
    branches:
      - main
    paths:
      - "src/**"
      - "public/**"
      - "index.html"
      - "package.json"
      - "vite.config.ts"

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci
      - run: npm run build
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: "--prod"
```

## 8.2 バックエンド用

`.github/workflows/backend-deploy.yml` を作成:

```yaml
name: Deploy Backend to Cloud Run

on:
  push:
    branches:
      - main
    paths:
      - "todo-api/**"

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  REGION: ${{ secrets.GCP_REGION }}
  SERVICE_NAME: ${{ secrets.CLOUD_RUN_SERVICE_NAME }}
  REPOSITORY: todo-api
  IMAGE_NAME: todo-api

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: google-github-actions/auth@v2
        with:
          credentials_json: "${{ secrets.GCP_SERVICE_ACCOUNT_KEY }}"

      - uses: google-github-actions/setup-gcloud@v2

      - run: gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev

      - name: Build and Push
        working-directory: ./todo-api
        run: |
          docker build -t ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} .
          docker push ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy ${{ env.SERVICE_NAME }} \
            --image ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            --region ${{ env.REGION }} \
            --platform managed \
            --allow-unauthenticated \
            --set-env-vars "DATABASE_URL=${{ secrets.DATABASE_URL }},CORS_ORIGIN=${{ secrets.CORS_ORIGIN }}" \
            --port 8080

      - name: Run database migrations
        working-directory: ./todo-api
        run: |
          npm ci
          npx prisma migrate deploy
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

### ワークフローの説明

このワークフローは、`todo-api/**` 配下のファイルが変更されると自動的に実行されます。

**主要な処理の流れ:**

1. **GCP認証** (`google-github-actions/auth@v2`)
   - GitHub Secretsに登録したサービスアカウントキーで認証

2. **Docker認証設定** (`gcloud auth configure-docker`)
   - DockerクライアントがArtifact Registryにアクセスできるように設定

3. **Dockerイメージのビルドとプッシュ** (`Build and Push`)
   - `todo-api/Dockerfile`からイメージをビルド
   - Artifact Registryにプッシュ
   - イメージタグには`${{ github.sha }}`（コミットハッシュ）を使用

4. **Cloud Runにデプロイ** (`Deploy to Cloud Run`)
   - プッシュしたイメージをCloud Runにデプロイ
   - 環境変数`DATABASE_URL`を設定

5. **データベースマイグレーション** (`Run database migrations`)
   - `npx prisma migrate deploy`でテーブルを作成・更新
   - ここで初めてNeonデータベースにテーブルが作成されます

**重要**: このワークフローにより、コードをpushするだけで、ビルド→デプロイ→マイグレーションが自動的に実行されます。

---

# 完成

これで Todo アプリのデプロイが完了しました。

## 構成のおさらい

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│     Vercel      │────▶│   Cloud Run     │────▶│      Neon       │
│  (Frontend)     │     │   (Backend)     │     │  (PostgreSQL)   │
│  React + Vite   │     │   Hono API      │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        ▲                       ▲
        │                       │
        └───────────────────────┘
              GitHub Actions
           (自動ビルド・デプロイ)
```

# おわりに

この記事では、コンテナ初心者の方向けに、Todo アプリを本番環境にデプロイする手順を解説しました。

Docker、Cloud Run、Vercel、GitHub Actions といった技術を組み合わせることで、モダンな CI/CD パイプラインを構築できました。

---
