---
name: vercel-deploy
description: "Vercelへのデプロイ手順。vercel deploy、デプロイ、本番デプロイ、プレビューデプロイ、vercel production、Vercelにデプロイ、vercel link、Vercelデプロイフロー等のリクエストで自動発動。"
---

# Vercel デプロイ手順

## 前提条件

- `vercel link` が完了していること（`.vercel/` ディレクトリが存在すること）
- **危険**: 未link状態で `vercel deploy --yes` を実行すると、新規プロジェクトが自動作成されてしまう
- `vercel link` 時は必ず `--project` と `--scope` を明示指定すること

## プレビューデプロイ

```bash
vercel deploy
```

- ブランチに紐づいたプレビューURLが生成される
- URL形式: `https://<project>-<hash>-<scope>.vercel.app`
- PRのプレビュー確認にも利用できる
- 本番環境に影響しないため、変更内容の動作確認に最適

## 本番デプロイ

```bash
vercel deploy --prod
```

- 本番URLに反映される
- URL形式: `https://<project>.vercel.app`（カスタムドメイン未設定時）
- **ベストプラクティス**: 必ずプレビューデプロイで動作確認してから本番デプロイすること

## よくあるエラーと対処

### `No Output Directory named "public" found`

`vercel.json` にフレームワーク設定を追加する:

```json
{
  "framework": "nextjs"
}
```

フレームワーク自動検知が失敗した場合に発生する。

### ビルドキャッシュ

- 初回デプロイ後にキャッシュが生成され、2回目以降は高速化される
- キャッシュが原因で問題が起きる場合は Vercel ダッシュボードからキャッシュをクリアできる

### 環境変数未設定

- ビルドは通るが実行時エラーになる場合がある
- Vercel ダッシュボードの「Settings > Environment Variables」で設定すること
- `.env.local` はローカルのみ有効であり、Vercel には自動反映されない

## Inspect リンク

- デプロイ実行後、コンソールに Inspect URL が出力される
- ビルドログ・関数ログ・エラー詳細を確認できる
- デプロイ失敗時はまず Inspect URL のビルドログを確認すること

## 推奨デプロイフロー

```bash
# 初回のみ: プロジェクトをリンク
vercel link --yes --project <project-name> --scope <scope-name>

# プレビューデプロイで動作確認
vercel deploy

# 問題なければ本番に反映
vercel deploy --prod
```

## 注意事項

- `vercel link` 前に `vercel deploy` を実行してはいけない（意図しない新規プロジェクトが作成される）
- 各プロジェクトの `CLAUDE.md` に許可プロジェクト名とスコープ名を記載しておくこと
- Vercel CLI はローカルファイルシステムから直接デプロイするため、コミットやプッシュは不要（worktree 内の未コミット変更もそのままデプロイされる）
- ただし GitHub 連携（Git Integration）による自動デプロイを使用している場合は、ブランチへのプッシュが必要
