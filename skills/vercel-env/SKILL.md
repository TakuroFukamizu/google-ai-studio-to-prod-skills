---
name: vercel-env
description: "vercel env、環境変数、env設定、secret、GEMINI_API_KEY、APP_URL、vercel環境変数の設定・管理・追加・削除・確認方法"
---

# Vercel 環境変数の管理

## 1. 環境変数の確認

```bash
vercel env ls
```

## 2. 環境変数の追加

```bash
vercel env add <KEY>
```

- 対話形式で値と適用環境（production / preview / development）を選択
- 非対話で追加する場合:
  ```bash
  echo "value" | vercel env add KEY production
  ```

## 3. 環境変数の取得（ローカルに反映）

```bash
vercel env pull .env.local
```

- `.env.local` に development 環境の環境変数がダウンロードされる
- `vercel link` 時にも自動で実行される
- `VERCEL_OIDC_TOKEN` が自動追加される（VercelのOpenID Connectトークン。ローカル開発では通常無視して構わない）

## 4. 環境変数の削除

```bash
vercel env rm <KEY> <environment>
```

- environment: `production` / `preview` / `development`

## 5. 環境の種類

| 環境 | 用途 |
|------|------|
| production | `vercel deploy --prod` 時に使用 |
| preview | `vercel deploy`（プレビュー）時に使用 |
| development | ローカル開発時（`vercel dev` や `vercel env pull`）に使用 |

## 6. ベストプラクティス

- APIキーや機密情報はVercelの環境変数で管理（`.env` ファイルをgitにコミットしない）
- `NEXT_PUBLIC_` プレフィックスの変数はクライアントサイドに公開される
- 環境ごとに異なる値を設定可能（本番DBと開発DBのURL等）
- `.env.example` に変数名のみ記載してドキュメント代わりにする

## 7. 注意点

- Vercelダッシュボードからも設定可能（Settings > Environment Variables）
- 環境変数変更後は再デプロイが必要
- Sensitive（暗号化）設定はダッシュボードから行う
