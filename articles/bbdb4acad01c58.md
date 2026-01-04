---
title: ""
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# はじめに

今回は**コンテナに触れたことがない方**を対象に、Todo アプリをインターネットにデプロイするまでの手順を丁寧に説明します。

# React アプリをデプロイする

React のアプリはすでに作成済みのところからスタートします。
今回は Vercel を使用します。

## Vercel とは

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
