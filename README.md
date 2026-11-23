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

| プレースホルダー | 意味 | 既定値 / 例 |
| :--- | :--- | :--- |
| `{{PROJECT_NAME}}` | 開発リポジトリ名・アプリ表示名 | `my-pkm`（例: `niki`） |
| `{{USER_ID}}` | 人間アクター ID（`human:{{USER_ID}}` に使用） | `alice`（英小文字・数字・`-`・`_`） |
| `{{LINUX_USER}}` | 常時稼働サーバーの OS ユーザー名 | `alice` |
| `{{VAULT_NAME}}` | PKM バンドル名・Obsidian Vault 名 | `pkm` |
| `{{PKM_ROOT}}` | サーバー上の PKM バンドル配置先 | `/srv/pkm` |
| `{{REPO_DIR}}` | サーバー上の開発リポジトリ配置先 | `/opt/{{PROJECT_NAME}}` |
| `{{INFERENCE_HOST}}` | 推論サーバーのホスト名または IP（Tailscale 推奨） | `http://gpu-node:8080` |
| `{{APP_HOST}}` | Web アプリ稼働サーバーのホスト名または IP | `http://server-node:8000` |
| `{{TIMEZONE}}` | タイムゾーン | `Asia/Tokyo` |
| `{{PRIMARY_LANG}}` | 主言語（プロンプト・Whisper・SearXNG の既定） | `ja` |
| `{{PRIVATE_CATEGORY}}` | Git / lineage / Share から除外する私的カテゴリ | `28_me`（不要なら無効化可） |
| `{{CATEGORIES}}` | `20_Literature/` 配下のカテゴリ構成（21〜29） | [02-okf-profile-and-vault.md](specs/02-okf-profile-and-vault.md) の標準8カテゴリ |
| `{{OUTPUT_DIRS}}` | `60_Output/` 配下の成果物フォルダ | `61_youtube`, `62_note`（例: `61_blog`, `62_slides` 等） |
| `{{TOPOLOGY}}` | ハードウェア構成パターン | `A: 常時稼働Linux + GPU/Mac推論機` / `B: GPU付きLinux 1台完結` / `C: クラウドAPI併用` |
| `{{ENABLE_SHARE}}` | 外部共有（はてなブックマーク / X）を使うか | `不要` / `はてブのみ` / `両方` |

### ステップ 2: AI エージェントに構築を依頼する

1. 新規の空リポジトリ（例: `{{PROJECT_NAME}}/`）を用意し、この `how-to-niki/` ディレクトリを参照できる状態にします。
2. [PROMPT.md](PROMPT.md) のプロンプトに上記パラメータを記入し、AI エージェントに投入します。
3. **Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5** の順に、各 Phase のテストと完了条件（DoD）を確認しながら段階的に進めてください。

---

## ライセンス・利用方針

本仕様書群はテンプレート・設計リファレンスとして自由に改変・再利用できます。構築するシステムで外部サイトをスクレイピングする際は、対象サイトの `robots.txt` と利用規約を遵守してください。
