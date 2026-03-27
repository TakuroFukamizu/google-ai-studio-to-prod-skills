---
name: graduate-from-ai-studio
description: Use when taking an AI Studio-generated project to independent production deployment. Analyzes the project, generates Dockerfile, IaC (Terraform/Pulumi/CLI), GitHub Actions CI/CD, and Firebase configuration for Cloud Run deployment.
---

# Graduate from AI Studio

## Overview

AI Studio Apps are deployed on a shared GCP project managed by Google. This skill "graduates" an AI Studio project to an independent, production-ready deployment by analyzing the existing codebase and generating all necessary infrastructure files in one batch.

## When to Use

- User says "AI Studioから卒業したい", "make this independently deployable", "本番環境を自分で管理したい"
- Project has `firebase-applet-config.json` (AI Studio signature file)
- User wants to deploy to their own GCP project with proper IaC and CI/CD
- User wants to stop depending on AI Studio's managed Cloud Run

**When NOT to use:**
- Project was never on AI Studio → use `cloud-run-deploy` directly
- User just wants to export code → use `export-from-ai-studio`

## Execution Flow

This skill follows a **plan-then-execute** approach: analyze the project, present the plan, generate all files upon approval.

### Phase 1: Analyze (automatic)

- Detect framework (Express/Vite, Next.js, Flask, FastAPI, etc.)
- Detect runtime (Node.js version, Python version)
- Read `firebase-applet-config.json` → extract project ID, database ID
- Read `package.json` / `requirements.txt` → detect build commands, port
- Read `firestore.rules` → collect security rules
- Read server entry point → detect PORT usage, static file serving
- Scan for environment variables (`GEMINI_API_KEY`, `APP_URL`, etc.)

### Phase 2: Present choices to user

| Choice | Options |
|--------|---------|
| **Firebase project** | (a) Create new Firebase project (full independence) / (b) Continue using AI Studio's project (`gen-lang-client-*`) |
| **IaC tool** | (a) Terraform / (b) Pulumi (TypeScript) / (c) gcloud + firebase CLI scripts |
| **GCP region** | default: `asia-northeast1` |

### Phase 3: Present plan → get approval

### Phase 3.5: Sanitize firebase-applet-config.json

- `apiKey` を削除し、プレースホルダーに置換
- コード側で環境変数から注入するよう修正
- git history に残っている場合は警告を表示

### Phase 4: Generate all files

| File | Description |
|------|-------------|
| `Dockerfile` + `.dockerignore` | Multi-stage build |
| IaC files | Based on user's choice (Terraform / Pulumi / CLI scripts) |
| `.github/workflows/deploy.yml` | CI/CD pipeline |
| `firebase.json` + `.firebaserc` | Firebase deploy config |
| `firestore.indexes.json` | Inferred from code queries |
| `.env.example` | Updated with all detected env vars |
| README section | Deployment instructions |

## Phase 1: Project Analysis

Read the following files and extract configuration:

| File | Extract |
|------|---------|
| `firebase-applet-config.json` | `projectId`, `firestoreDatabaseId`, `appId`, **`apiKey` (要サニタイズ)** |
| `package.json` / `requirements.txt` | Runtime, dependencies, build script, start script |
| `server.ts` / `server.js` / `main.py` | Port number, static file serving path, API routes |
| `firestore.rules` | Security rules (copy as-is) |
| `firebase-blueprint.json` | Collections, field types → infer composite indexes |
| `.env.example` | Existing env vars |
| `vite.config.ts` / `next.config.js` | Build tool config, env var injection |
| `metadata.json` | App name, description |

### Detection Heuristics

**Framework detection:**
- `express` in dependencies → Express.js
- `next` in dependencies → Next.js
- `fastapi` in requirements → FastAPI
- `flask` in requirements → Flask
- `@vitejs/plugin-react` in dependencies → Vite + React SPA

**Port detection:**
- Search for `.listen(PORT` or `.listen(Number(` patterns
- Default: 3000 (Node.js) or 8080 (Python)
- Cloud Run will override via `PORT` env var

**Build command detection:**
- `package.json` → `scripts.build` (typically `vite build` or `next build`)
- Python → no build step needed

**Start command detection:**
- `package.json` → `scripts.dev` or `scripts.start`
- Common: `tsx server.ts`, `node dist/server.js`, `next start`

## Phase 2: User Choices

Present detected configuration and ask for 3 choices:

```
=== AI Studio Graduation Plan ===

Detected:
  Framework: Express + Vite (React SPA)
  Runtime: Node.js (tsx)
  Port: 3000
  Build: vite build
  Start: tsx server.ts
  Firestore DB: ai-studio-ffcfd659-...
  Collections: claude_usage, user_configs
  Env vars: GEMINI_API_KEY, APP_URL

Questions:
  1. Firebase project:
     (a) Create new Firebase project [recommended for production]
     (b) Continue using AI Studio project (gen-lang-client-*)

  2. IaC tool:
     (a) Terraform [recommended, most ecosystem support]
     (b) Pulumi (TypeScript) [same language as your project]
     (c) gcloud + firebase CLI scripts [simplest, no extra tools]

  3. GCP region: [asia-northeast1]
```

## Phase 3: Plan Presentation

After user answers, present the full file list that will be generated:

```
Files to generate:
  ✦ Dockerfile                           (multi-stage Node.js build)
  ✦ .dockerignore
  ✦ firebase.json                        (Firestore rules + indexes deploy config)
  ✦ .firebaserc                          (project alias)
  ✦ firestore.indexes.json               (composite indexes)
  ✦ .github/workflows/deploy.yml         (CI/CD pipeline)
  ✦ .env.example                         (updated)
  ✦ infra/terraform/                     (if Terraform selected)
  │   ├── main.tf
  │   ├── variables.tf
  │   ├── outputs.tf
  │   ├── terraform.tfvars.example
  │   └── modules/
  │       ├── cloud-run/main.tf
  │       ├── artifact-registry/main.tf
  │       ├── iam/main.tf
  │       ├── secret-manager/main.tf
  │       └── firestore/main.tf          (if new Firebase project)

Proceed? (y/n)
```

## Phase 4: File Generation

### firebase-applet-config.json のサニタイズ（Phase 3.5）

AI Studio が生成する `firebase-applet-config.json` には Firebase Web API Key (`apiKey`) が平文で含まれている。これをそのままgitにコミットするとセキュリティリスクになるため、卒業時に必ずサニタイズする。

**手順:**

1. `firebase-applet-config.json` の `apiKey` をプレースホルダーに置換:
```json
{
  "projectId": "your-project-id",
  "apiKey": "SET_VIA_ENV_VAR",
  "authDomain": "your-project-id.firebaseapp.com",
  ...
}
```

2. アプリのFirebase初期化コードを修正し、環境変数から `apiKey` を注入:
```typescript
import baseConfig from "./firebase-applet-config.json";
const app = initializeApp({
  ...baseConfig,
  apiKey: process.env.FIREBASE_API_KEY,
});
```

3. `.env.example` に `FIREBASE_API_KEY=` を追加

4. IaC で Secret Manager に `firebase-api-key` を追加し、Cloud Run の `--set-secrets` で注入

5. git history に既に `apiKey` がコミットされている場合、ユーザーに警告:
```
⚠️  firebase-applet-config.json の apiKey が git history に残っています。
    Google Cloud Console で API Key をローテーションすることを推奨します。
    手順: https://console.cloud.google.com/apis/credentials
```

### Dockerfile

Use the template at `templates/Dockerfile.node`. Key requirements:
- Multi-stage build (builder → production)
- `npm ci --omit=dev` for production dependencies only
- `npm run build` for frontend assets
- Copy `dist/` from builder stage
- Copy server files and `firebase-applet-config.json`
- `PORT` env var support (Cloud Run injects this)
- Run as non-root user

### IaC — Terraform

Use templates at `templates/terraform/`. Generate into `infra/terraform/`:

**Resources to create:**
- `google_artifact_registry_repository` — Docker image registry
- `google_cloud_run_v2_service` — Cloud Run service
- `google_secret_manager_secret` + `_version` — For GEMINI_API_KEY and other secrets
- `google_service_account` — Dedicated SA for Cloud Run
- `google_project_iam_member` — SA permissions (Secret Manager accessor, Firestore user)
- `google_cloud_run_v2_service_iam_member` — Public access (if needed)
- (If new project) `google_firestore_database` — Firestore database
- (If new project) `google_identity_platform_config` — Firebase Auth

**Variables:**
- `project_id` (required)
- `region` (default from user choice)
- `service_name` (from metadata or directory name)
- `firebase_project_mode` — `"new"` or `"existing"`

### IaC — Pulumi (TypeScript)

Generate into `infra/pulumi/`:
- `Pulumi.yaml` — Project definition
- `index.ts` — All resources in TypeScript
- `package.json` — `@pulumi/gcp` dependency
- Same resources as Terraform

### IaC — CLI Scripts

Generate into `infra/scripts/`:
- `setup.sh` — One-time setup (enable APIs, create SA, create secrets, create Artifact Registry repo)
- `deploy.sh` — Build + push + deploy to Cloud Run
- `firebase-deploy.sh` — Deploy Firestore rules and indexes

### GitHub Actions

Use template at `templates/deploy.yml`. Key features:
- Trigger: push to `main` branch
- Authentication: Workload Identity Federation (no service account keys)
- Steps: checkout → auth → Docker build → Artifact Registry push → Cloud Run deploy → Firebase deploy
- Secrets required: `WIF_PROVIDER`, `WIF_SERVICE_ACCOUNT`, `GCP_PROJECT_ID`

### Firebase Configuration

**`firebase.json`:**
```json
{
  "firestore": {
    "rules": "firestore.rules",
    "indexes": "firestore.indexes.json"
  }
}
```

**`.firebaserc`:**
```json
{
  "projects": {
    "default": "<project-id>"
  }
}
```

**`firestore.indexes.json`:**
Infer composite indexes from Firestore queries in the codebase. Look for patterns:
- `where('field1', ...).where('field2', ...)` → composite index on field1 + field2
- `where('field', ...).orderBy('field2')` → composite index on field + field2

### Environment Variables Update

Update `.env.example` with all detected variables plus new ones.
**`GEMINI_API_KEY` は、ソースコードで Gemini SDK が実際に import/使用されている場合のみ含める** (export-from-ai-studio スキルの「Gemini 実使用の検出」を参照)。

```
# Existing (Gemini SDK を使用している場合のみ)
GEMINI_API_KEY=your-gemini-api-key
APP_URL=https://your-service-url.run.app

# Added by graduation
GOOGLE_CLOUD_PROJECT=your-project-id
FIREBASE_API_KEY=your-firebase-web-api-key
PORT=3000
```

### README Deployment Section

Append a deployment section to the existing README:

```markdown
## Deployment

### Prerequisites
- Google Cloud SDK (`gcloud`)
- Firebase CLI (`firebase-tools`)
- Docker (for local testing)

### First-time Setup
1. Set up GCP project and enable APIs
2. Configure IaC (see `infra/` directory)
3. Set GitHub Secrets for CI/CD

### Manual Deploy
\`\`\`bash
gcloud run deploy SERVICE_NAME --source . --region REGION
firebase deploy --only firestore
\`\`\`
```

## Post-Generation Checklist

After generating all files, present this checklist:

```
=== Graduation Complete ===

Generated files:
  ✓ Dockerfile + .dockerignore
  ✓ IaC files (infra/)
  ✓ GitHub Actions (deploy.yml)
  ✓ Firebase config (firebase.json, .firebaserc, firestore.indexes.json)
  ✓ .env.example (updated)

Next steps:
  1. Create GCP project (if new): gcloud projects create YOUR_PROJECT_ID
  2. Run IaC setup:
     - Terraform: cd infra/terraform && terraform init && terraform plan
     - Pulumi: cd infra/pulumi && npm install && pulumi up
     - Scripts: bash infra/scripts/setup.sh
  3. Set up Workload Identity Federation for GitHub Actions:
     See: https://github.com/google-github-actions/auth#workload-identity-federation
  4. Set GitHub Secrets: WIF_PROVIDER, WIF_SERVICE_ACCOUNT, GCP_PROJECT_ID
  5. Push to main to trigger first deploy
  6. (If new project) Migrate Firestore data from AI Studio project
```

## Common Mistakes

- **`firebase-applet-config.json` の `apiKey` をサニタイズせず卒業** — API Key が git history に残り、ローテーションしない限り漏洩リスクが継続する。Phase 3.5 を必ず実行すること
- **Forgetting `firebase-applet-config.json` in Docker image** — This file is needed at runtime for Firebase SDK init（ただし `apiKey` は環境変数で注入）
- **Hardcoding AI Studio's `gen-lang-client-*` project ID** — If using a new project, all references must be updated
- **Not setting `PORT` env var** — Cloud Run injects `PORT`; the app must listen on it
- **Missing composite indexes** — Firestore queries with multiple `where` or `where` + `orderBy` need indexes
- **Skipping Workload Identity Federation** — Service account keys are a security risk; always use WIF for CI/CD
- **Not copying `firestore.rules` into the deploy pipeline** — Rules changes need to be deployed alongside code
