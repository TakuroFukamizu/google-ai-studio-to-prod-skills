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
| 3 | graduate-from-ai-studio **または** Vercelパス | デプロイ先に応じた構成 |
| 4 | monitoring-sentry-datadog | 監視・エラートラッキング |
| 5 | security-hardening-gcp | セキュリティ強化 |

**Step 3 の代替: Vercel デプロイパス** (デプロイ先=Vercel 選択時のみ)

| Sub-Step | Skill | やること |
|:--------:|-------|---------|
| 3a | vercel-nextjs-config | next.config.ts 修正 + vercel.json 生成 |
| 3b | vercel-link | Vercel プロジェクトにリンク |
| 3c | vercel-env | 環境変数設定 (GEMINI_API_KEY 等) |
| 3d | vercel-deploy | プレビュー → 本番デプロイ |
| 3e | vercel-firebase-auth-domain | Firebase Auth authorized domain に Vercel URL を追加 |

**ナレッジスキル** (直接実行ステップではなく、フロー実行時に参照される):

| Skill | 参照タイミング |
|-------|--------------|
| vercel-ai-studio-export | Step 1 (export-from-ai-studio) 実行時: AI Studio エクスポートの構造理解のため |
| vercel-gcp-project-identification | Step 3e (vercel-firebase-auth-domain) 実行時: 正しい GCP プロジェクトの特定のため |

## When to Use

- User says "本番に移行して", "migrate to production", "ワンパスでマイグレーション"
- AI Studio で作ったプロジェクトを初めて本番化する
- 複数のスキルを個別に呼ぶのが面倒なとき

**When NOT to use:**
- 特定のステップだけ実行したい → 個別スキルを直接使う
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
| `vercel` | `vercel --version 2>/dev/null` | **Vercel選択時のみ必須** | Vercel 操作コマンドを Summary に集約 |

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
- `gcloud` 未認証 → 続行。リモート操作 (SA 作成, Secret Manager, API key 制限等) はすべて手動コマンドとして Phase 5 Summary に集約。
- その他のツールが不在 → 続行。影響を受けるステップは graceful degrade。

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

5. **卒業状態を確認** (Cloud Run パスの場合)
   - `Dockerfile` が存在する
   - `infra/terraform/` or `infra/pulumi/` or `infra/scripts/` のいずれかが存在する
   - `.github/workflows/deploy.yml` (または類似の CD workflow) が存在する
   - `firebase.json` が存在する
   - **すべて存在 → Step 3 完了。一部のみ → Step 3 部分完了 (不足分のみ生成)**
   - 存在する Dockerfile のベースイメージが検出した runtime と一致するか確認。不一致なら警告。

   **Vercel パスの場合の追加チェック:**
   - `.vercel/` ディレクトリが存在するか → `vercel link` 済みかどうか
   - `vercel.json` が存在するか
   - `next.config.ts` または `next.config.js` に `output: 'standalone'` が含まれていないことの確認 (Vercel では不要)

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

--- デプロイ先 ---

Q0. デプロイ先を選択してください:
  (a) Cloud Run [GCP完結、コンテナベース]
  (b) Vercel [Next.js最適化、git-push-to-deploy]
  (c) Railway [コンテナ、長時間処理向け] ← 将来拡張予定 (現在は選択不可)

--- デプロイ・インフラ設定 ---

  ▼ デプロイ先 = Cloud Run の場合:

Q1. Firebase プロジェクト:
  (a) 新規作成 [推奨: 完全独立した本番環境]
  (b) AI Studio の既存プロジェクトを継続 (gen-lang-client-xxxxx)
  ← gen-lang-client-xxxxx で Cloud Run, Firestore が検出されました

Q2. IaC ツール:
  (a) Terraform [推奨: エコシステムが最も充実]
  (b) Pulumi (TypeScript) [プロジェクトと同じ言語]
  (c) gcloud + firebase CLI スクリプト [最もシンプル、追加ツール不要]

Q3. GCP リージョン: [asia-northeast1] ← Enter でデフォルト

  ▼ デプロイ先 = Vercel の場合:

Q1-V. Firebase プロジェクト:
  (a) 新規作成 [推奨: 完全独立した本番環境]
  (b) AI Studio の既存プロジェクトを継続 (gen-lang-client-xxxxx)

Q2-V. Vercel project name: [プロジェクト名を入力]

Q3-V. Vercel scope (team名 or personal): [スコープ名を入力]

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
| Q0 (デプロイ先) | 常に表示 |
| Q1 (Firebase project) / Q1-V | Step 3 未完了 OR Step 5 未完了/未確認 |
| Q2 (IaC tool) | Step 3 未完了 **AND** デプロイ先=Cloud Run |
| Q3 (GCP region) | Step 3 未完了 **AND** デプロイ先=Cloud Run |
| Q2-V (Vercel project name) | Step 3 未完了 **AND** デプロイ先=Vercel |
| Q3-V (Vercel scope) | Step 3 未完了 **AND** デプロイ先=Vercel |
| Q4 (Monitoring) | Step 4 未完了/未確認 |
| Q5 (Security) | Step 5 未完了/未確認 |

**追加条件:**
- Q1/Q1-V で既存プロジェクト選択 AND `gcloud` 認証済み → 既存インフラの情報 (Cloud Run URL, Firestore DB) を自動取得
- Q2 で terraform/pulumi 選択 AND 未インストール → 「選択したツールのインストールが別途必要です」と注記
- Q0 で Vercel 選択 AND `vercel` CLI 未インストール → 「vercel CLI のインストールが必要です: `npm i -g vercel`」と注記
- Q4 = スキップ → Step 4 を丸ごとスキップ
- Q5 = スキップ → Step 5 を丸ごとスキップ

**Step 2 (repo-initializer-google) の入力について:**
- `.git` が存在しない場合のみ、プロジェクト名を追加で質問する
- `.git` が既にある場合は `.gitignore` / README の補完のみで、追加質問なし

---

## Phase 3: Plan Presentation

Phase 2 の回答を元に実行計画を表示。ユーザーの `y` で実行開始。

**Cloud Run パスの場合:**

```
=== Migration Plan ===

Target: Cloud Run (asia-northeast1) / Terraform / Sentry
Firebase: New project
Tools: gcloud ✓ / firebase ✓ / terraform △ (要インストール)

Step 1  export-from-ai-studio
        → firebase-applet-config.json apiKey サニタイズ
        → Firebase init コード → 環境変数注入に変更
        → .env.example 生成

Step 2  repo-initializer-google (部分実行)
        → .gitignore: .env, service-account*.json 追加
        → README: セットアップ手順追記

Step 3  graduate-from-ai-studio
        → Dockerfile + .dockerignore (Node.js multi-stage)
        → infra/terraform/ (main.tf, variables.tf, outputs.tf, terraform.tfvars.example)
        → .github/workflows/deploy.yml (Workload Identity Federation)
        → firebase.json + .firebaserc + firestore.indexes.json
        → .env.example 更新

Step 4  monitoring-sentry-datadog
        → npm install @sentry/node
        → Sentry 初期化コード挿入 (server.ts)
        → Gemini API メトリクス追加

Step 5  security-hardening-gcp
        → 専用 SA 作成 + 最小権限 IAM
        → Secret Manager 移行 (GEMINI_API_KEY, FIREBASE_API_KEY)
        → Firebase API Key リファラー制限
        → Budget alert 設定

生成予定ファイル数: 約 15
実行しますか？ (y/n)
```

**Vercel パスの場合:**

```
=== Migration Plan ===

Target: Vercel / Sentry
Vercel Project: my-app (scope: my-team)
Firebase: Existing project (gen-lang-client-xxxxx)
Tools: vercel ✓ / firebase ✓ / gcloud ✓

Step 1  export-from-ai-studio
        → firebase-applet-config.json apiKey サニタイズ
        → Firebase init コード → 環境変数注入に変更
        → .env.example 生成

Step 2  repo-initializer-google (部分実行)
        → .gitignore: .env 追加
        → README: セットアップ手順追記

Step 3a  vercel-nextjs-config
        → next.config.ts: output: 'standalone' を除去 (存在する場合)
        → vercel.json 生成 (headers, rewrites 等)

Step 3b  vercel-link
        → vercel link --project my-app --scope my-team

Step 3c  vercel-env
        → GEMINI_API_KEY, NEXT_PUBLIC_FIREBASE_* を Vercel 環境変数に設定

Step 3d  vercel-deploy
        → vercel deploy (プレビュー) → vercel deploy --prod (本番)

Step 3e  vercel-firebase-auth-domain
        → Firebase Auth authorized domains に Vercel URL を追加
        → vercel-gcp-project-identification スキルで GCP プロジェクトを特定

Step 4  monitoring-sentry-datadog
        → npm install @sentry/nextjs
        → Sentry 初期化コード挿入
        → Gemini API メトリクス追加

Step 5  security-hardening-gcp
        → Secret Manager 移行 (GEMINI_API_KEY, FIREBASE_API_KEY)
        → Firebase API Key リファラー制限
        → Budget alert 設定

生成予定ファイル数: 約 10
実行しますか？ (y/n)
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
- **`skills/vercel-ai-studio-export/SKILL.md` を Read** — AI Studio エクスポートの構造理解のためナレッジ参照
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

**Step 3: graduate-from-ai-studio** (未完了/部分完了の場合 / **デプロイ先=Cloud Run のみ**)
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

**Step 3 (Vercel パス): デプロイ先=Vercel の場合のみ実行** — graduate-from-ai-studio の代わりに以下の Sub-Step を順番に実行する:

**Step 3a: vercel-nextjs-config**
- `skills/vercel-nextjs-config/SKILL.md` を Read
- `next.config.ts` / `next.config.js` を確認:
  - `output: 'standalone'` が存在する場合は除去 (Vercel では不要)
  - Vercel 向け最適化設定を追加
- `vercel.json` を生成 (headers, rewrites, functions 設定等)
- SKILL.md の手順に従ってファイルを生成/更新

**Step 3b: vercel-link**
- `skills/vercel-link/SKILL.md` を Read
- Q2-V (Vercel project name) と Q3-V (Vercel scope) の回答を使用
- `vercel link --project <project-name> --scope <scope>` を実行
- `.vercel/` ディレクトリが既に存在する場合は再リンクの要否を確認

**Step 3c: vercel-env**
- `skills/vercel-env/SKILL.md` を Read
- Step 1 で確認した環境変数 (GEMINI_API_KEY, NEXT_PUBLIC_FIREBASE_* 等) を Vercel 環境変数に設定
- `.env.example` の内容を参照して必要な変数を特定
- vercel env add で各変数を設定 (値はユーザーが後で設定 → Summary に含める)

**Step 3d: vercel-deploy**
- `skills/vercel-deploy/SKILL.md` を Read
- `vercel deploy` でプレビューデプロイを実行
- プレビュー URL を取得・表示
- `vercel deploy --prod` で本番デプロイを実行
- 本番 URL を取得・記録 (Step 3e で使用)

**Step 3e: vercel-firebase-auth-domain**
- `skills/vercel-gcp-project-identification/SKILL.md` を Read — **ナレッジ参照**: Q1-V で選択した Firebase プロジェクトに対応する GCP プロジェクト ID を正確に特定するため
- `skills/vercel-firebase-auth-domain/SKILL.md` を Read
- Step 3d で取得した本番 URL (例: `my-app.vercel.app`) を Firebase Auth の authorized domains に追加
- `gcloud` 認証済みの場合: Firebase Console API または gcloud CLI で直接設定
- `gcloud` 未認証の場合: 手動設定コマンドを Summary に集約

**Step 4: monitoring-sentry-datadog** (未完了/未確認 AND スキップ未選択の場合)
- `skills/monitoring-sentry-datadog/SKILL.md` を Read
- Q4 の回答に基づいて Sentry / Datadog / 両方を設定
- SDK インストール (`npm install` or `pip install`)
- サーバーエントリポイントに初期化コード挿入
- `.env.example` に DSN/API key を追加
- **DSN/API key の値自体はユーザーが後で設定** → Summary の手動作業に含める

**Step 5: security-hardening-gcp** (未完了/未確認 AND スキップ未選択の場合)
- `skills/security-hardening-gcp/SKILL.md` を Read
- `gcloud` 認証済みの場合:
  - 専用 SA 作成、IAM 設定、Secret Manager 移行、API key 制限を実行
  - Q1 で既存プロジェクト選択 → 既存インフラに対して hardening
  - Q1 で新規プロジェクト選択 → 新規構成でセキュア設計 (IaC に反映済み)
- `gcloud` 未認証の場合:
  - コード内のセキュリティ改善のみ実行
  - リモート操作コマンドを Summary に集約

### エラーハンドリング

- **ファイル生成の失敗** → エラーを表示し、そのステップをスキップして次へ進む。Summary で未完了として報告。
- **`gcloud` / `firebase` CLI 未認証** → リモート操作はスキップ。手動コマンドを Summary に集約。
- **既存ファイルとの衝突** → 上書き前に diff を表示し、マージするか確認する。**これが唯一の実行中対話。**
- **npm install / pip install の失敗** → 警告を出して続行。Summary に手動インストールコマンドを記載。

### 進捗表示フォーマット

**Cloud Run パスの場合:**

```
[1/5] export-from-ai-studio ...
      ✓ firebase-applet-config.json apiKey → プレースホルダー化
      ✓ src/lib/firebase.ts → 環境変数注入に変更
      ✓ .env.example 生成

[2/5] repo-initializer-google (部分実行) ...
      ✓ .gitignore 更新 (.env, service-account*.json 追加)
      ✓ README.md セットアップ手順追記
      — GitHub repo 作成: スキップ (既存)

[3/5] graduate-from-ai-studio ...
      ✓ Dockerfile + .dockerignore
      ✓ infra/terraform/ (4 files)
      ✓ .github/workflows/deploy.yml
      ✓ firebase.json + .firebaserc + firestore.indexes.json
      ✓ .env.example 更新

[4/5] monitoring-sentry-datadog ...
      ✓ npm install @sentry/node
      ✓ Sentry 初期化コード挿入 (server.ts)
      ✓ Gemini API メトリクス追加

[5/5] security-hardening-gcp ...
      ⚠ gcloud 未認証 — リモート操作は手動コマンドとして案内
      ✓ コード内セキュリティ改善完了
```

**Vercel パスの場合:**

```
[1/7] export-from-ai-studio ...
      ✓ firebase-applet-config.json apiKey → プレースホルダー化
      ✓ src/lib/firebase.ts → 環境変数注入に変更
      ✓ .env.example 生成

[2/7] repo-initializer-google (部分実行) ...
      ✓ .gitignore 更新 (.env 追加)
      ✓ README.md セットアップ手順追記

[3a/7] vercel-nextjs-config ...
      ✓ next.config.ts: output: 'standalone' を除去
      ✓ vercel.json 生成

[3b/7] vercel-link ...
      ✓ vercel link --project my-app --scope my-team
      ✓ .vercel/project.json 生成

[3c/7] vercel-env ...
      ✓ GEMINI_API_KEY: 設定完了 (値は要設定)
      ✓ NEXT_PUBLIC_FIREBASE_API_KEY: 設定完了
      ✓ NEXT_PUBLIC_FIREBASE_PROJECT_ID: 設定完了

[3d/7] vercel-deploy ...
      ✓ vercel deploy → https://my-app-xxxx.vercel.app (プレビュー)
      ✓ vercel deploy --prod → https://my-app.vercel.app (本番)

[3e/7] vercel-firebase-auth-domain ...
      ✓ GCP プロジェクト特定: gen-lang-client-0a1b2c3d
      ✓ Firebase Auth authorized domains に my-app.vercel.app を追加

[6/7] monitoring-sentry-datadog ...
      ✓ npm install @sentry/nextjs
      ✓ Sentry 初期化コード挿入
      ✓ Gemini API メトリクス追加

[7/7] security-hardening-gcp ...
      ✓ Secret Manager 移行 (GEMINI_API_KEY, FIREBASE_API_KEY)
      ✓ Firebase API Key リファラー制限
      ✓ Budget alert 設定
```

---

## Phase 5: Summary & Next Steps

全ステップ完了後、生成ファイル一覧と手動作業リストを表示する。

### 出力フォーマット

**Cloud Run パスの場合:**

```
=== Migration Complete ===

デプロイ先: Cloud Run (asia-northeast1)

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

  Step 4: monitoring-sentry-datadog
    ✓ package.json (@sentry/node 追加)
    ✓ server.ts (Sentry 初期化挿入)

  Step 5: security-hardening-gcp
    ✓ コード内セキュリティ改善

⚠ 手動作業:
  1. GCP プロジェクト作成:
     gcloud projects create YOUR_PROJECT_ID
     firebase projects:addfirebase YOUR_PROJECT_ID

  2. Terraform 初期化:
     cd infra/terraform && terraform init && terraform plan

  3. GitHub Secrets 設定:
     - WIF_PROVIDER
     - WIF_SERVICE_ACCOUNT
     - GCP_PROJECT_ID
     - SENTRY_DSN

  4. Workload Identity Federation 設定:
     https://github.com/google-github-actions/auth#workload-identity-federation

  5. 初回デプロイ:
     git push origin main

  6. API Key ローテーション (推奨):
     firebase-applet-config.json の apiKey が git history に残っている場合、
     Google Cloud Console で API Key を再発行してください。

=== Next: Development Environment ===

Claude Code の開発体験を整えるには:
→ 「開発環境セットアップして」 or 「setup dev environment」
  (setup-dev-environment skill: rules, hooks, permissions を .claude/ に構築)
```

**Vercel パスの場合:**

```
=== Migration Complete ===

デプロイ先: Vercel
本番 URL: https://my-app.vercel.app
プレビュー URL: https://my-app-xxxx.vercel.app

生成・変更したファイル:
  Step 1: export-from-ai-studio
    ✓ firebase-applet-config.json (apiKey サニタイズ)
    ✓ src/lib/firebase.ts (環境変数注入)
    ✓ .env.example

  Step 2: repo-initializer-google
    ✓ .gitignore (更新)
    ✓ README.md (追記)

  Step 3a: vercel-nextjs-config
    ✓ next.config.ts (更新)
    ✓ vercel.json (生成)

  Step 3b: vercel-link
    ✓ .vercel/project.json (生成)

  Step 3c: vercel-env
    ✓ Vercel 環境変数: 5 件設定

  Step 3d: vercel-deploy
    ✓ 本番デプロイ完了: https://my-app.vercel.app

  Step 3e: vercel-firebase-auth-domain
    ✓ Firebase Auth authorized domains 更新

  Step 4: monitoring-sentry-datadog
    ✓ package.json (@sentry/nextjs 追加)
    ✓ Sentry 初期化コード挿入

  Step 5: security-hardening-gcp
    ✓ Secret Manager 移行
    ✓ Firebase API Key リファラー制限

⚠ 手動作業:
  1. Vercel 環境変数の値を設定:
     vercel env add GEMINI_API_KEY
     (Vercel Dashboard → Project → Settings → Environment Variables でも設定可)

  2. Sentry DSN の設定:
     vercel env add SENTRY_DSN

  3. 本番デプロイの確認:
     https://my-app.vercel.app が正常に動作しているか確認

  4. API Key ローテーション (推奨):
     firebase-applet-config.json の apiKey が git history に残っている場合、
     Google Cloud Console で API Key を再発行してください。

  5. カスタムドメイン設定 (任意):
     Vercel Dashboard → Project → Settings → Domains

=== Next: Development Environment ===

Claude Code の開発体験を整えるには:
→ 「開発環境セットアップして」 or 「setup dev environment」
  (setup-dev-environment skill: rules, hooks, permissions を .claude/ に構築)
```

### 手動作業リストの動的生成ルール

**Cloud Run パス共通:**

| 条件 | 手動作業に含める |
|------|----------------|
| Q1 = 新規プロジェクト | `gcloud projects create` + `firebase projects:addfirebase` |
| Q1 = 既存プロジェクト | プロジェクト作成はスキップ |
| Q2 = Terraform | `terraform init && terraform plan` |
| Q2 = Pulumi | `cd infra/pulumi && npm install && pulumi up` |
| Q2 = CLI スクリプト | `bash infra/scripts/setup.sh` |
| `terraform`/`pulumi` 未インストール | ツールのインストール手順 |
| Q4 = Sentry | Sentry DSN の取得 + GitHub Secrets 設定 |
| Q4 = Datadog | Datadog API Key の取得 + GitHub Secrets 設定 |
| Q4 = スキップ | 監視関連なし |
| `gcloud` 未認証 | SA 作成, Secret Manager, API key 制限のコマンド一覧 |
| apiKey が git history に存在 | API Key ローテーション手順 |
| WIF 未設定 | WIF 設定ドキュメントへのリンク |

**Vercel パス追加:**

| 条件 | 手動作業に含める |
|------|----------------|
| Q0 = Vercel | `vercel env add GEMINI_API_KEY` (値設定) |
| Q0 = Vercel AND Q4 = Sentry | `vercel env add SENTRY_DSN` |
| Q0 = Vercel AND `gcloud` 未認証 | Firebase Auth authorized domain 手動設定手順 |
| Q0 = Vercel AND カスタムドメイン希望 | Vercel Dashboard でのドメイン設定手順 |
| apiKey が git history に存在 | API Key ローテーション手順 |

---

## Common Mistakes

- **Phase 0 をスキップして `gcloud` 未認証のまま進む** → Step 3, 5 のリモート操作が全部手動になる。最初に認証を推奨。
- **Step 0 (CLAUDE.md) なしで進む** → Step 3 のフレームワーク検出精度が下がる。CLAUDE.md があると情報が補完される。
- **既存ファイルの衝突で「上書き」を安易に選ぶ** → 既にカスタマイズされた Dockerfile や workflow を壊す可能性。diff を確認すること。
- **手動作業リストを無視する** → 特に WIF と GitHub Secrets は設定しないと CI/CD が動かない。
- **apiKey ローテーションを忘れる** → git history に残った apiKey は取り消せない。Google Cloud Console で再発行が必要。
