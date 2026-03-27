---
name: migrate-wizard
description: Use when migrating an AI Studio project to production in one pass. Orchestrates all migration steps (analyze, export, repo setup, graduation, monitoring, security) with upfront configuration and automatic step detection. Trigger examples - "AI Studioから本番に移行して", "migrate to production", "ワンパスでマイグレーション", "production migration wizard"
---

# Migrate Wizard

AI Studio プロジェクトを本番環境にワンパスでマイグレーションするオーケストレーター。各スキルを順番に実行し、必要な設定は最初にまとめて聞く。

## Overview

このスキルは以下の6ステップを自動検出・一括実行する:

| Step | Skill | やること |
|:----:|-------|---------|
| 0 | analyze-and-document | プロジェクト分析 → CLAUDE.md 生成 |
| 1 | export-from-ai-studio | コード整形 + apiKey サニタイズ |
| 2 | repo-initializer-google | GitHub リポ整備 (.gitignore, README) |
| 3 | graduate-from-ai-studio | Dockerfile + IaC + CI/CD + Firebase 設定 **ファイル生成** |
| 3.5 | cloud-run-deploy | **GCP インフラ構築 + 初回デプロイ** (gcloud 認証済みの場合) |
| 4 | monitoring-sentry-datadog | 監視・エラートラッキング |
| 5 | security-hardening-gcp | セキュリティ強化 (SA, Secret Manager, API Key 制限) |

**v1 ではデプロイ先は Cloud Run のみ。** Vercel/Railway を使いたい場合は `vercel-railway-deploy` スキルを個別に使用すること。

## When to Use

- User says "本番に移行して", "migrate to production", "ワンパスでマイグレーション"
- AI Studio で作ったプロジェクトを初めて本番化する
- 複数のスキルを個別に呼ぶのが面倒なとき

**When NOT to use:**
- 特定のステップだけ実行したい → 個別スキルを直接使う
- Vercel/Railway にデプロイしたい → `vercel-railway-deploy` を使う
- 開発環境のセットアップだけ → `setup-dev-environment` を使う

---

## Phase 0: Prerequisites Check

**最初に実行環境を確認する。** 以下のツールの存在と認証状態をチェック:

| Tool | Check command | Required | Fallback |
|------|--------------|----------|----------|
| `git` | `git --version` | **必須** (なければ中止) | — |
| `gcloud` | `gcloud auth print-access-token 2>/dev/null` | 推奨 | リモート操作を手動コマンドとして Summary に集約 |
| `firebase` | `npx firebase-tools --version 2>/dev/null` | 推奨 | Firebase デプロイコマンドを Summary に集約 |
| `gh` | `gh auth status 2>/dev/null` | 推奨 | GitHub repo 作成を手動手順として案内 |
| `npm` or `pip` | `npm --version` / `pip --version` | 推奨 | SDK インストールコマンドを Summary に集約 |
| `docker` | `docker --version 2>/dev/null` | 任意 | Dockerfile 生成はするがローカル検証不可と案内 |

**出力フォーマット:**
```
=== Environment Check ===

✓ git       installed
✓ gcloud    authenticated (project: my-project-123)
✓ firebase  installed (v13.x)
✓ gh        authenticated (user: TakuroFukamizu)
✓ npm       installed (v20.x)
△ docker    not found — Dockerfile is generated but cannot validate locally
✗ terraform not found — will ask about IaC choice

→ Proceeding. Missing tools affect remote operations only;
  manual commands will be listed in the summary.
```

**判定ルール:**
- `git` がない → **中止**。`git --version` が失敗したらエラーメッセージを出して終了。
- `gcloud` 認証済み → **フル自動化モード**: GCP プロジェクト作成、API 有効化、Cloud Run デプロイ、Secret Manager、IAM 設定をすべて自動実行。
- `gcloud` 未認証 → **ファイル生成モード**: IaC ファイルと CI/CD 設定を生成し、リモート操作は手動コマンドとして Phase 5 Summary に集約。ユーザーに `gcloud auth login` を推奨する。
- その他のツールが不在 → 続行。影響を受けるステップは graceful degrade。

**`gcloud` 認証の推奨:**
```
⚠ gcloud が未認証です。
  認証すると GCP インフラ構築・デプロイまで自動化できます。
  今すぐ認証しますか？

  (a) はい — gcloud auth login を実行 [推奨]
  (b) いいえ — ファイル生成のみで進める (デプロイは手動)
```

---

## Phase 1: Auto-Detect

プロジェクトのファイルと状態をスキャンし、各ステップの完了状況を判定する。

### 検出手順

以下を順番に実行:

1. **`firebase-applet-config.json` を読む** (存在すれば)
   - `projectId` を抽出 → `gen-lang-client-*` パターンなら AI Studio プロジェクトとして記録
   - `apiKey` フィールドの値を確認 → `AIzaSy` で始まるなら「未サニタイズ」
   - `firestoreDatabaseId`, `appId` も記録 (後続ステップで使用)

2. **`CLAUDE.md` の存在と内容を確認**
   - ファイルが存在し、`## Architecture` または `## Tech Stack` セクションを含む → Step 0 完了
   - 存在しない or 空 → Step 0 未完了

3. **apiKey サニタイズ状態を確認**
   - `firebase-applet-config.json` に `AIzaSy` で始まる apiKey がない → Step 1 完了
   - `.env.example` が存在し `GEMINI_API_KEY` を含む → Step 1 完了の追加条件
   - どちらか欠けている → Step 1 未完了

4. **リポジトリ状態を確認**
   - `.git` ディレクトリが存在する
   - `git remote -v` でリモートが設定されている
   - `.gitignore` に以下が含まれている: `.env`, `.env.local`, `service-account*.json`
   - `README.md` が存在しセットアップ手順を含む
   - **すべて満たす → Step 2 完了。一部欠ける → Step 2 部分完了 (不足分のみ実行)**

5. **卒業状態を確認**
   - `Dockerfile` が存在する
   - `infra/terraform/` or `infra/pulumi/` or `infra/scripts/` のいずれかが存在する
   - `.github/workflows/deploy.yml` (または類似の CD workflow) が存在する
   - `firebase.json` が存在する
   - **すべて存在 → Step 3 完了。一部のみ → Step 3 部分完了 (不足分のみ生成)**
   - 存在する Dockerfile のベースイメージが検出した runtime と一致するか確認。不一致なら警告。

6. **監視状態を確認**
   - `package.json` の dependencies に `@sentry/node` or `sentry-sdk` or `dd-trace` が含まれる
   - サーバーエントリポイント (`server.ts`, `server.js`, `main.py`) に Sentry/Datadog 初期化コードがある
   - **両方あれば完了。dependencies のみ → 部分完了。なければ未完了。**

7. **セキュリティ状態を確認**
   - `gcloud` が認証済みの場合:
     - `gcloud iam service-accounts list` → デフォルト compute SA 以外の専用 SA が存在するか
     - `gcloud secrets list` → Secret Manager にシークレットが格納されているか
     - 両方 OK → 完了。一方のみ → 部分完了。
   - `gcloud` 未認証の場合:
     - `unconfirmed` (実行推奨) として扱う

8. **AI Studio 既存インフラの検出** (`gcloud` 認証済みの場合のみ)
   - `firebase-applet-config.json` の `projectId` を使って:
     - `gcloud run services list --project=PROJECT_ID` → 既存 Cloud Run サービス
     - `gcloud firestore databases list --project=PROJECT_ID` → 既存 Firestore DB
   - 検出結果を記録し、Phase 2 で表示

### 出力フォーマット

```
=== Migration Status ===

✓ Step 0  analyze-and-document      — CLAUDE.md detected
✗ Step 1  export-from-ai-studio     — apiKey not sanitized, .env.example missing
△ Step 2  repo-initializer-google   — .git exists / .gitignore missing .env entry
✗ Step 3  graduate-from-ai-studio   — Dockerfile missing, no IaC, no CI/CD
?  Step 4  monitoring-sentry-datadog — unconfirmed (recommend execution)
?  Step 5  security-hardening-gcp    — unconfirmed (recommend execution)

AI Studio infrastructure detected:
  Project: gen-lang-client-0a1b2c3d
  Cloud Run: ai-studio-app (asia-northeast1)
  Firestore: (default) database

→ Steps 1, 2 (partial), 3, 4, 5 will be executed
```

**記号凡例:**
- ✓ = 完了 (スキップ)
- △ = 部分完了 (不足分のみ実行)
- ✗ = 未完了 (フル実行)
- ? = 未確認 (実行推奨)

---

## Phase 2: Intake Questions

未完了ステップが必要とする入力を**1回のメッセージでまとめて質問**する。完了済みステップの質問は表示しない。

**重要: 質問の表示条件は「どのステップが未完了か」ではなく「後続ステップが必要とする入力があるか」で判定する。** 例えば Step 3 が完了済みでも Step 5 が未完了なら、Firebase プロジェクト情報 (Q1) は聞く必要がある (既存インフラの hardening に必要)。

### 質問一覧

```
=== Migration Configuration ===

以下の質問に回答してください:

--- デプロイ・インフラ設定 ---

Q1. Firebase プロジェクト:
  (a) 新規作成 [推奨: 完全独立した本番環境]
  (b) AI Studio の既存プロジェクトを継続 (gen-lang-client-xxxxx)
  ← gen-lang-client-xxxxx で Cloud Run, Firestore が検出されました

Q2. IaC ツール:
  (a) Terraform [推奨: エコシステムが最も充実]
  (b) Pulumi (TypeScript) [プロジェクトと同じ言語]
  (c) gcloud + firebase CLI スクリプト [最もシンプル、追加ツール不要]

Q3. GCP リージョン: [asia-northeast1] ← Enter でデフォルト

Q3.5. GCP プロジェクト ID:              ← gcloud 認証済みの場合のみ
  現在のプロジェクト: my-project-123
  (a) このプロジェクトを使う
  (b) 別のプロジェクト ID を入力: ________
  (c) 新規プロジェクトを作成: ________

--- 監視 ---

Q4. 監視ツール:
  (a) Sentry [エラートラッキング]
  (b) Datadog [APM + メトリクス]
  (c) 両方
  (d) スキップ

--- セキュリティ ---

Q5. セキュリティ強化:
  (a) 実行する [推奨]
  (b) スキップ
```

### 表示条件

| 質問 | 表示条件 |
|------|---------|
| Q1 (Firebase project) | Step 3 未完了 OR Step 5 未完了/未確認 |
| Q2 (IaC tool) | Step 3 未完了 |
| Q3 (GCP region) | Step 3 未完了 |
| Q3.5 (GCP project ID) | `gcloud` 認証済み AND (Step 3 OR Step 3.5 OR Step 5 未完了) |
| Q4 (Monitoring) | Step 4 未完了/未確認 |
| Q5 (Security) | Step 5 未完了/未確認 |

**追加条件:**
- Q1 で既存プロジェクト選択 AND `gcloud` 認証済み → 既存インフラの情報 (Cloud Run URL, Firestore DB) を自動取得
- Q2 で terraform/pulumi 選択 AND 未インストール → 「選択したツールのインストールが別途必要です」と注記
- Q4 = スキップ → Step 4 を丸ごとスキップ
- Q5 = スキップ → Step 5 を丸ごとスキップ

**Step 2 (repo-initializer-google) の入力について:**
- `.git` が存在しない場合のみ、プロジェクト名を追加で質問する
- `.git` が既にある場合は `.gitignore` / README の補完のみで、追加質問なし

---

## Phase 3: Plan Presentation

Phase 2 の回答を元に実行計画を表示。ユーザーの `y` で実行開始。

```
=== Migration Plan ===

Target: Cloud Run (asia-northeast1) / Terraform / Sentry
Firebase: New project
GCP Project: my-project-123
Automation: FULL (gcloud ✓ / firebase ✓)

Step 1  export-from-ai-studio
        → firebase-applet-config.json apiKey サニタイズ
        → Firebase init コード → 環境変数注入に変更
        → .env.example 生成

Step 2  repo-initializer-google (部分実行)
        → .gitignore: .env, service-account*.json 追加
        → README: セットアップ手順追記

Step 3  graduate-from-ai-studio [ファイル生成]
        → Dockerfile + .dockerignore (Node.js multi-stage)
        → infra/terraform/ (main.tf, variables.tf, outputs.tf, terraform.tfvars.example)
        → .github/workflows/deploy.yml (Workload Identity Federation)
        → firebase.json + .firebaserc + firestore.indexes.json
        → .env.example 更新

Step 3.5  cloud-run-deploy [GCP 自動構築 + デプロイ] ★
        → GCP API 有効化 (Cloud Run, Artifact Registry, Secret Manager)
        → Artifact Registry リポジトリ作成
        → Secret Manager にシークレット格納
        → 専用 SA 作成 + IAM 設定
        → Cloud Run に初回デプロイ
        → Firebase ルール・インデックスをデプロイ

Step 4  monitoring-sentry-datadog
        → npm install @sentry/node
        → Sentry 初期化コード挿入 (server.ts)
        → Gemini API メトリクス追加

Step 5  security-hardening-gcp [自動実行] ★
        → Firebase API Key リファラー制限
        → Budget alert 設定
        → Cloud Audit Logs 確認

生成予定ファイル数: 約 15
★ = gcloud で自動実行 (認証済み)
実行しますか？ (y/n)
```

**gcloud 未認証の場合の Plan 表示:**
```
Automation: FILE GENERATION ONLY (gcloud ✗)

Step 3.5  cloud-run-deploy [スキップ — gcloud 未認証]
        → 手動コマンドを Summary に記載

Step 5  security-hardening-gcp [コード改善のみ]
        → リモート操作は手動コマンドを Summary に記載
```

**完了済みステップは表示しない。** スキップしたステップ (Q4/Q5 でスキップ選択) も表示しない。

---

## Phase 4: Sequential Execution

承認後、各ステップを順番に実行する。

### 実行プロトコル

各ステップについて以下を行う:

1. **スキルの SKILL.md を読み込む** — `skills/<skill-name>/SKILL.md` を Read で読む
2. **スキルの検出/分析フェーズを実行** — Phase 1 の結果をキャッシュとして活用
3. **スキルの質問/承認フェーズをスキップ** — Phase 2 で回答済みの設定をそのまま使用
4. **スキルの生成フェーズを実行** — SKILL.md の手順に従ってファイルを生成
5. **進捗を報告** — 生成したファイルをリスト表示

### 各ステップの実行詳細

**Step 0: analyze-and-document** (未完了の場合のみ)
- `skills/analyze-and-document/SKILL.md` を Read
- Phase 1 (自動スキャン) → Phase 3 (CLAUDE.md 生成) を実行
- Phase 2 (ユーザー確認) は最小限: 検出結果を表示し、重大な誤りがなければ自動で進める

**Step 1: export-from-ai-studio** (未完了の場合のみ)
- `skills/export-from-ai-studio/SKILL.md` を Read
- `firebase-applet-config.json` の apiKey をプレースホルダーに置換
- Firebase 初期化コードを環境変数注入に変更
- `.env.example` を生成/更新
- 言語は `package.json` (Node.js) or `requirements.txt` (Python) から自動検出

**Step 2: repo-initializer-google** (未完了/部分完了の場合)
- `skills/repo-initializer-google/SKILL.md` を Read
- Phase 1 で検出した不足項目のみ実行:
  - `.gitignore` に不足エントリがあれば追加
  - README にセットアップ手順がなければ追記
  - `.git` がなく `gh` が使える場合: リポ作成 + 初回 push
  - `.git` があるが remote がない場合: remote 追加のみ

**Step 3: graduate-from-ai-studio** (未完了/部分完了の場合)
- `skills/graduate-from-ai-studio/SKILL.md` を Read
- Phase 2 の回答を渡す:
  - Firebase project strategy → Q1 の回答
  - IaC tool → Q2 の回答
  - GCP region → Q3 の回答
- SKILL.md の Phase 1 (分析) は wizard の Phase 1 結果を使用
- SKILL.md の Phase 2, 3 (質問, 承認) をスキップ
- SKILL.md の Phase 3.5 (apiKey サニタイズ) は Step 1 で実行済みならスキップ
- SKILL.md の Phase 4 (ファイル生成) を実行
- 部分完了の場合: 不足ファイルのみ生成

**Step 3.5: cloud-run-deploy — GCP インフラ構築 + 初回デプロイ** (`gcloud` 認証済みの場合のみ)

> **このステップが「手動作業」を自動化する核心。** `cloud-run-deploy` スキルの実行能力を使い、Step 3 で生成したファイルを使って実際に GCP 上にインフラを構築しデプロイする。

- `skills/cloud-run-deploy/SKILL.md` を Read
- **前提:** Step 3 で Dockerfile, IaC ファイル, firebase.json が生成済み
- **GCP project ID:** Q3.5 の回答を使用

実行手順 (`gcloud` 認証済みの場合):

1. **GCP プロジェクト設定:**
   ```
   gcloud config set project PROJECT_ID
   ```
   - Q1 = 新規 AND Q3.5 = 新規作成 の場合:
     ```
     gcloud projects create PROJECT_ID
     gcloud billing projects link PROJECT_ID --billing-account=BILLING_ACCOUNT
     firebase projects:addfirebase PROJECT_ID
     ```

2. **API 有効化:**
   ```
   gcloud services enable \
     run.googleapis.com \
     artifactregistry.googleapis.com \
     secretmanager.googleapis.com \
     cloudbuild.googleapis.com \
     iam.googleapis.com
   ```

3. **Artifact Registry リポジトリ作成:**
   ```
   gcloud artifacts repositories create SERVICE_NAME \
     --repository-format=docker \
     --location=REGION
   ```

4. **Secret Manager にシークレット格納:**
   - GEMINI_API_KEY の値をユーザーに確認 (`.env` または `.env.local` から読み取り)
   - 値が取得できない場合はユーザーに入力を求める
   ```
   echo -n "VALUE" | gcloud secrets create gemini-api-key --data-file=-
   ```
   - FIREBASE_API_KEY も同様 (存在する場合)

5. **専用サービスアカウント作成 + IAM 設定:**
   ```
   gcloud iam service-accounts create SERVICE_NAME-sa \
     --display-name="SERVICE_NAME Cloud Run SA"

   gcloud projects add-iam-policy-binding PROJECT_ID \
     --member=serviceAccount:SERVICE_NAME-sa@PROJECT_ID.iam.gserviceaccount.com \
     --role=roles/secretmanager.secretAccessor

   gcloud projects add-iam-policy-binding PROJECT_ID \
     --member=serviceAccount:SERVICE_NAME-sa@PROJECT_ID.iam.gserviceaccount.com \
     --role=roles/datastore.user
   ```

6. **Cloud Run にデプロイ:**
   ```
   gcloud run deploy SERVICE_NAME \
     --source . \
     --region REGION \
     --service-account SERVICE_NAME-sa@PROJECT_ID.iam.gserviceaccount.com \
     --set-secrets="GEMINI_API_KEY=gemini-api-key:latest" \
     --allow-unauthenticated
   ```

7. **Firebase デプロイ** (`firebase` CLI がある場合):
   ```
   firebase use PROJECT_ID
   firebase deploy --only firestore:rules,firestore:indexes
   ```

8. **デプロイ確認:**
   ```
   gcloud run services describe SERVICE_NAME --region REGION --format='value(status.url)'
   curl -s SERVICE_URL/health
   ```

`gcloud` 未認証の場合: Step 3.5 は丸ごとスキップ。上記コマンドを Summary の手動作業リストに集約。

**Step 4: monitoring-sentry-datadog** (未完了/未確認 AND スキップ未選択の場合)
- `skills/monitoring-sentry-datadog/SKILL.md` を Read
- Q4 の回答に基づいて Sentry / Datadog / 両方を設定
- SDK インストール (`npm install` or `pip install`)
- サーバーエントリポイントに初期化コード挿入
- `.env.example` に DSN/API key を追加
- **DSN/API key の値自体はユーザーが後で設定** → Summary の手動作業に含める
- Step 3.5 でデプロイ済みの場合: 監視コード追加後に再デプロイが必要 → Summary に案内

**Step 5: security-hardening-gcp** (未完了/未確認 AND スキップ未選択の場合)
- `skills/security-hardening-gcp/SKILL.md` を Read
- `gcloud` 認証済みの場合 — 以下を**自動実行**:
  - **API Key 制限** (Firebase Web API Key にリファラー制限):
    ```
    gcloud services api-keys update KEY_ID \
      --allowed-referrers="SERVICE_URL/*" \
      --api-target=service=firestore.googleapis.com \
      --api-target=service=identitytoolkit.googleapis.com
    ```
  - **Budget alert 設定:**
    ```
    gcloud billing budgets create \
      --billing-account=BILLING_ACCOUNT \
      --display-name="SERVICE_NAME monthly budget" \
      --budget-amount=50USD \
      --threshold-rule=percent=0.5 \
      --threshold-rule=percent=0.9
    ```
  - **Cloud Audit Logs 確認:**
    ```
    gcloud projects get-iam-policy PROJECT_ID --format=json | grep -c auditLogConfigs
    ```
  - Q1 で既存プロジェクト選択 → 既存インフラに対して hardening
  - Q1 で新規プロジェクト選択 → Step 3.5 で既にセキュア構成済み (SA, Secret Manager)。追加で API Key 制限と Budget alert のみ。
  - **注意:** Step 3.5 で SA 作成・Secret Manager 設定済みなので重複しない。Step 5 は Step 3.5 がカバーしない項目 (API Key 制限, Budget, Audit Logs) に集中。
- `gcloud` 未認証の場合:
  - コード内のセキュリティ改善のみ実行
  - リモート操作コマンドを Summary に集約

### エラーハンドリング

- **ファイル生成の失敗** → エラーを表示し、そのステップをスキップして次へ進む。Summary で未完了として報告。
- **`gcloud` / `firebase` CLI 未認証** → リモート操作はスキップ。手動コマンドを Summary に集約。
- **既存ファイルとの衝突** → 上書き前に diff を表示し、マージするか確認する。**これが唯一の実行中対話。**
- **npm install / pip install の失敗** → 警告を出して続行。Summary に手動インストールコマンドを記載。

### 進捗表示フォーマット

**gcloud 認証済み (フル自動化モード):**
```
[1/7] export-from-ai-studio ...
      ✓ firebase-applet-config.json apiKey → プレースホルダー化
      ✓ src/lib/firebase.ts → 環境変数注入に変更
      ✓ .env.example 生成

[2/7] repo-initializer-google (部分実行) ...
      ✓ .gitignore 更新 (.env, service-account*.json 追加)
      ✓ README.md セットアップ手順追記
      — GitHub repo 作成: スキップ (既存)

[3/7] graduate-from-ai-studio [ファイル生成] ...
      ✓ Dockerfile + .dockerignore
      ✓ infra/terraform/ (4 files)
      ✓ .github/workflows/deploy.yml
      ✓ firebase.json + .firebaserc + firestore.indexes.json
      ✓ .env.example 更新

[4/7] cloud-run-deploy [GCP 自動構築] ...
      ✓ GCP API 有効化 (run, artifactregistry, secretmanager, cloudbuild, iam)
      ✓ Artifact Registry リポジトリ作成: asia-northeast1-docker.pkg.dev/PROJECT/SERVICE
      ✓ Secret Manager: gemini-api-key 作成
      ✓ SA 作成: SERVICE-sa@PROJECT.iam.gserviceaccount.com
      ✓ IAM: secretmanager.secretAccessor, datastore.user 付与
      ✓ Cloud Run デプロイ: https://SERVICE-xxxxx-an.a.run.app
      ✓ Firebase デプロイ: firestore rules + indexes

[5/7] monitoring-sentry-datadog ...
      ✓ npm install @sentry/node
      ✓ Sentry 初期化コード挿入 (server.ts)
      ✓ Gemini API メトリクス追加

[6/7] security-hardening-gcp [自動実行] ...
      ✓ Firebase API Key リファラー制限設定
      ✓ Budget alert 作成: $50/月
      ✓ Cloud Audit Logs 確認: 有効

[7/7] デプロイ確認 ...
      ✓ Health check: https://SERVICE-xxxxx-an.a.run.app/health → 200 OK
```

**gcloud 未認証 (ファイル生成モード):**
```
[1/5] export-from-ai-studio ...
      ✓ (同上)

[2/5] repo-initializer-google (部分実行) ...
      ✓ (同上)

[3/5] graduate-from-ai-studio [ファイル生成] ...
      ✓ (同上)

[4/5] monitoring-sentry-datadog ...
      ✓ (同上)

[5/5] security-hardening-gcp [コード改善のみ] ...
      ✓ コード内セキュリティ改善完了
      ⚠ gcloud 未認証 — GCP 操作は手動コマンドとして Summary に記載
```

---

## Phase 5: Summary & Next Steps

全ステップ完了後、生成ファイル一覧と手動作業リストを表示する。

### 出力フォーマット — フル自動化モード (gcloud 認証済み)

```
=== Migration Complete ===

生成・変更したファイル:
  Step 1: export-from-ai-studio
    ✓ firebase-applet-config.json (apiKey サニタイズ)
    ✓ src/lib/firebase.ts (環境変数注入)
    ✓ .env.example

  Step 2: repo-initializer-google
    ✓ .gitignore (更新)
    ✓ README.md (追記)

  Step 3: graduate-from-ai-studio
    ✓ Dockerfile + .dockerignore
    ✓ infra/terraform/ (main.tf, variables.tf, outputs.tf, terraform.tfvars.example)
    ✓ .github/workflows/deploy.yml
    ✓ firebase.json + .firebaserc + firestore.indexes.json
    ✓ .env.example (更新)

GCP 構築済み:
  Step 3.5: cloud-run-deploy
    ✓ API 有効化: run, artifactregistry, secretmanager, cloudbuild, iam
    ✓ Artifact Registry: asia-northeast1-docker.pkg.dev/PROJECT/SERVICE
    ✓ Secret Manager: gemini-api-key
    ✓ SA: SERVICE-sa@PROJECT.iam.gserviceaccount.com
    ✓ Cloud Run: https://SERVICE-xxxxx-an.a.run.app
    ✓ Firebase: firestore rules + indexes deployed

  Step 4: monitoring-sentry-datadog
    ✓ package.json (@sentry/node 追加)
    ✓ server.ts (Sentry 初期化挿入)

  Step 5: security-hardening-gcp
    ✓ Firebase API Key リファラー制限
    ✓ Budget alert: $50/月
    ✓ Cloud Audit Logs 確認

⚠ 残りの手動作業 (CI/CD 用):
  1. Workload Identity Federation 設定:
     → GitHub Actions から GCP に認証するために必要
     https://github.com/google-github-actions/auth#workload-identity-federation

  2. GitHub Secrets 設定:
     gh secret set WIF_PROVIDER --body "..."
     gh secret set WIF_SERVICE_ACCOUNT --body "SERVICE-sa@PROJECT.iam.gserviceaccount.com"
     gh secret set GCP_PROJECT_ID --body "PROJECT"

  3. 監視の認証情報 (Sentry DSN 等):
     gh secret set SENTRY_DSN --body "https://xxx@sentry.io/yyy"

  4. API Key ローテーション (推奨):
     firebase-applet-config.json の apiKey が git history に残っている場合、
     Google Cloud Console で API Key を再発行してください。

  5. CI/CD 初回テスト:
     git push origin main → GitHub Actions が自動デプロイ

=== サービス稼働中 ===

  URL: https://SERVICE-xxxxx-an.a.run.app
  Health: https://SERVICE-xxxxx-an.a.run.app/health → 200 OK

=== Next: Development Environment ===

Claude Code の開発体験を整えるには:
→ 「開発環境セットアップして」 or 「setup dev environment」
  (setup-dev-environment skill: rules, hooks, permissions を .claude/ に構築)
```

### 出力フォーマット — ファイル生成モード (gcloud 未認証)

```
=== Migration Complete (ファイル生成のみ) ===

生成・変更したファイル:
  (上記と同じ Step 1-3 のファイルリスト)

  Step 4: monitoring-sentry-datadog
    ✓ package.json (@sentry/node 追加)
    ✓ server.ts (Sentry 初期化挿入)

  Step 5: security-hardening-gcp
    ✓ コード内セキュリティ改善

⚠ 手動作業 (gcloud 認証後に実行):
  1. GCP 認証:
     gcloud auth login

  2. GCP プロジェクト設定:
     gcloud config set project YOUR_PROJECT_ID

  3. GCP API 有効化:
     gcloud services enable run.googleapis.com artifactregistry.googleapis.com \
       secretmanager.googleapis.com cloudbuild.googleapis.com iam.googleapis.com

  4. Artifact Registry リポジトリ作成:
     gcloud artifacts repositories create SERVICE --repository-format=docker --location=REGION

  5. Secret Manager にシークレット格納:
     echo -n "YOUR_KEY" | gcloud secrets create gemini-api-key --data-file=-

  6. SA 作成 + IAM 設定:
     gcloud iam service-accounts create SERVICE-sa --display-name="SERVICE Cloud Run SA"
     gcloud projects add-iam-policy-binding PROJECT \
       --member=serviceAccount:SERVICE-sa@PROJECT.iam.gserviceaccount.com \
       --role=roles/secretmanager.secretAccessor
     gcloud projects add-iam-policy-binding PROJECT \
       --member=serviceAccount:SERVICE-sa@PROJECT.iam.gserviceaccount.com \
       --role=roles/datastore.user

  7. Cloud Run デプロイ:
     gcloud run deploy SERVICE --source . --region REGION \
       --service-account SERVICE-sa@PROJECT.iam.gserviceaccount.com \
       --set-secrets="GEMINI_API_KEY=gemini-api-key:latest" \
       --allow-unauthenticated

  8. Firebase デプロイ:
     firebase use PROJECT
     firebase deploy --only firestore:rules,firestore:indexes

  9. セキュリティ強化:
     (API Key 制限, Budget alert — security-hardening-gcp スキルを個別実行)

  10. CI/CD 設定:
      (WIF, GitHub Secrets — 上記フル自動化モードの手動作業 1-3 と同じ)

=== Next: Development Environment ===

Claude Code の開発体験を整えるには:
→ 「開発環境セットアップして」 or 「setup dev environment」
```

### 手動作業リストの動的生成ルール

| 条件 | 手動作業に含める |
|------|----------------|
| `gcloud` 認証済み | WIF + GitHub Secrets + 監視認証情報 + apiKey ローテーション のみ |
| `gcloud` 未認証 | GCP 構築の全コマンド (API 有効化 → AR → SM → SA → Cloud Run → Firebase) |
| Q4 = Sentry | Sentry DSN の GitHub Secrets 設定 |
| Q4 = Datadog | Datadog API Key の GitHub Secrets 設定 |
| Q4 = スキップ | 監視関連なし |
| apiKey が git history に存在 | API Key ローテーション手順 |
| WIF 未設定 | WIF 設定ドキュメントへのリンク (常に含める — GitHub 側設定は自動化不可) |

---

## Common Mistakes

- **`gcloud` 未認証のまま進む** → ファイル生成モードになり、GCP 構築・デプロイが全部手動になる。Phase 0 で認証を強く推奨。
- **Step 0 (CLAUDE.md) なしで進む** → Step 3 のフレームワーク検出精度が下がる。CLAUDE.md があると情報が補完される。
- **既存ファイルの衝突で「上書き」を安易に選ぶ** → 既にカスタマイズされた Dockerfile や workflow を壊す可能性。diff を確認すること。
- **WIF と GitHub Secrets を設定しない** → フル自動化モードでも CI/CD は WIF がないと動かない。Summary の手動作業は必ず実行すること。
- **apiKey ローテーションを忘れる** → git history に残った apiKey は取り消せない。Google Cloud Console で再発行が必要。
- **Step 3.5 で GEMINI_API_KEY の値を確認せずに進む** → Secret Manager に空の値が入る。`.env` / `.env.local` に正しい値があることを事前に確認。
- **監視コード追加後の再デプロイを忘れる** → Step 4 で Sentry/Datadog を追加した場合、Cloud Run を再デプロイしないと反映されない。
