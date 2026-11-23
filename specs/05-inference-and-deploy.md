# 05. 推論サーバー・常駐サーバー・デプロイ・環境変数仕様

---

## 1. 推論サーバー（`llama-swap` + `llama.cpp` + `whisper.cpp`）

GPU メモリ帯域を最大限に使うため、Docker ではなく **ホストネイティブ** で稼働させる（Apple Silicon Mac は Homebrew + `launchd`、Linux GPU 機はビルド/パッケージ + `systemd`）。

### 1.1 公開モデル ID とグループ設計（`config.yaml`）

| モデル ID | 実体 | コンテキスト | グループ | 同時稼働方針 |
| :--- | :--- | :--- | :--- | :--- |
| `gemma4-review` | Gemma 4 26B-A4B Q4_0 + mmproj Q8_0 | 65536 (KV `q8_0`) | `llm` | `swap: true`, `exclusive: false`（`gemma4-fast` と排他） |
| `gemma4-fast` | Gemma 4 12B Q4_0 + mmproj Q8_0 | 32768 | `llm` | **`--reasoning-budget 0` で思考無効化**（短判定・OCR・構造化で高速化・空応答防止） |
| `gemma-embed` | EmbeddingGemma-300M QAT Q8_0 | 2048 (`--embedding --pooling mean`) | `embed` | `swap: false`, `exclusive: false`, `persistent: true`（LLM と並行稼働） |
| `whisper` | `whisper-server` (`ggml-large-v3-turbo.bin`) | — | `audio` | `swap: false`, `exclusive: false`, `persistent: true`（LLM/Embed と並行稼働） |

### 1.2 `llama-swap` 設定の必須要件
- **ポート:** 外部公開は `llama-swap` の `0.0.0.0:8080` のみ。バックエンド各サーバーは `127.0.0.1:${PORT}`（`startPort: 10001`）で待ち受け、`proxy: http://127.0.0.1:${PORT}` を明示（`localhost` の IPv6 `::1` 解決トラブル回避）。
- **アイドル自動アンロード:** `globalTTL: 300` および全モデルに `ttl: 300`（5分無通信でプロセス終了し RAM/VRAM を返還。`persistent: true` は他グループからの強制終了を防ぐだけで `ttl` は有効）。
- **コールドロード猶予:** `healthCheckTimeout: 300`。
- **チャット共通オプション:** `--n-gpu-layers 99 --no-webui --jinja --flash-attn on --temp 1.0 --top-k 64 --top-p 0.95 --parallel 1`。
- **`whisper-server` 必須オプション:**
  ```text
  --host 127.0.0.1 --port ${PORT}
  --model ${models}/ggml-large-v3-turbo.bin
  --request-path /v1/audio/transcriptions --inference-path ""
  --language {{PRIMARY_LANG}}
  --convert
  --tmp-dir /tmp
  ```
  - `checkEndpoint: /v1/audio/transcriptions/`（`whisper-server` には `/health` がないため）。
  - `--convert` には `ffmpeg` のインストールが必要（mp3/m4a → 16kHz wav 変換）。
  - **`--tmp-dir /tmp` 必須:** `launchd` や `systemd` から起動すると作業ディレクトリが `/` になり、一時ファイル作成で HTTP 500 になるのを防ぐ。
- **macOS 固有注意点:** 32GB 機で大型モデルがロード失敗する場合、`sudo sysctl iogpu.wired_limit_mb=24576` で GPU 有線メモリ上限を引き上げる。
- **常駐化:**
  - macOS: `LaunchAgents`（`com.{{PROJECT_NAME}}.llama-swap.plist`、`RunAtLoad=true`, `KeepAlive=true`、Homebrew の絶対パス `/opt/homebrew/bin` を `PATH` に含める）。
  - Linux: `systemd` ユニット（同等の引数・環境変数）。
- **疎通テスト (`smoke_test.sh`):**
  1. `GET /v1/models`
  2. `gemma4-fast` テキスト応答（`content` 非空確認）
  3. `gemma4-fast` 画像入力（base64 `image_url`）
  4. `gemma-embed` が 768 次元を返すこと＋`/running` に `gemma4-fast` と `gemma-embed` が同時表示されること（Parallel Group 確認）
  5. `gemma4-review` テキスト応答（`max_tokens: 2048` 以上で思考＋回答が収まること、`gemma4-fast` と入れ替わること）
  6. `--wait-unload` 指定時: 330 秒待機後に `/running` が空になること

---

## 2. 常駐サーバー（Docker Compose & systemd）

### 2.1 ディレクトリ配置と前提
| パス | 内容 |
| :--- | :--- |
| `{{PKM_ROOT}}`（例: `/srv/pkm`） | PKM バンドル実体（専用ストレージを使う場合は `/etc/fstab` に `UUID=...` + `noatime,nofail` で固定マウント。Desktop の `/media/` 自動マウントは使わない） |
| `{{REPO_DIR}}`（例: `/opt/{{PROJECT_NAME}}`） | 開発リポジトリ＋ホスト用 Python 仮想環境（`.venv`、`uv sync --locked --no-dev`） |
| `/etc/{{PROJECT_NAME}}/{{VAULT_NAME}}.env` | バンドルごとの環境変数ファイル（systemd と Docker Compose で共用） |

- Docker は **Docker 公式 apt リポジトリ**（`docker-ce` + `docker-compose-plugin`）を使用（snap 版は `/srv` マウント制約や自動更新再起動があるため非推奨）。
- 24時間稼働のため OS の自動サスペンドを無効化する（`systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target`）。

### 2.2 `docker-compose.yml` 構成（4サービス）

プロジェクト名: `name: {{PROJECT_NAME}}-${NIKI_VAULT:-pkm}`

| サービス | イメージ / ビルド | ポート | プロファイル | 役割 |
| :--- | :--- | :--- | :--- | :--- |
| `rss-triage-app` | `apps/rss_triage/Dockerfile` (`python:3.13-slim` + `git` + `uv`) | `${NIKI_APP_PORT:-8000}:8000` | 常時 | トリアージ・検索・再レビュー・ダッシュボード Web UI。`user: "${NIKI_UID}:${NIKI_GID}"`、`${PKM_ROOT}:/pkm` マウント、`HOME=/tmp`、`PKM_ENV_FILE=/nonexistent`。ビルド時に `sqlite-vec` ロード確認＋`/healthz` HEALTHCHECK。 |
| `hermes-agent` | `rss-triage-app` と同一イメージ | なし | 常時 | `command: ["python", "/pkm/00_System/scripts/hermes.py", "--watch"]`、`healthcheck: { disable: true }`。`SEARXNG_URL` を参照。 |
| `searxng` | `${SEARXNG_IMAGE:-searxng/searxng:<pinned-tag>}` | `${NIKI_SEARXNG_PORT:-8888}:8080` | `tools` | メタ検索。`deploy/server/searxng/settings.yml` を `:ro` マウント（`search.formats: [html, json]`, `server.limiter: false`, `server.method: GET`）。`SEARXNG_SECRET` 必須。 |
| `open-webui` | `${OPEN_WEBUI_IMAGE:-ghcr.io/open-webui/open-webui:<pinned-tag>}` | `${NIKI_WEBUI_PORT:-3000}:8080` | `tools` | 週次レビュー・長文対話 UI。`OPENAI_API_BASE_URL=${MAC_MINI_URL}/v1`, `OPENAI_API_KEY=local`, `ENABLE_OLLAMA_API=false`, `DEFAULT_MODELS=gemma4-review`, Web 検索エンジンに `http://searxng:8080/search?q=<query>` を接続、`ANONYMIZED_TELEMETRY=false`。 |

> **ポイント:** `COMPOSE_PROFILES=tools` はサーバー上の1バンドルでのみ有効にし、2つ目のバンドルでは空にして SearXNG / Open WebUI の重複起動を避ける。

### 2.3 systemd ユニット＆タイマー一覧（ログイン不要のシステムユニット）

すべて `RequiresMountsFor=/srv/%i`（または `{{PKM_ROOT}}`）を指定し、ディスクマウント前に走らないようにする。

| ユニット | スケジュール | 実行コマンド・設定 |
| :--- | :--- | :--- |
| `{{PROJECT_NAME}}-compose@.service` | ブート時（`multi-user.target`） | `docker compose -p {{PROJECT_NAME}}-%i --env-file /etc/{{PROJECT_NAME}}/%i.env up -d --build --remove-orphans`（`Requires=docker.service`） |
| `{{PROJECT_NAME}}-build-index.timer` | 30分ごと（`*:0/30`, `RandomizedDelaySec=60`, `Persistent=true`） | `build_index.py`（`Nice=10`, `IOSchedulingClass=idle`, `TimeoutStartSec=2h`） |
| `{{PROJECT_NAME}}-autocommit@.timer` | 毎日 `02:30` | `autocommit.py`（ゲート不合格はコミットせず終了コード 1、push 失敗は 3 で失敗検知可能にする） |
| `{{PROJECT_NAME}}-lint-wiki.timer` | 毎日 `03:30` | `lint_wiki.py --save-db`（結果を `pkm.sqlite` へ保存してダッシュボード表示） |
| `{{PROJECT_NAME}}-graphify@.timer` | 毎日 `04:00`（`RandomizedDelaySec=5min`） | `run_graphify.py`（`Nice=15`, `TimeoutStartSec=4h`） |

**夜間ジョブの順序意図:** `02:30` 自動コミット（日中の人間編集を HEAD に確定） → `03:30` Wiki 健全性点検 → `04:00` 確定 Silver/Gold のグラフ生成。

---

## 3. 環境変数リファレンス (`/etc/{{PROJECT_NAME}}/{{VAULT_NAME}}.env`)

| 変数名 | 必須 | 既定値 / 例 | 用途 |
| :--- | :---: | :--- | :--- |
| `PKM_ROOT` | 必須 | `/srv/pkm` | ホスト上の PKM バンドルパス |
| `MAC_MINI_URL` | 必須 | `http://gpu-node:8080` | 推論サーバー（`llama-swap`）のベース URL |
| `PKM_DB` | 任意 | `$PKM_ROOT/00_System/pkm.sqlite` | 検索・lint 用 SQLite パス |
| `PKM_LLM_TIMEOUT` | 任意 | `300` | LLM 応答タイムアウト秒 |
| `PKM_MODEL_REVIEW` / `FAST` / `EMBED` | 任意 | `gemma4-review` / `gemma4-fast` / `gemma-embed` | 使用モデル ID の上書き |
| `NIKI_VAULT` | 任意 | `pkm` | Compose プロジェクト名サフィックス |
| `NIKI_APP_PORT` | 任意 | `8000` | `rss-triage-app` 公開ポート |
| `NIKI_UID` / `NIKI_GID` | 推奨 | `1000` / `1000` | コンテナ実行 UID/GID（バンドル所有者と一致させる） |
| `NIKI_VAULT_NAME` | 任意 | `pkm` | Obsidian `obsidian://open` 用の Vault 名 |
| `NIKI_REVIEWER` | 推奨 | `human:{{USER_ID}}` | 再レビュー画面「承認する」のアクター名 |
| `NIKI_CRAWL_INTERVAL` | 任意 | `120` | ソース巡回間隔の既定値（分） |
| `NIKI_ARCHIVE_DAYS` | 任意 | `7` | 未読自動アーカイブ日数 |
| `HERMES_INTERVAL` / `HERMES_LIMIT` | 任意 | `300` / `5` | Hermes 監視間隔（秒）と1回の処理上限 |
| `HERMES_SEARXNG_URL` | 任意 | `http://searxng:8080` | Hermes が参照する SearXNG URL |
| `COMPOSE_PROFILES` | 任意 | `tools`（1バンドルのみ） | SearXNG + Open WebUI の起動制御 |
| `SEARXNG_SECRET` | `tools`時必須 | `openssl rand -hex 32` | SearXNG 署名シークレット |
| `NIKI_SEARXNG_PORT` / `NIKI_WEBUI_PORT` | 任意 | `8888` / `3000` | SearXNG / Open WebUI 公開ポート |
| `SEARXNG_IMAGE` / `OPEN_WEBUI_IMAGE` | 推奨 | 日付/バージョン固定タグ | 本番イメージのタグ固定 |
| `GRAPHIFY_CMD` | 任意 | `graphify {input} --output {output}` | 外部 Graphify CLI 連携コマンド（未設定なら `graph.json` のみ） |
| `HATENA_CONSUMER_KEY` / `SECRET` / `ACCESS_TOKEN` / `ACCESS_SECRET` | 任意 | — | はてなブックマーク OAuth 1.0a 認証情報（4つ揃うと有効） |
| `NIKI_XURL` | 任意 | `xurl`（マウント時は `/opt/xurl/xurl`） | X 投稿用公式 CLI パス（`docker-compose.override.yml` でバイナリと `~/.xurl` を `/tmp/.xurl` へマウント） |
| `SYNCTHING_URL` / `SYNCTHING_API_KEY` | 任意 | `http://<host-ip>:8384` | ダッシュボードの Syncthing 接続状態表示 |

---

## 4. クライアント側セットアップ（作業 PC / モバイル）

1. **Obsidian + Obsidian Web Clipper:**
   - Syncthing で同期された `~/pkm/` を Vault として開く。
   - 添付ファイルの保存先設定を「現在のフォルダ下のサブフォルダ `_assets`」に設定（相対パス `./_assets/...` と整合）。
   - Web Clipper の保存先を `10_Inbox/12_web-clipper/` に設定（`type` キーを付けないこと）。
2. **AI コーディングエージェント（Claude Code / Antigravity 等）:**
   - OKF 公式 Agent Skills（`open-knowledge-format/open-knowledge-format` の `okf-generate` / `okf-consume`）を導入。
   - バンドルルートでエージェントを起動すると `AGENTS.md` が読み込まれ、ローカルプロファイル優先で Gold 合成・コンテンツ執筆を行える。
3. **スマホ PWA:**
   - Tailscale 経由で `http://{{APP_HOST}}:8000/` を開き「ホーム画面に追加」。
   - ブラウザの共有メニューから URL を直接 `/share-target` へ送ってクリップ可能。
