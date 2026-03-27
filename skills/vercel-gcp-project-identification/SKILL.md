---
name: vercel-gcp-project-identification
description: "AI Studioエクスポートアプリの正しいGCPプロジェクトを特定し、安全に操作する方法。gcloud、GCPプロジェクト、プロジェクトID、firebase-applet-config、取り違え防止、gen-lang-client に関するリクエストで自動発動。"
---

# GCPプロジェクト特定と安全な操作

## AI Studioエクスポートアプリのプロジェクト構造

AI Studioはユーザーごとに `gen-lang-client-XXXXXXXXXX` 形式のGCPプロジェクトを自動プロビジョニングする。
このプロジェクトは独立したGCPプロジェクトであり、以下が同一プロジェクト内に共存する:

```
[GCPプロジェクト] gen-lang-client-XXXXXXXXXX (表示名: "Gemini API")
 ├── Gemini API (generativelanguage.googleapis.com)
 │    └── APIキー: GEMINI_API_KEY（汎用APIキー）
 ├── Firebase
 │    ├── Webアプリ（appId）
 │    ├── Auth (identitytoolkit.googleapis.com)
 │    └── Firestore（名前付きDB: ai-studio-*）
 └── APIキー
      ├── 汎用キー → GEMINI_API_KEY として使用
      └── Browser key (auto created by Firebase) → firebase-applet-config.json の apiKey
```

### 重要な区別

| 項目 | 場所 | 用途 |
|---|---|---|
| `projectId` | firebase-applet-config.json | GCPプロジェクトID（= Firebaseプロジェクト） |
| `projectNumber` (= messagingSenderId) | firebase-applet-config.json | GCPプロジェクト番号（数字のみ） |
| `apiKey` | firebase-applet-config.json | Firebase SDK用ブラウザキー（公開前提） |
| `GEMINI_API_KEY` | 環境変数 | Gemini API用キー（非公開、サーバーサイドのみ） |

## プロジェクトIDの特定方法

### Step 1: firebase-applet-config.json から取得

```bash
# プロジェクトIDを取得
cat firebase-applet-config.json | jq -r .projectId
# → gen-lang-client-XXXXXXXXXX
```

これがGCPプロジェクトIDであり、Firebaseプロジェクトでもある。

### Step 2: gcloudで確認

```bash
# プロジェクトの詳細を確認（権限確認も兼ねる）
gcloud projects describe $(cat firebase-applet-config.json | jq -r .projectId)
```

## ⚠️ 取り違え防止ルール

### CRITICAL: gcloudデフォルトプロジェクトを信用しない

```bash
# これは現在のデフォルトプロジェクトを表示するだけ — 対象プロジェクトとは無関係
gcloud config get-value project
# → 例: my-other-project（全く別のプロジェクト）
```

**全てのgcloudコマンドに `--project=` を明示指定すること。**

```bash
# ❌ 危険: デフォルトプロジェクトに対して実行される
gcloud services list

# ✅ 安全: 対象プロジェクトを明示
gcloud services list --project=gen-lang-client-XXXXXXXXXX
```

### デフォルトプロジェクトを切り替えない

```bash
# ❌ 禁止: 他の作業に影響する
gcloud config set project gen-lang-client-XXXXXXXXXX

# ✅ 推奨: 毎回 --project= で指定
gcloud <command> --project=gen-lang-client-XXXXXXXXXX
```

### REST API使用時も同様

```bash
# URLにプロジェクトIDを含める
curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://firebase.googleapis.com/v1beta1/projects/gen-lang-client-XXXXXXXXXX"
```

## プロジェクトへのアクセス権確認

```bash
# プロジェクトの説明が取得できれば権限あり
gcloud projects describe gen-lang-client-XXXXXXXXXX

# 有効なAPIの確認
gcloud services list --project=gen-lang-client-XXXXXXXXXX --enabled

# IAMロールの確認
gcloud projects get-iam-policy gen-lang-client-XXXXXXXXXX
```

## CLIでの操作パターン

プロジェクトIDを変数に入れて使うと安全:

```bash
# プロジェクトIDを変数化
PROJECT_ID=$(cat firebase-applet-config.json | jq -r .projectId)

# 以降すべてのコマンドで使用
gcloud services list --project=$PROJECT_ID
gcloud firestore databases list --project=$PROJECT_ID
```
