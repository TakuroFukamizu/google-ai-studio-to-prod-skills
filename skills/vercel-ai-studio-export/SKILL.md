---
name: vercel-ai-studio-export
description: "Google AI StudioからエクスポートされたアプレットをVercel等の別環境にデプロイ・移行する際の知見スキル。AI Studio、アプレット、applet、エクスポート、Google AI Studio、移行、マイグレーション、firebase-applet-config、Cloud Notepadに関するリクエストで自動発動。"
---

# Google AI Studio エクスポート → Vercel 移行ガイド

## AI Studioエクスポートの構成

AI Studioからエクスポートされたプロジェクトには以下のファイルが含まれる。

| ファイル | 内容 | デプロイ先での扱い |
|---|---|---|
| `firebase-applet-config.json` | Firebase設定（projectId, apiKey, authDomain等）がハードコード | そのまま残してよい（後述） |
| `firebase-blueprint.json` | Firestoreのデータモデルスキーマ（AI Studio固有） | 削除不要。参照されなければ無害 |
| `metadata.json` | アプリメタデータ（名前、説明、requestFramePermissions） | AI Studio固有。削除不要。参照されなければ無害 |
| `firestore.rules` | Firestoreセキュリティルール | `firebase deploy --only firestore:rules` で別途適用が必要 |
| `.env.example` | `GEMINI_API_KEY`（AI Studioが自動注入していたもの）と `APP_URL` | 環境変数として登録が必要 |

## Firebase設定の扱い

`firebase-applet-config.json` にはAPIキーがハードコードされているが、**Firebase Web APIキーはGoogleの設計上「公開前提」**。Firestoreのセキュリティルールによって保護する思想のため、Gitリポジトリに含めても直接的なセキュリティリスクは低い。

ただし、環境変数化（`NEXT_PUBLIC_FIREBASE_*`）したい場合は `firebase.ts` がJSONを直接インポートしているため、このファイルの修正も合わせて必要。

## Vercelへの移行チェックリスト

- [ ] `output: 'standalone'` を `next.config.ts` から**除去**（Vercelでは不要・競合する）
- [ ] `vercel.json` に `{"framework": "nextjs"}` を追加（フレームワーク自動検出失敗対策）
- [ ] `GEMINI_API_KEY` をVercel環境変数に登録
- [ ] `APP_URL` をVercel環境変数に登録（デプロイ先のURLを設定）
- [ ] `vercel link --yes --project <name> --scope <scope>` でプロジェクトに接続
- [ ] `vercel deploy` でプレビュービルド確認
- [ ] `vercel deploy --prod` で本番反映
- [ ] Firebase認証が必要な場合、VercelのURLをFirebase Authの**承認済みドメイン**に追加
- [ ] `firestore.rules` の内容を確認し、認証・認可ルールが適切であることを検証する（Firebase Web APIキーが公開前提のため、ルールが最後の防御線）

### vercel.json の最小構成例

```json
{
  "framework": "nextjs"
}
```

## AI Studio固有の設定で安全に残せるもの

- **`DISABLE_HMR` webpack設定**: 環境変数が未設定なら発動しない（無害）
- **`metadata.json`**: 参照されなければ無害
- **`firebase-blueprint.json`**: 参照されなければ無害

## 注意点・よくある落とし穴

### Gemini API呼び出し
AI StudioではGemini API呼び出しがサーバーサイドで自動注入されるが、**Vercelでは自分でAPIルートを実装する必要がある**。`.env.example` に記載の `GEMINI_API_KEY` を環境変数として設定し、APIルートから呼び出す実装を追加すること。

### Firestoreセキュリティルール
`firestore.rules` はVercelデプロイとは別に、Firebase CLIで適用が必要。

```bash
firebase deploy --only firestore:rules
```

### requestFramePermissions
`metadata.json` の `requestFramePermissions` はiframe埋め込み用の設定。スタンドアロンデプロイでは不要なため、特別な対応不要。

### Firebase認証の承認済みドメイン
Vercelデプロイ後にFirebase Authを使用する場合、Firebase ConsoleでVercelのドメイン（`xxx.vercel.app` および独自ドメイン）を承認済みドメインに追加しないと認証エラーになる。

**Firebase Console** → Authentication → Settings → 承認済みドメイン → ドメインを追加
