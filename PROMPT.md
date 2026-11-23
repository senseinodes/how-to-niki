# AI エージェント向け構築指示プロンプト集 (`PROMPT.md`)

このファイルのプロンプトをコピーし、`{{...}}` の部分を自分の環境に合わせて書き換えて AI コーディングエージェント（Claude Code / Antigravity / Cursor 等）に渡してください。

一度に全 Phase を作らせるのではなく、**1 Phase ずつ実装 → テスト合格 → 次 Phase** と進めるのが最も確実に動作します。

---

## 0. 初回キックオフ（設計確認・環境ヒアリング用）

まずは仕様書全体を読み込ませ、前提を揃えます。

```text
`how-to-niki/` にある仕様書一式（`README.md` と `specs/01`〜`06`）をすべて読んでください。
この仕様に基づき、現在のリポジトリに私専用の自律型 PKM & トリアージシステムを構築したいです。

私の環境パラメータは以下の通りです:
- PROJECT_NAME: {{PROJECT_NAME}}
- USER_ID: {{USER_ID}} (human:{{USER_ID}})
- LINUX_USER: {{LINUX_USER}}
- VAULT_NAME: {{VAULT_NAME}}
- PKM_ROOT: {{PKM_ROOT}}
- REPO_DIR: {{REPO_DIR}}
- INFERENCE_HOST: {{INFERENCE_HOST}}（ホスト名/IP のみ）
- APP_HOST: {{APP_HOST}}（ホスト名/IP のみ）
- TIMEZONE: {{TIMEZONE}}
- PRIMARY_LANG: {{PRIMARY_LANG}}
- PRIVATE_CATEGORY: {{PRIVATE_CATEGORY}}
- TOPOLOGY: {{A: 2台構成 / B: 1台完結}}
- CATEGORIES: {{標準構成 / カスタム一覧}}
- OUTPUT_DIRS: {{標準構成 / カスタム一覧}}
- ENABLE_SHARE: {{不要 / はてブ / X / 両方}}

まずコードは書かず、以下を提示してください:
1. この環境パラメータに合わせた全体ディレクトリ構成と技術選定の差分（あれば）
2. Phase 1〜5 の実装計画と、各 Phase で私（人間）側が行うべきハードウェア・外部設定作業
3. 確認しておきたい不明点があれば質問リスト
```

---

## 1. Phase 1: PKM バンドル骨格・OKF スキーマ・決定論的ガーディアン

```text
Phase 1 を実装してください。
参照仕様:
- `how-to-niki/specs/01-architecture.md`（Do's & Don'ts を厳守）
- `how-to-niki/specs/02-okf-profile-and-vault.md`
- `how-to-niki/specs/03-guardians-and-pipeline.md`（`pkm_config.py`, `okf_v03.py`, `gate_okf.py`, `lint_wiki.py`, `verify_note.py`, `autocommit.py`）
- `how-to-niki/specs/06-implementation-and-tests.md`（Phase 1 の DoD とテスト項目）

実装物:
1. `pyproject.toml`（uv 管理・バージョン固定）および `.github/workflows/test.yml`
2. `pkm/` バンドルテンプレート一式（ディレクトリ骨格、各階層の `index.md`、`log.md`、`.gitignore`、`.stignore`、`AGENTS.md`、`00_System/specs/okf_local_profile.md`）
3. `pkm/00_System/schemas/okf_v03.py`
4. `pkm/00_System/scripts/` の `pkm_config.py`, `gate_okf.py`, `lint_wiki.py`, `verify_note.py`, `autocommit.py`
5. `tests/schemas/` と `tests/scripts/` の単体テスト一式

完了後、`uv run pytest` を実行し、全テストが通ることを確認してから報告してください。
```

---

## 2. Phase 2: 推論サーバー基盤（llama-swap + llama.cpp + whisper.cpp）

```text
Phase 2 を実装してください。
参照仕様:
- `how-to-niki/specs/05-inference-and-deploy.md`（推論サーバー節）
- `how-to-niki/specs/03-guardians-and-pipeline.md`（`gemma_client.py`）
- `how-to-niki/specs/06-implementation-and-tests.md`（Phase 2 の DoD）

実装物:
1. 私のトポロジー（{{TOPOLOGY}}）に合わせた推論サーバー設定一式（`deploy/inference/` 配下に `config.yaml`、常駐ユニット `plist` または `systemd`、`smoke_test.sh`、`README.md`）
   - アイドル自動アンロード（`ttl: 300`）と Parallel Group（LLM swap、Embedding/Whisper 並行稼働）を含めること
2. `pkm/00_System/scripts/gemma_client.py`（チャット・768次元 Embedding 接頭辞・`gate_okf.py` 自己修復ループ）
3. `tests/deploy/` の設定検証テストと `tests/scripts/test_gemma_client.py`

完了後、`uv run pytest` を実行して全テスト合格を確認し、推論ノードで人間が実行すべきセットアップコマンド一覧をまとめてください。
```

---

## 3. Phase 3: 全層統合検索ビルダー・検索 CLI・タグ統計・ナレッジグラフ

```text
Phase 3 を実装してください。
参照仕様:
- `how-to-niki/specs/03-guardians-and-pipeline.md`（`build_index.py`, `search.py`, `tag_stats.py`, `run_graphify.py`）
- `how-to-niki/specs/06-implementation-and-tests.md`（Phase 3 の DoD と Gotchas）

実装物:
1. `pkm/00_System/scripts/build_index.py`（MarkItDown サイドカー抽出・SQLite FTS5 trigram + sqlite-vec 差分同期・全階層 `index.md` 生成・`gate_okf.record_lineage` による lineage 記録・`log.md` 生成）
2. `pkm/00_System/scripts/search.py`（RRF ハイブリッド検索・トラストティア重み付け・3文字未満 LIKE フォールバック・再レビュー待ち一覧）
3. `pkm/00_System/scripts/tag_stats.py`（期間別タグランキングと前期比増減）
4. `pkm/00_System/scripts/run_graphify.py`（ゲート合格 Silver/Gold 限定の決定論 `graph.json` 生成＋任意 Graphify CLI 連携）
5. 対応する `tests/scripts/` のテスト一式（モックの extractor / embedder を用いてオフラインで完結すること）

完了後、`uv run pytest` で全テスト合格を確認してください。
```

---

## 4. Phase 4: トリアージ Web アプリ（`rss-triage-app`）＆ 自律 Silver 昇格ワーカー（`hermes.py`）

```text
Phase 4 を実装してください。
参照仕様:
- `how-to-niki/specs/01-architecture.md`（Do's & Don'ts: Vanilla CSS + HTMX + Jinja2、Node.js ビルド禁止、3DB 分離）
- `how-to-niki/specs/03-guardians-and-pipeline.md`（`hermes.py`）
- `how-to-niki/specs/04-triage-web-app.md`（全モジュール・全5画面・PWA・Share・AI セレクタ・Podcast 遅延文字起こし）
- `how-to-niki/specs/06-implementation-and-tests.md`（Phase 4 の DoD）

実装物:
1. `pkm/00_System/scripts/hermes.py` と `tests/scripts/test_hermes.py`
2. `apps/rss_triage/` 一式（`Dockerfile`, `README.md`, `rss_triage/*.py`, `templates/*.html`, `static/app.css`, `static/app.js`, `static/manifest.webmanifest`, `static/sw.js`, `static/icon.svg`, `static/vendor/htmx.min.js`）
   - CSS フレームワークや JS ビルドツールは使わず、ライト/ダーク対応・768px 未満のモバイル1ペイン切替・キーボードショートカットを実装すること
   - `pkm.sqlite` は読み取り専用で開き、アプリ状態は別ファイル `rss_triage.sqlite` に保存すること
   - 「私的」ソースの Keep・文字起こしは `10_Inbox/19_private/` に保存し、Hermes・Share 側の隔離（04 §3.6–3.8）も実装すること
3. `tests/apps/` のテスト一式（コア・Web ルート・全5画面・非 RSS サイト・Podcast 遅延文字起こし・Share 保護・PWA）

完了後、`uv run pytest` で全テスト合格を確認してください。
```

---

## 5. Phase 5: 常駐サーバーデプロイ構成（Docker Compose・systemd・SearXNG・Open WebUI）

```text
Phase 5 を実装してください。
参照仕様:
- `how-to-niki/specs/05-inference-and-deploy.md`（常駐サーバー・Docker Compose・systemd・同期・バックアップ）
- `how-to-niki/specs/06-implementation-and-tests.md`（Phase 5 の DoD）

実装物:
1. `docker-compose.yml`（`rss-triage-app`, `hermes-agent`, `searxng`, `open-webui`。`tools` プロファイル分離、バンドル別プロジェクト対応）
2. `deploy/server/` 配下:
   - `pkm.env.example`
   - `searxng/settings.yml`（JSON API 有効化）
   - `systemd/` ユニット一式（**すべて `@` テンプレート**: `build-index@`, `lint-wiki@`, `autocommit@`, `graphify@`, `compose@`。`/srv/%i` と `/etc/{{PROJECT_NAME}}/%i.env` を使う）
   - `README.md`（固定マウント・バンドル展開と `git init`・スクリプト更新時のバンドルへのコピー・Python/uv 環境・Docker 公式リポジトリ・systemd 有効化・Syncthing/Git 運用・スリープ無効化手順）
3. `tests/deploy/` の Compose / 設定整合性テスト（`pkm.env.example` の全変数が Compose でコンテナへ渡されていること、全 systemd ユニットがテンプレートであることを含む）
4. ルート `README.md` の最終更新

完了後、`uv run pytest` で全テスト合格を確認し、実機デプロイのチェックリストを出力してください。
```
