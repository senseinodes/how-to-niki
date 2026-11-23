# how-to-niki — 自律型 PKM & トリアージパイプライン構築仕様書

このディレクトリは、**「APIコスト完全ゼロのローカル推論 × 決定論的コード制御による腐敗しない個人ナレッジベース（LLM Wiki）」** を、誰でも AI コーディングエージェント（Claude Code / Antigravity / Cursor / Codex 等）に依頼してゼロから再構築できるようにまとめた仕様書一式です。

オリジナル実装（`niki`）の設計思想・データ構造・スクリプト契約・Web アプリ・デプロイ構成・実運用で得られた注意点（Gotchas）を、個人環境固有の値を排して汎用仕様として整理しています。

---

## このシステムで実現できること

1. **API コストゼロの収集・一次トリアージ:**
   - RSS / Atom / 非 RSS サイト（AI が CSS セレクタを自動推定）/ Podcast を定期巡回。
   - ローカル LLM で記事ごとに関心スコア（0〜100）と3行要約を自動生成し、自分の Keep / Dismiss 履歴から好みを学習。
   - Podcast はショーノートのみで軽量判定し、人間が「取り込む」と選んだ回だけ配信元トランスクリプト取得またはローカル `whisper.cpp` で遅延文字起こし（Lazy Transcription）。
2. **腐敗しないメダリオン型 LLM Wiki（OKF v0.3 準拠）:**
   - **Bronze (`10_Inbox/`)**: 生データ（非 OKF）。PDF / Office / 画像は `MarkItDown` + ローカル VLM で検索用サイドカー（`.extracted.md`）を自動生成。
   - **Silver (`20_Literature/`)**: 自律ワーカー（Hermes）が検索エンジン（SearXNG）で背景調査しつつ OKF ノート化し、Pydantic ゲート（`gate_okf.py`）に合格するまで自己修復。
   - **Gold (`50_Knowledge/`)**: 人間が AI エージェントと対話し、Silver を引用して合成・人間承認（`human:<id>`）を付けた恒久知識。AI が後から編集すると自動で「再レビュー待ち」を検知し UI にバッジ表示。
   - **Output (`60_Output/`)**: Gold と人間の一次感想（`x-pkm-my-comment`）から生成した成果物（記事・台本等）。
3. **単一 SQLite 全層ハイブリッド検索 ＋ Silver/Gold 限定ナレッジグラフ:**
   - 外部の重厚なベクトル DB を使わず、単一 SQLite（`FTS5` trigram + `sqlite-vec` 768次元ベクトル）で Bronze〜Gold をトラストティア重み付きハイブリッド検索。
   - ゲート合格済みの Silver / Gold のみからナレッジグラフを生成。
4. **ビルド不要の軽量 Web UI（PWA 対応）:**
   - FastAPI + Jinja2 + HTMX + Vanilla CSS（Node.js ビルド不要）。スマホの共有メニューから直接 Web クリップできる PWA 対応。

---

## 仕様書一覧

| ファイル | 内容 | 主な参照タイミング |
| :--- | :--- | :--- |
| [PROMPT.md](PROMPT.md) | AI エージェントへそのまま渡せる構築依頼プロンプト集 | 構築開始時・各 Phase 着手時 |
| [specs/01-architecture.md](specs/01-architecture.md) | 設計思想、Do's & Don'ts、ハードウェア構成パターン、同期・バックアップ戦略 | 全体設計・前提合意 |
| [specs/02-okf-profile-and-vault.md](specs/02-okf-profile-and-vault.md) | PKM バンドル構成、OKF v0.3 Local Profile 完全スキーマ、導出値、7つのゲートルール、`AGENTS.md` 仕様 | Phase 1（基盤・ゲート） |
| [specs/03-guardians-and-pipeline.md](specs/03-guardians-and-pipeline.md) | 決定論的スクリプト群（全11本）の仕様、SQLite スキーマ、CLI・終了コード契約 | Phase 1, 3, 4 |
| [specs/04-triage-web-app.md](specs/04-triage-web-app.md) | `rss-triage-app` の全5画面、エンドポイント、DB スキーマ、クローラー、AI セレクタ、Podcast 遅延文字起こし、Share、PWA 仕様 | Phase 4（Web アプリ） |
| [specs/05-inference-and-deploy.md](specs/05-inference-and-deploy.md) | ローカル推論サーバー（`llama-swap` + `llama.cpp` + `whisper.cpp`）、Docker Compose、systemd timer、SearXNG、Open WebUI、環境変数一覧 | Phase 2, 5（インフラ・運用） |
| [specs/06-implementation-and-tests.md](specs/06-implementation-and-tests.md) | Phase 1〜5 実装ロードマップ、完了条件（DoD）、テスト仕様、ハマりやすいポイント集 | 全 Phase の実装・検証 |

---

## クイックスタート（AI エージェントへの依頼手順）

### ステップ 1: 自分の環境に合わせてパラメータを決める

以下のカスタマイズシートを埋めておくと、AI エージェントが迷わずあなたの環境向けのコード・設定を生成できます。未定の項目は既定値のままで構いません。

| プレースホルダー | 意味 | 既定値 / 例 | 主な反映先 |
| :--- | :--- | :--- | :--- |
| `{{PROJECT_NAME}}` | 開発リポジトリ名・アプリ表示名・systemd ユニット接頭辞 | `my-pkm`（例: `niki`） | 05 §2 |
| `{{USER_ID}}` | 人間アクター ID（`human:{{USER_ID}}` に使用） | `alice`（英小文字・数字・`-`・`_`） | 02 §4, 05 `NIKI_REVIEWER` |
| `{{LINUX_USER}}` | 常時稼働サーバーの OS ユーザー名（バンドル所有者・systemd の `User=`） | `alice` | 05 §2.3, `NIKI_UID/GID` |
| `{{VAULT_NAME}}` | PKM バンドル名・Obsidian Vault 名（systemd テンプレートの `%i`） | `pkm` | 05 §2 |
| `{{PKM_ROOT}}` | サーバー上の PKM バンドル配置先。**systemd テンプレートを使うため `/srv/{{VAULT_NAME}}` 固定** | `/srv/pkm` | 05 §2.1 |
| `{{REPO_DIR}}` | サーバー上の開発リポジトリ配置先 | `/opt/{{PROJECT_NAME}}` | 05 §2.1 |
| `{{INFERENCE_HOST}}` | 推論サーバーの**ホスト名または IP のみ**（スキーム・ポートは付けない。Tailscale 推奨） | `gpu-node` | `MAC_MINI_URL=http://{{INFERENCE_HOST}}:8080` |
| `{{APP_HOST}}` | Web アプリ稼働サーバーの**ホスト名または IP のみ** | `server-node` | `http://{{APP_HOST}}:8000/` |
| `{{TIMEZONE}}` | タイムゾーン（IANA 名） | `Asia/Tokyo` | 05 `TZ` |
| `{{PRIMARY_LANG}}` | 主言語。**オリジナル実装は `ja` 固定**（変える場合の箇所は 06 §3 Gotcha 15） | `ja` | Whisper・SearXNG・各プロンプト |
| `{{PRIVATE_CATEGORY}}` | Git / lineage / Share から除外する私的カテゴリ | `28_me` | 02 §2, 04 §3.6 |
| `{{CATEGORIES}}` | `20_Literature/` 配下のカテゴリ構成（21〜29。受け皿の `29_misc` は必須） | [02](specs/02-okf-profile-and-vault.md) §2.2 の標準8カテゴリ | 02 §2, `AGENTS.md` |
| `{{OUTPUT_DIRS}}` | `60_Output/` 配下の成果物フォルダ | `61_youtube`, `62_note`（例: `61_blog`, `62_slides` 等） | 02 §2 |
| `{{TOPOLOGY}}` | ハードウェア構成パターン | `A` / `B`（[01](specs/01-architecture.md) §4。`C` は参考扱い） | 01 §4, 05 §1 |
| `{{ENABLE_SHARE}}` | 外部共有（はてなブックマーク / X）を使うか | `不要` / `はてブのみ` / `両方` | 04 §3.8, 05 `HATENA_*`/`NIKI_XURL` |

### ステップ 2: AI エージェントに構築を依頼する

1. 新規の空リポジトリ（例: `{{PROJECT_NAME}}/`）を用意し、この `how-to-niki/` ディレクトリを参照できる状態にします。
2. [PROMPT.md](PROMPT.md) のプロンプトに上記パラメータを記入し、AI エージェントに投入します。
3. **Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5** の順に、各 Phase のテストと完了条件（DoD）を確認しながら段階的に進めてください。

> **2つのリポジトリ:** 開発リポジトリ（`{{PROJECT_NAME}}`。コード・テスト・テンプレート `pkm/`）と、サーバー上の PKM バンドル（`{{PKM_ROOT}}`。テンプレートを展開した**別の private Git リポジトリ**）は別物です。スクリプトを更新したらバンドル側の `00_System/` へコピーしてコミットします（[05](specs/05-inference-and-deploy.md) §2.1）。

---

## ライセンス・利用方針

本仕様書群はテンプレート・設計リファレンスとして自由に改変・再利用できます。構築するシステムで外部サイトをスクレイピングする際は、対象サイトの `robots.txt` と利用規約を遵守してください。

---

# how-to-niki — Autonomous PKM & Triage Pipeline Specification (English)

This directory contains a complete set of specifications enabling anyone to rebuild a **zero-API-cost local-inference × deterministic code-controlled, non-decaying personal knowledge base (LLM Wiki)** from scratch by handing them to an AI coding agent (Claude Code, Antigravity, Cursor, Codex, etc.).

The design philosophy, data structures, script contracts, web application, deployment architecture, and real-world operational gotchas of the original implementation (`niki`) have been stripped of personal environment values and organized as a reusable general-purpose specification.

---

## What This System Achieves

1. **Zero-API-Cost Collection & First-Pass Triage:**
   - Periodically crawls RSS / Atom feeds, non-RSS websites (with AI-inferred CSS selectors), and Podcasts.
   - Uses a local LLM to automatically generate an interest score (0–100) and a 3-line summary for each article, learning your preferences from your Keep / Dismiss history.
   - Performs lightweight triage on Podcasts using show notes only, and runs lazy transcription (fetching publisher transcripts or transcribing locally via `whisper.cpp`) exclusively for episodes you explicitly choose to ingest.
2. **Non-Decaying Medallion LLM Wiki (OKF v0.3 Compliant):**
   - **Bronze (`10_Inbox/`)**: Raw data (non-OKF). PDFs, Office files, and images automatically get searchable sidecars (`.extracted.md`) via `MarkItDown` + a local VLM.
   - **Silver (`20_Literature/`)**: An autonomous worker (Hermes) performs background research via a metasearch engine (SearXNG), converts items into OKF notes, and self-repairs until they pass the Pydantic gate (`gate_okf.py`).
   - **Gold (`50_Knowledge/`)**: Permanent knowledge synthesized through human + AI agent dialogue citing Silver notes, stamped with human approval (`human:<id>`). If an AI edits a Gold note later, the system automatically detects "review pending" and displays a badge in the UI.
   - **Output (`60_Output/`)**: Deliverables (articles, scripts, etc.) generated from Gold notes and the human's primary reactions (`x-pkm-my-comment`).
3. **Single-SQLite Full-Layer Hybrid Search + Silver/Gold-Only Knowledge Graph:**
   - Without standing up heavy external vector databases, a single SQLite file (`FTS5` trigram + `sqlite-vec` 768-dim vectors) provides trust-tier-weighted hybrid search across Bronze through Gold.
   - Generates a knowledge graph strictly from gate-passing Silver / Gold notes.
4. **Build-Free Lightweight Web UI (PWA-Enabled):**
   - FastAPI + Jinja2 + HTMX + Vanilla CSS (no Node.js build step). Supports PWA installation so you can clip web pages directly from your smartphone's share menu.

---

## Specification Index

| File | Contents | Primary Reference Timing |
| :--- | :--- | :--- |
| [en/PROMPT.md](en/PROMPT.md) | Ready-to-use build prompts to hand directly to an AI agent | Project kickoff & start of each Phase |
| [en/specs/01-architecture.md](en/specs/01-architecture.md) | Design philosophy, Do's & Don'ts, hardware topology patterns, sync & backup strategy | Overall architecture & upfront alignment |
| [en/specs/02-okf-profile-and-vault.md](en/specs/02-okf-profile-and-vault.md) | PKM bundle layout, complete OKF v0.3 Local Profile schema, derived values, 7 gate rules, `AGENTS.md` spec | Phase 1 (Foundation & Gate) |
| [en/specs/03-guardians-and-pipeline.md](en/specs/03-guardians-and-pipeline.md) | Specs for all 11 deterministic scripts, SQLite schemas, CLI & exit-code contracts | Phase 1, 3, 4 |
| [en/specs/04-triage-web-app.md](en/specs/04-triage-web-app.md) | All 5 screens of `rss-triage-app`, HTTP endpoints, DB schema, crawler, AI selectors, Podcast lazy transcription, Share, PWA spec | Phase 4 (Web App) |
| [en/specs/05-inference-and-deploy.md](en/specs/05-inference-and-deploy.md) | Local inference server (`llama-swap` + `llama.cpp` + `whisper.cpp`), Docker Compose, systemd timers, SearXNG, Open WebUI, env var reference | Phase 2, 5 (Infra & Operations) |
| [en/specs/06-implementation-and-tests.md](en/specs/06-implementation-and-tests.md) | Phase 1–5 implementation roadmap, Definition of Done (DoD), test specs, operational gotchas | Implementation & verification of all Phases |

---

## Quick Start (How to Instruct an AI Agent)

### Step 1: Choose Parameters for Your Environment

Filling out the customization sheet below lets the AI agent generate code and configuration tailored to your environment without guesswork. Leave any undecided fields at their defaults.

| Placeholder | Meaning | Default / Example | Primary Targets |
| :--- | :--- | :--- | :--- |
| `{{PROJECT_NAME}}` | Dev repository name, app display name, systemd unit prefix | `my-pkm` (e.g., `niki`) | 05 §2 |
| `{{USER_ID}}` | Human actor ID (used in `human:{{USER_ID}}`) | `alice` (lowercase letters, digits, `-`, `_`) | 02 §4, 05 `NIKI_REVIEWER` |
| `{{LINUX_USER}}` | OS username on the always-on server (bundle owner & systemd `User=`) | `alice` | 05 §2.3, `NIKI_UID/GID` |
| `{{VAULT_NAME}}` | PKM bundle name / Obsidian Vault name (`%i` in systemd templates) | `pkm` | 05 §2 |
| `{{PKM_ROOT}}` | PKM bundle path on the server. **Fixed to `/srv/{{VAULT_NAME}}` because systemd templates are used** | `/srv/pkm` | 05 §2.1 |
| `{{REPO_DIR}}` | Dev repository path on the server | `/opt/{{PROJECT_NAME}}` | 05 §2.1 |
| `{{INFERENCE_HOST}}` | Inference server **hostname or IP only** (no scheme or port; Tailscale recommended) | `gpu-node` | `MAC_MINI_URL=http://{{INFERENCE_HOST}}:8080` |
| `{{APP_HOST}}` | Web app server **hostname or IP only** | `server-node` | `http://{{APP_HOST}}:8000/` |
| `{{TIMEZONE}}` | Timezone (IANA name) | `Asia/Tokyo` | 05 `TZ` |
| `{{PRIMARY_LANG}}` | Primary language. **The original implementation hardcodes `ja`** (see 06 §3 Gotcha 15 for places to change) | `ja` | Whisper, SearXNG, LLM prompts |
| `{{PRIVATE_CATEGORY}}` | Private category excluded from Git / lineage / Share | `28_me` | 02 §2, 04 §3.6 |
| `{{CATEGORIES}}` | Category layout under `20_Literature/` (21–29; fallback `29_misc` is required) | 8 standard categories in [02](en/specs/02-okf-profile-and-vault.md) §2.2 | 02 §2, `AGENTS.md` |
| `{{OUTPUT_DIRS}}` | Deliverable folders under `60_Output/` | `61_youtube`, `62_note` (e.g., `61_blog`, `62_slides`) | 02 §2 |
| `{{TOPOLOGY}}` | Hardware topology pattern | `A` / `B` ([01](en/specs/01-architecture.md) §4; `C` is reference-only) | 01 §4, 05 §1 |
| `{{ENABLE_SHARE}}` | Whether to use external sharing (Hatena Bookmark / X) | `None` / `Hatena only` / `Both` | 04 §3.8, 05 `HATENA_*`/`NIKI_XURL` |

### Step 2: Ask an AI Agent to Build It

1. Prepare a new empty repository (e.g., `{{PROJECT_NAME}}/`) with access to this `how-to-niki/` directory.
2. Fill in the parameters above in the prompts from [en/PROMPT.md](en/PROMPT.md) and send them to your AI agent.
3. Proceed step by step in order — **Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5** — verifying each Phase's tests and Definition of Done (DoD) before moving on.

> **Two separate repositories:** The dev repository (`{{PROJECT_NAME}}`: code, tests, template `pkm/`) and the PKM bundle on the server (`{{PKM_ROOT}}`: a **separate private Git repository** initialized from the template) are distinct. Whenever you update scripts in the dev repo, copy them into `00_System/` on the bundle side and commit them there ([05](en/specs/05-inference-and-deploy.md) §2.1).

---

## License & Usage Policy

You are free to modify and reuse these specifications as templates and design references. When scraping external websites with the system you build, please comply with each target site's `robots.txt` and terms of service.
