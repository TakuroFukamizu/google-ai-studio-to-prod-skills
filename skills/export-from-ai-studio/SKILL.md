---
name: export-from-ai-studio
description: Use when extracting code from Google AI Studio exports, converting playground prompts to structured application code, or migrating AI Studio prototypes to a local development environment
---

# Export from AI Studio

## Overview

Google AI Studio exports code as a single script with hardcoded API keys and inline prompts. This skill extracts, restructures, and cleans up that code into a production-ready project structure.

## When to Use

- User has a Google AI Studio export (Python or Node.js)
- User wants to convert an AI Studio playground session to local code
- User pastes code with `genai.configure(api_key=...)` or similar patterns
- User says "AI Studioからコード持ってきて" or "export my AI Studio project"

**When NOT to use:** If the user already has structured code that just needs deployment, skip to `cloud-run-deploy`.

## Core Pattern

### Before (AI Studio Export)
```python
import google.generativeai as genai
genai.configure(api_key="AIzaSy...")
model = genai.GenerativeModel("gemini-2.0-flash")
response = model.generate_content("Translate this to French: Hello world")
print(response.text)
```

### After (Production-Ready)
```python
import os
from google import genai

client = genai.Client(api_key=os.environ["GEMINI_API_KEY"])

def translate(text: str, target_lang: str) -> str:
    response = client.models.generate_content(
        model="gemini-2.0-flash",
        contents=f"Translate this to {target_lang}: {text}",
    )
    return response.text
```

## firebase-applet-config.json の API Key 問題

AI Studio が生成するプロジェクトには `firebase-applet-config.json` が含まれ、Firebase Web API Key が平文で埋め込まれている:

```json
{
  "projectId": "gen-lang-client-...",
  "apiKey": "AIzaSy...",          ← これがそのままgitにコミットされる
  "authDomain": "gen-lang-client-....firebaseapp.com",
  "firestoreDatabaseId": "ai-studio-...",
  ...
}
```

**リスク:**
- Firebase Web API Key は厳密には「シークレット」ではない（クライアントJSに埋め込まれる前提）が、**API Key に利用制限がかかっていなければ**、第三者がこの Key で Firebase Auth / Firestore を叩ける
- AI Studio の共有プロジェクト (`gen-lang-client-*`) の Key が漏洩すると、他ユーザーの quota にも影響しうる
- git history に残ると、Key をローテーションしても過去のコミットから復元可能

**修正手順:**

1. `firebase-applet-config.json` から `apiKey` フィールドを環境変数に外出しする
2. コードで実行時に注入する:

```typescript
// Before (AI Studio default)
import config from "./firebase-applet-config.json";
const app = initializeApp(config);

// After (production-ready)
import baseConfig from "./firebase-applet-config.json";
const app = initializeApp({
  ...baseConfig,
  apiKey: process.env.FIREBASE_API_KEY,
});
```

3. `firebase-applet-config.json` から `apiKey` を削除し、代わりにプレースホルダーを設定:

```json
{
  "projectId": "gen-lang-client-...",
  "apiKey": "SET_VIA_ENV_VAR",
  ...
}
```

4. `.env.example` に追記:
```
FIREBASE_API_KEY=your-firebase-web-api-key
```

5. Google Cloud Console で API Key にHTTPリファラー制限を設定する

## Quick Reference

| AI Studio Pattern | Production Replacement |
|-------------------|----------------------|
| Hardcoded API key | `os.environ["GEMINI_API_KEY"]` |
| `firebase-applet-config.json` の `apiKey` | 環境変数 `FIREBASE_API_KEY` で実行時注入 |
| `genai.configure()` | `genai.Client()` |
| Inline prompt strings | Parameterized functions |
| `print(response)` | Return values / structured output |
| Single script | Modular files with `main.py` entry point |
| No error handling | Try/catch with retry logic |

## Implementation Steps

1. **Identify the export type** — Python (`google-generativeai`) or Node.js (`@google/generative-ai`)
2. **Gemini 実使用を検出** — ソースコードで Gemini SDK が実際に import/使用されているか確認 (下記参照)
3. **Extract Gemini API key** — (Gemini 使用時のみ) Replace hardcoded key with environment variable
4. **Sanitize `firebase-applet-config.json`** — `apiKey` を環境変数に外出し（上記参照）
5. **Migrate to latest SDK** — (Gemini 使用時のみ) Use `google-genai` (Python) or latest `@google/genai` (Node.js)
6. **Extract functions** — Each `generate_content` call becomes a named function
7. **Add typing** — Type hints (Python) or TypeScript types (Node.js)
8. **Create `.env.example`** — Gemini 使用時は `GEMINI_API_KEY` を記載、未使用時は除外。`FIREBASE_API_KEY` は常に記載。
9. **Add `requirements.txt` / `package.json`** — Pin dependency versions

## Gemini 実使用の検出

AI Studio のエクスポートは、プロジェクトが Gemini API を実際に使っていなくても `@google/genai` (または `google-generativeai`) を dependencies に含め、`.env.example` に `GEMINI_API_KEY` を記載し、`vite.config.ts` で `process.env.GEMINI_API_KEY` をブラウザに expose する。

**検出手順:**

1. **ソースコードで import を検索:**
   - Node.js: `src/` 以下の `.ts`, `.tsx`, `.js`, `.jsx` ファイルで `@google/genai` or `@google/generative-ai` or `google-generativeai` の import/require を検索
   - Python: `.py` ファイルで `import google.generativeai` or `from google import genai` or `import genai` を検索
   - `vite.config.ts`, `next.config.js` 等のビルド設定ファイルは**検索対象外** (expose しているだけなので)

2. **import が見つかった → Gemini 使用中。** 通常のフロー (API Key 環境変数化、SDK 更新等) を実行。

3. **import が見つからない → Gemini 未使用。** 以下のクリーンアップを実行:

### Gemini 未使用時のクリーンアップ

**a. 不要な依存を削除:**
- `package.json` から `@google/genai` (または `@google/generative-ai`, `google-generativeai`) を削除
- `requirements.txt` から `google-generativeai` (または `google-genai`) を削除
- `npm install` / `pip install` を再実行して lockfile を更新

**b. vite.config.ts / ビルド設定から不要な expose を削除:**
```typescript
// Before: AI Studio テンプレートのデフォルト
define: {
  'process.env.GEMINI_API_KEY': JSON.stringify(env.GEMINI_API_KEY),
  'import.meta.env.VITE_APP_URL': JSON.stringify(env.APP_URL),
},

// After: GEMINI_API_KEY の expose を削除
define: {
  'import.meta.env.VITE_APP_URL': JSON.stringify(env.APP_URL),
},
```

**c. `.env.example` から GEMINI_API_KEY を削除:**
```
# Before:
GEMINI_API_KEY="MY_GEMINI_API_KEY"
APP_URL="MY_APP_URL"

# After:
APP_URL="MY_APP_URL"
```

**d. README のセットアップ手順を更新:**
- 「`GEMINI_API_KEY` を `.env.local` に設定」のステップを削除
- Gemini を使わないプロジェクトであることを明記

**e. ユーザーに報告:**
```
Gemini SDK (@google/genai) は dependencies に含まれていますが、
ソースコードでは使用されていません。

以下をクリーンアップしました:
  ✓ package.json から @google/genai を削除
  ✓ vite.config.ts から GEMINI_API_KEY の expose を削除
  ✓ .env.example から GEMINI_API_KEY を削除

GEMINI_API_KEY の設定は不要です。
```

## Common Mistakes

- **Leaving API key in code** — Always extract to env var immediately
- **`firebase-applet-config.json` の `apiKey` を放置** — Gemini API Key と同様に環境変数化すること。git history に残るとローテーションしても復元可能
- **Using deprecated SDK** — AI Studio may export `google-generativeai`, migrate to `google-genai`
- **Ignoring streaming** — If AI Studio used streaming, preserve that in the export
- **Losing system instructions** — AI Studio system prompts must be extracted and preserved
- **Gemini 未使用なのに GEMINI_API_KEY を必須にする** — AI Studio のテンプレートは Gemini SDK を未使用でも dependencies に含める。`src/` 以下で実際に import されているか検出し、未使用なら依存削除 + `.env.example` から除外すること。
