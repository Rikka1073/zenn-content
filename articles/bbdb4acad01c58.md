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

![](https://storage.googleapis.com/zenn-user-upload/8096e1afc24e-20260114.png)

### 1.2 API を有効化

以下の API を有効化（各リンクをクリックして「有効にする」）:

- [Cloud Run API](https://console.cloud.google.com/apis/library/run.googleapis.com)
  ![](https://storage.googleapis.com/zenn-user-upload/8181033cfd74-20260114.png)

- [Artifact Registry API](https://console.cloud.google.com/apis/library/artifactregistry.googleapis.com)
  ![](https://storage.googleapis.com/zenn-user-upload/98f2f4859060-20260114.png)

- [Cloud SQL API](https://console.cloud.google.com/apis/library/sqladmin.googleapis.com)

### 1.3 Artifact Registry リポジトリ作成

1. [Artifact Registry](https://console.cloud.google.com/artifacts) を開く
2. 「リポジトリを作成」をクリック
3. 設定:

   - 名前: `todo-api`
   - 形式: `Docker`
   - リージョン: `asia-northeast1`

   ![](https://storage.googleapis.com/zenn-user-upload/646c3b1d2722-20260114.png)

4. 「作成」をクリック

![](https://storage.googleapis.com/zenn-user-upload/34629701ce99-20260114.png)

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
