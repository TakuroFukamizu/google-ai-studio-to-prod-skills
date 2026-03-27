---
name: vercel-railway-deploy
description: Use when deploying to Vercel or Railway. For Vercel, detailed sub-skills (vercel-link, vercel-env, vercel-deploy, vercel-nextjs-config, vercel-firebase-auth-domain) are available. migrate-wizard orchestrates them automatically
---

# Vercel / Railway Deploy

## Overview

Deploy AI Studio prototypes to Vercel (frontend/edge functions) or Railway (full-stack containers). Use when the user doesn't want GCP or needs a simpler deployment experience.

## When to Use

- User says "Vercelにデプロイ", "deploy to Railway", "I don't want GCP"
- Next.js / React frontend with Gemini API backend
- User wants git-push-to-deploy simplicity

**When NOT to use:** If user wants GCP → use `cloud-run-deploy`.

## Platform Decision

| Criteria | Vercel | Railway |
|----------|--------|---------|
| Best for | Next.js, frontend + API routes | Any container, long-running processes |
| Gemini integration | API routes / Edge Functions | Direct container with SDK |
| Pricing | Generous free tier (hobby) | $5/mo + usage |
| Deploy method | Git push / `vercel deploy` | Git push / `railway up` |
| Streaming support | Edge Functions | Native |
| Long-running tasks | Limited (30s hobby / 300s pro) | Unlimited |

## Vercel Quick Reference

```bash
# Install CLI
npm i -g vercel

# Deploy
vercel

# Set env var
vercel env add GEMINI_API_KEY

# Deploy to production
vercel --prod
```

### Vercel Project Structure (Next.js)
```
app/
├── page.tsx              # Frontend
├── api/
│   └── generate/
│       └── route.ts      # Gemini API route
├── layout.tsx
└── globals.css
```

### Vercel 詳細スキル

Vercel デプロイの各ステップには専用スキルがあります。`migrate-wizard` でワンパスマイグレーションする場合はこれらが自動で呼ばれます:

| Skill | 役割 |
|-------|------|
| [vercel-ai-studio-export](../vercel-ai-studio-export/) | AI Studio エクスポート構造のナレッジ |
| [vercel-gcp-project-identification](../vercel-gcp-project-identification/) | GCP プロジェクト特定のナレッジ |
| [vercel-nextjs-config](../vercel-nextjs-config/) | Next.js 設定の Vercel 向け調整 |
| [vercel-link](../vercel-link/) | Vercel プロジェクトへのリンク |
| [vercel-env](../vercel-env/) | Vercel 環境変数管理 |
| [vercel-deploy](../vercel-deploy/) | プレビュー → 本番デプロイ |
| [vercel-firebase-auth-domain](../vercel-firebase-auth-domain/) | Firebase Auth authorized domain 追加 |

## Railway Quick Reference

```bash
# Install CLI
npm i -g @railway/cli

# Login
railway login

# Init project
railway init

# Deploy
railway up

# Set env var
railway variables set GEMINI_API_KEY=xxx
```

## Common Mistakes

- **Vercel: Timeout on long Gemini calls** — Use streaming or Edge Functions for long responses
- **Railway: Forgetting to set PORT** — Railway sets `PORT` env var; your app must listen on it
- **Both: API key in code** — Always use platform's env var / secrets management
- **Vercel: Using Node.js runtime for streaming** — Use Edge Runtime for streaming responses
