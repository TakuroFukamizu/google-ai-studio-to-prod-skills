---
name: vercel-firebase-auth-domain
description: "デプロイ先URL変更時にFirebase Authの承認済みドメインを管理する方法。Firebase Auth、承認済みドメイン、authorized domains、Google Sign-In、signInWithPopup、ドメイン追加、認証エラー、auth/unauthorized-domain に関するリクエストで自動発動。"
---

# Firebase Auth 承認済みドメイン管理

## 概要

Firebase AuthのGoogle Sign-In（`signInWithPopup`）を使用するアプリのデプロイ先URLが変わった場合、新しいドメインをFirebase Authの承認済みドメインに追加しないと認証が失敗する。

### よくあるエラー

- `auth/unauthorized-domain` エラー
- Google Sign-Inポップアップが開いた後にエラーで戻る
- 「This domain is not authorized」メッセージ

## 前提条件

- gcloud CLIがセットアップ済み
- 対象GCPプロジェクトへのアクセス権がある
- **GCPプロジェクトIDの特定には `vercel-gcp-project-identification` スキルを参照**

## 現在の承認済みドメインを確認する

### REST API経由（gcloud認証を利用）

```bash
PROJECT_ID=$(cat firebase-applet-config.json | jq -r .projectId)

curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://identitytoolkit.googleapis.com/admin/v2/projects/${PROJECT_ID}/config" \
  | jq '.authorizedDomains'
```

出力例:
```json
[
  "gen-lang-client-XXXXXXXXXX.firebaseapp.com",
  "gen-lang-client-XXXXXXXXXX.web.app",
  "your-app-name.asia-northeast1.run.app"
]
```

## 承認済みドメインを追加する

### Step 1: 現在のドメイン一覧を取得

```bash
PROJECT_ID=$(cat firebase-applet-config.json | jq -r .projectId)

CURRENT_DOMAINS=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://identitytoolkit.googleapis.com/admin/v2/projects/${PROJECT_ID}/config" \
  | jq -r '.authorizedDomains')

echo "現在の承認済みドメイン:"
echo "$CURRENT_DOMAINS" | jq .
```

### Step 2: 新しいドメインを追加してPATCH

```bash
PROJECT_ID=$(cat firebase-applet-config.json | jq -r .projectId)
NEW_DOMAIN="your-project.vercel.app"  # 追加したいドメイン

# 現在のドメイン一覧を取得
CURRENT_DOMAINS=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://identitytoolkit.googleapis.com/admin/v2/projects/${PROJECT_ID}/config" \
  | jq '.authorizedDomains')

# 新ドメインを追加した配列を作成
UPDATED_DOMAINS=$(echo "$CURRENT_DOMAINS" | jq --arg d "$NEW_DOMAIN" '. + [$d] | unique')

# PATCH リクエストで更新
curl -s -X PATCH \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d "{\"authorizedDomains\": $UPDATED_DOMAINS}" \
  "https://identitytoolkit.googleapis.com/admin/v2/projects/${PROJECT_ID}/config?updateMask=authorizedDomains" \
  | jq '.authorizedDomains'
```

### Step 3: 追加されたことを確認

```bash
curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://identitytoolkit.googleapis.com/admin/v2/projects/${PROJECT_ID}/config" \
  | jq '.authorizedDomains'
```

## ワンライナー版（自動化向け）

```bash
# 変数設定
PROJECT_ID=$(cat firebase-applet-config.json | jq -r .projectId)
NEW_DOMAIN="your-app.vercel.app"
TOKEN=$(gcloud auth print-access-token)
API_URL="https://identitytoolkit.googleapis.com/admin/v2/projects/${PROJECT_ID}/config"

# 現在のドメイン取得 → 追加 → PATCH を一発で
UPDATED=$(curl -s -H "Authorization: Bearer $TOKEN" "$API_URL" | jq --arg d "$NEW_DOMAIN" '.authorizedDomains + [$d] | unique')
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{\"authorizedDomains\": $UPDATED}" "${API_URL}?updateMask=authorizedDomains" | jq '.authorizedDomains'
```

## 注意事項

### Firebase ConsoleのGUI操作でも可能

Firebase Console → Authentication → Settings → 承認済みドメイン → ドメインを追加

ただし、CLIで自動化できる方が再現性が高く、Skills化に適している。

### 追加すべきドメインのパターン

| デプロイ先 | 追加するドメイン例 |
|---|---|
| Vercel | `your-project.vercel.app` |
| Vercel（カスタムドメイン） | `your-domain.com` |
| Cloud Run | `your-service-xxxx.region.run.app` |
| localhost（開発時） | `localhost`（デフォルトで含まれている場合が多い） |

### Identity Platform API について

Firebase AuthはGCPの Identity Platform (`identitytoolkit.googleapis.com`) の上に構築されている。
`gcloud identity-platform` サブコマンドが使えない場合は、REST APIを直接使用する（上記の方法）。

### 403エラーが出る場合（quota project）

REST APIで403エラーが出る場合、quota project（課金先プロジェクト）の指定が必要。

```bash
# 方法1: x-goog-user-project ヘッダーを追加
curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "x-goog-user-project: ${PROJECT_ID}" \
  "https://identitytoolkit.googleapis.com/admin/v2/projects/${PROJECT_ID}/config"

# 方法2: gcloud の billing-project を一時設定
gcloud auth application-default set-quota-project ${PROJECT_ID}
```

**AI Studioが自動プロビジョニングしたプロジェクトでは、gcloudのデフォルトquota projectと異なるためこのエラーが発生しやすい。**

### `authDomain` との関係

`firebase-applet-config.json` の `authDomain`（例: `gen-lang-client-XXXXXXXXXX.firebaseapp.com`）は、OAuthリダイレクト先として使われるFirebase Hostingドメイン。承認済みドメインとは別の概念:

- **`authDomain`**: OAuthフロー中のリダイレクト先（通常変更不要）
- **承認済みドメイン**: `signInWithPopup`/`signInWithRedirect` を呼び出せるドメイン（デプロイ先を追加する）
