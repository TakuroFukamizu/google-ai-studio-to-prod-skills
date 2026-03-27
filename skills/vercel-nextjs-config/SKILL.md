---
name: vercel-nextjs-config
description: "Next.jsアプリをVercelにデプロイする際の設定注意点。next.config、vercel設定、standalone、Next.js Vercel、output directory error、フレームワーク設定、vercel.json nextjs に関連する場合に自動発動。"
---

# Vercel × Next.js デプロイ設定ナレッジ

Next.jsアプリをVercelにデプロイする際に注意すべき設定事項をまとめる。

---

## 1. `output: 'standalone'` の除去（必須）

`next.config.ts` または `next.config.js` に以下のような設定がある場合、**Vercelデプロイ前に除去が必要**。

```ts
// NG: Vercelデプロイ時は削除すること
const nextConfig = {
  output: 'standalone',
};
```

- `standalone` はDocker/Node.jsスタンドアロンサーバー向けの設定
- Vercelは独自のビルドパイプラインを持つため、この設定があるとデプロイが正しく動作しない
- Cloud Run, Railway, Fly.io等のコンテナベースPaaSでは必要だが、**Vercelでは不要かつ有害**

---

## 2. `vercel.json` のフレームワーク指定

Next.jsアプリの場合、`vercel.json` に以下を追加することを推奨:

```json
{
  "framework": "nextjs"
}
```

- これがないと `No Output Directory named "public" found` エラーが発生する場合がある
- Vercelは通常フレームワークを自動検出するが、AI Studioエクスポート等の特殊な構成では失敗することがある
- 明示指定することで確実にNext.js向けビルドパイプラインが使われる

---

## 3. ESLint設定

```ts
const nextConfig = {
  eslint: {
    ignoreDuringBuilds: true, // Vercelでも有効
  },
};
```

- `eslint.ignoreDuringBuilds: true` はVercelでも有効（ビルド時のLintスキップ）
- 本番デプロイでLintエラーによるビルド失敗を避けたい場合に便利

---

## 4. TypeScript設定

```ts
const nextConfig = {
  typescript: {
    ignoreBuildErrors: false, // デフォルト値（推奨）
    // ignoreBuildErrors: true, // 型エラーを無視したい場合（非推奨）
  },
};
```

- `typescript.ignoreBuildErrors: false` がデフォルト（ビルド時の型エラーで失敗する）
- 必要に応じて `true` に変更可能だが、品質上非推奨

---

## 5. webpack カスタム設定（AI Studio由来）

```ts
// AI Studio固有のwebpack設定（Vercelでは無害）
webpack: (config, { dev }) => {
  if (process.env.DISABLE_HMR) {
    // ...
  }
  return config;
},
```

- `DISABLE_HMR` のようなAI Studio固有のwebpack設定はVercelでは無害（環境変数未設定なら発動しない）
- ただし不要な設定はコメントで「AI Studio由来」と記しておくと管理しやすい

---

## 6. 画像最適化

```ts
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'example.com',
      },
    ],
  },
};
```

- Vercelでは `next/image` の最適化が自動で有効
- 外部画像ドメインは `images.remotePatterns` で設定（古い `domains` 配列は非推奨）

---

## 7. ミドルウェア・Edge Runtime

- Vercel上ではEdge Runtimeが自動的に利用可能
- `middleware.ts` はVercelのEdge Networkで実行される
- 特別な設定なしでMiddlewareが動作する

---

## デプロイ前チェックリスト

- [ ] `output: 'standalone'` を `next.config.ts/js` から除去
- [ ] `vercel.json` に `"framework": "nextjs"` を追加
- [ ] `vercel link` でプロジェクトをリンク済みか確認（未リンク状態での `--yes` 付きデプロイは新規プロジェクト自動作成の危険あり）
- [ ] 環境変数をVercelダッシュボードまたは `vercel env` コマンドで設定済みか確認
