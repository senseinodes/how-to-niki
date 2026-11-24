# 01. システムアーキテクチャ・設計思想・制約事項

## 1. コンセプトと解決する課題

### 1.1 コアコンセプト
**「確率論的 LLM（意味理解・要約・構造化・対話）」** と **「決定論的コード制御（スキーマ検証・導出値計算・索引生成・監査ログ）」** を明確に分離し、外部の有料 LLM API・有料クラウド DB に依存せず長期運用できる個人用ナレッジパイプライン（LLM Wiki）を構築する。

### 1.2 解決する3つの課題
1. **情報過多と LLM API 課金の増大:**
   RSS・Web 記事・Podcast・PDF 等の大量インプットをローカル LLM で処理し、ランニングコストをゼロにする。
2. **LLM Wiki の「腐敗（Decay）」防止:**
   LLM にファイル配置・索引・履歴維持を任せると、フォーマット崩れ・リンク切れ・未検証情報の混入が蓄積する。以下を徹底して防ぐ。
   - LLM が書くべき値（意味内容）とコードが導出する値（階層・カテゴリ番号・トラストティア）を分離する。
   - LLM の出力はすべて Pydantic ゲート（`gate_okf.py`）に通し、不合格ならエラーを差し戻して自己修復させる。
   - 目録（`index.md`）・更新ログ（`log.md`）・監査ログ（`lineage.jsonl`）はコードのみが生成する（`lineage.jsonl` は `build_index.py` が30分ごとに状態変化を拾って記録するため、人間やエージェントがゲートを実行し忘れても漏れない）。
3. **会話自動記憶による記憶捏造リスクの排除:**
   会話ログから自動記憶を生成する仕組み（他人の記事内容と自分の体験が混ざるリスクや常駐グラフ DB の重さ）を避け、**ゲートを通過した確定 OKF Markdown ファイル群のみ** を Single Source of Truth とする。

---

## 2. 厳守すべき技術制約（Do's & Don'ts）

AI エージェントが実装時に独自判断で逸脱してはならない制約事項。

### 禁止事項（Don'ts）
1. **CSS フレームワーク・JS ビルドツールの導入禁止:**
   Web UI は **FastAPI + Jinja2 + HTMX + Vanilla CSS（CSS Variables / Grid / Flexbox）+ 最小限の Vanilla JS** で構築する。Node.js、Tailwind CSS、React/Vue、Vite、Webpack 等のフロントエンドビルド環境は導入しない。HTMX はベンダーファイル（`static/vendor/htmx.min.js`）としてローカル同梱し CDN に依存しない。
2. **重厚な外部 DB・別系統メモリシステムの導入禁止:**
   PostgreSQL、MySQL、Neo4j、Qdrant、Chroma、Elasticsearch 等の外部 DB コンテナは立てない。永続化と検索はすべて **SQLite 3（`FTS5` + `sqlite-vec`）** に集約する。
3. **4つの SQLite ファイルを混ぜないこと:**
   用途とライフサイクルが異なるため、必ず同一フォルダ（`00_System/`）内の別ファイルに分ける。
   - `pkm.sqlite`: 検索インデックス・タグ統計・lint 結果（`build_index.py --rebuild` でファイルごと再作成可能）
   - `rss_triage.sqlite`: Web アプリの状態（ソース・記事・操作履歴・巡回ログ。`pkm.sqlite` を再作成しても消えない）
   - `hermes.sqlite`: 自律 Silver 生成ワーカーの処理履歴・リトライ状態・自己修復回数
   - `links.sqlite`: Silver 同士のつながり候補・人間の判断（つなぐ / 却下 / 保留）・LLM の関係判定（`related_links.py`。消えても候補が再び出るだけで、リンク自体はノートの本文に残る）
4. **Bronze 層への OKF フロントマター混入禁止:**
   `10_Inbox/` 配下のファイルに `type` キーを付与してはならない。
5. **導出値を LLM に書かせない・Silver を移動しない:**
   - `x-pkm-medal` はゲートのみが書く。`jd_id`・`okf_version` はノートに書かない（`okf_version` はルート `index.md` のみ）。
   - Silver ノートを `50_Knowledge/` へ移動してはならない。Gold は Silver を `sources` に引用する**新しいノート**として合成する。
6. **ファイル名・フォルダ名への空白・`&` 等の使用禁止:**
   CommonMark のリンク・シェル・Docker マウント破損を防ぐため、禁止文字（`\s & # % ? < > " | *`）を使わない。
7. **推論サーバーの常時メモリ占有禁止（GPU/開発機共用の場合）:**
   推論サーバーはアイドル自動アンロード（`ttl: 300`）を設定し、アイドル時にメモリを解放する。

### 必須事項（Do's）
1. **確率論と決定論の分離と自己修復ループ:**
   LLM が生成・更新した Silver / Gold は必ず `gate_okf.py` を通し、エラー時は JSON 形式のエラー詳細をそのまま LLM へ返して再生成させる（既定最大3回）。
2. **生 YAML 行編集による未知キー・並び順の保持:**
   OKF v0.3 の未知キー保持義務に従い、スクリプトがフロントマターを書き換える場合（`gate_okf.py --fix` の `x-pkm-medal`、`verify_note.py` の `verified` 追記）は、Pydantic モデルをダンプし直さず**生の YAML 行を編集**して他のキーと並び順を1文字も変えない。
3. **人間の記述の保護:**
   `gate_okf.py` は `git show HEAD:<path>` と比較し、`generated.by` が `human:` でない変更による `x-pkm-my-comment` の改変・削除、および `verified` 既存イベントの変更・削除・並べ替えを拒否する。
4. **Podcast の遅延文字起こし（Lazy Transcription）:**
   RSS 巡回時は音声ファイルを絶対にダウンロードしない。一次トリアージは `<title>` と `<description>`（ショーノート）のみで行い、人間が `[Send to PKM]` を押したエピソードだけ `<podcast:transcript>` 取得または `whisper.cpp` 文字起こしを実行する。
5. **マルチフォーマットのサイドカー抽出:**
   `10_Inbox/` の PDF / Office / 画像は `MarkItDown`（ローカル VLM 連携）で `<filename>.extracted.md` を生成し、元バイナリを汚さずに SHA256 で差分管理する。
6. **私的区分の隔離（Bronze `10_Inbox/19_private/` と Silver `20_Literature/28_me/`）:**
   - Web アプリで「私的」フラグを付けたソースの Keep 記事・Podcast 文字起こしは、通常の `11_rss/` / `14_podcast/` ではなく `10_Inbox/19_private/` に保存する。
   - 両フォルダを `.gitignore` で除外し（GitHub に上げない。Syncthing では同期する）、`gate_okf.py` / `build_index.py` は `git check-ignore` を見て `lineage.jsonl` / `log.md` へ記録しない。
   - `hermes.py` は `19_private/` 由来の Bronze を LLM の配置計画にかけず `28_me` に固定し、SearXNG（外部検索エンジン）へ検索語を送らない。
   - Web アプリは `sources.private == 1` と、パス接頭辞 `10_Inbox/19_private/` / `20_Literature/28_me/` の両方で Share を拒否する。
   - `related_links.py` は私的ノートと公開ノートを組にしない（リンクを書くと公開ノートに私的ノートのパスが残るため）。
7. **リンクは人間が選び、コードが書く:**
   Silver 同士の相互リンク（本文末尾の `## 関連`）は、夜間に作った候補から人間が選び、`related_links.py` が追記して `gate_okf.py` に通す（不合格なら元に戻す）。LLM は候補の関係（同意 / 補足 / 対立の可能性）を判定するだけで、リンクを書かない。対立は解消せず「— 対立する見方」として残し、Gold で論点として扱う。

---

## 3. システム全体アーキテクチャ

```text
┌─────────────────────────────────────────────────────────────────────────┐
│ クライアント端末（PC / スマホ / Obsidian / AI コーディングエージェント） │
│  • Claude Code / Antigravity 等（Gold 合成議論・成果物執筆・OKF Skills） │
│  • Obsidian + Web Clipper（Markdown 閲覧・手動編集・クリップ入力）       │
│  • ブラウザ / PWA（rss-triage-app & Open WebUI、スマホ共有メニュー連携） │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ LAN / Tailscale VPN + Syncthing
┌───────────────────────────────▼─────────────────────────────────────────┐
│ 常時稼働サーバー（Linux 24/7 / Docker Compose + systemd timer）          │
│  • Docker Compose:                                                       │
│      - rss-triage-app : FastAPI + HTMX + Vanilla CSS + rss_triage.sqlite │
│      - hermes-agent   : Bronze監視 → SearXNG調査 → Silver生成 + 自己修復 │
│      - searxng        : メタ検索エンジン（JSON API 有効）                 │
│      - open-webui     : 週次レビュー・長文対話 UI                         │
│  • systemd timer（ホスト側 /opt/<project>/.venv）:                       │
│      - build_index.py (30分毎) : サイドカー・pkm.sqlite・lineage・index/log│
│      - autocommit.py  (02:30)  : ゲート検証付き自動 git commit & push    │
│      - lint_wiki.py   (03:30)  : リンク切れ・孤立・SHOULD点検 → DB保存   │
│      - run_graphify.py(04:00)  : Silver/Gold 限定ナレッジグラフ生成      │
│        → related_links.py build : Silver のつながり候補＋関係判定        │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │ HTTP :8080 (OpenAI 互換 API / On-Demand)
┌───────────────────────────────▼─────────────────────────────────────────┐
│ 推論サーバー（GPU / Apple Silicon / オンデマンド稼働）                    │
│  • llama-swap (:8080, ttl: 300, Parallel Group):                         │
│      - llm グループ (swap):   gemma4-review (大) / gemma4-fast (中+VLM)  │
│      - embed グループ (並行): gemma-embed (768次元 Embedding)             │
│      - audio グループ (並行): whisper (whisper.cpp /v1/audio/transcriptions)│
└─────────────────────────────────────────────────────────────────────────┘
```

### 重要な結合ポイント（スクリプトの単一配置）
- 決定論的スクリプト群（`00_System/scripts/`）とスキーマ（`00_System/schemas/`）は **PKM バンドル内（`{{PKM_ROOT}}/00_System/`）に配置** する。
- ホストの systemd timer も、Docker コンテナ（`rss-triage-app` / `hermes-agent`）も、バンドルをマウント（`/pkm`）して **同一のスクリプトコード** を実行する。これにより、コンテナとホストで仕様のズレが発生しない。
- ただしスクリプトの「正本」は開発リポジトリの `pkm/00_System/`（テンプレート、テスト対象）で、実行されるのはバンドル側のコピーである。開発リポジトリで更新したら `{{PKM_ROOT}}/00_System/` へコピーし、バンドル側でもコミットする（手順は 05 §2.1）。

---

## 4. ハードウェア構成パターン（環境に合わせた選択）

オリジナル環境は「常時稼働省電力 Linux（Intel N100 32GB）＋ 推論ノード（M4 Mac mini 32GB）」の2台構成だが、利用者の手元ハードウェアに応じて以下から選択できる。

| パターン | 構成 | 変更点・注意点 |
| :--- | :--- | :--- |
| **A: 2台構成（推奨・オリジナル準拠）** | 常時稼働 Linux（省電力PC / ミニPC / ラズパイ等）＋ GPU 搭載機（Apple Silicon Mac または NVIDIA GPU Linux/Windows） | 常時稼働の電気代を抑えつつ、推論時だけ GPU ノードへ HTTP 接続。GPU 機がスリープ中でも `build_index.py`（FTS のみ）や Web UI 閲覧は止まらない設計。 |
| **B: 1台完結構成** | GPU / 十分な RAM（32GB+ 推奨）を備えた常時稼働 Linux または Mac 1台 | `llama-swap` と Docker 群を同一マシンで動かす。`MAC_MINI_URL` は**ホストの LAN / Tailscale IP**（例: `http://192.168.1.10:8080`）にする。env ファイルは systemd（ホスト）と Compose（コンテナ）で共用するため、`localhost`（コンテナ内では自分自身）や `host.docker.internal`（Linux のホストからは引けず、コンテナでも `extra_hosts: host-gateway` が必要）は使わない。 |
| **C: クラウド API 併用構成（参考・非サポート）** | 常時稼働 Linux（または VPS）＋ OpenAI 互換クラウド API | 「API コストゼロ」の前提から外れるため仕様の対象外。試す場合の変更点: `PKM_API_KEY`（`gemma_client` / MarkItDown / whisper / Open WebUI が共通で使う）と `PKM_MODEL_*` を設定、EmbeddingGemma 専用接頭辞（03 §3.1）と 768 次元（`pkm_config.EMBED_DIM`）を埋め込みモデルに合わせて変更、llama-swap 固有の `/running`（ヘッダーの推論状態表示）と whisper の `/v1/audio/transcriptions` の代替を用意。 |

### モデル容量の目安（メモリ別）
推論ノードの VRAM / 統合メモリに応じて `llama-swap` のモデルを調整できる（アプリ側のモデル ID は環境変数で抽象化済み）。

| 統合メモリ / VRAM | `gemma4-review`（深い対話） | `gemma4-fast`（高速・VLM） | `gemma-embed` | `whisper` |
| :--- | :--- | :--- | :--- | :--- |
| **32GB 以上（標準）** | Gemma 4 26B-A4B Q4_0 + mmproj (64K ctx) | Gemma 4 12B Q4_0 + mmproj (32K ctx) | EmbeddingGemma-300M Q8_0 (768d) | large-v3-turbo |
| **16GB** | Gemma 4 12B Q4_0 + mmproj (32K ctx) | Gemma 4 4B〜12B Q4_0 + mmproj (16K ctx) | EmbeddingGemma-300M Q8_0 (768d) | medium / large-v3-turbo |

---

## 5. ネットワーク・同期・バックアップ戦略

### 5.1 ネットワークとポート
外部インターネットには公開せず、自宅 LAN または **Tailscale** 等のプライベート VPN 内でのみアクセスする。

| 送信元 | 送信先 | プロトコル / ポート | 用途 |
| :--- | :--- | :--- | :--- |
| クライアント / スマホ | 常時稼働サーバー | `HTTP :8000`（バンドル毎に増設可） | `rss-triage-app` Web UI / PWA |
| クライアント | 常時稼働サーバー | `HTTP :3000` | `Open WebUI`（週次レビュー・対話） |
| サーバー（App / Hermes / Scripts） | 推論サーバー | `HTTP :8080` | `llama-swap`（Chat / Embed / Whisper） |
| サーバー（Hermes / Open WebUI） | Docker 内部 `http://searxng:8080`（ホストからは `:8888`） | `HTTP :8080`（コンテナ間）/ `:8888`（公開） | `SearXNG` メタ検索 JSON API |
| サーバー・クライアント・NAS | メッシュ P2P | `TCP/UDP :22000` | `Syncthing` ファイル双方向同期 |

### 5.2 二重同期（Syncthing × Git）の役割分担
**「ファイル同期（Syncthing）」と「バージョン管理（Git）」の責務を明確に分ける**ことでコンフリクトを防ぐ。

- **Syncthing（リアルタイム P2P 同期）:**
  - 対象: 常時稼働サーバー ⇄ 作業 PC / モバイル ⇄ バックアップ NAS（推論専用機には同期不要）。
  - 除外（`.stignore`）: `.git`, `*.sqlite*`, `00_System/graph`, `.obsidian/workspace*.json`, `__pycache__`, `.claude/worktrees`。
  - **重要:** `.git` は Syncthing で同期しない。複数端末から同時にコミットすると履歴が分岐・破損するため、**Git 管理と自動コミットは常時稼働サーバー1台に集約**する。
- **Git / GitHub Private リポジトリ（履歴保全・ゲート基準）:**
  - 対象: OKF Markdown、スクリプト、設定、`lineage.jsonl`、`index.md`、`log.md`。
  - 除外（`.gitignore`）: `90_Assets/*`、私的区分（`20_Literature/28_me/`, `10_Inbox/19_private/`）、`00_System/graph/*`、`*.extracted.md`、`10_Inbox/15_files/` のバイナリ原本、`*.sqlite*`、`00_System/logs/rejected/`、`lost+found/`、`.stfolder/`。
  - **重要:** `gate_okf.py` のルール7（人間記述の保護）は `git show HEAD:<path>` を基準にするため、毎晩 `autocommit.py` で合格ファイルだけを自動コミット＆push する。

### 5.3 マルチバンドル拡張性
仕事用・個人用などで PKM バンドルを分けたい場合、同じリポジトリコードのまま以下を分けるだけで複数起動できる設計とする。
- データ配置: `/srv/<vault_name>`（systemd ユニットはすべて `@` テンプレートで `/srv/%i` を参照するため、この配置は固定）
- 環境変数: `/etc/<project>/<vault_name>.env`（`PKM_ROOT`, `NIKI_VAULT`, `NIKI_APP_PORT` を変更）
- Compose プロジェクト: `docker compose -p <project>-<vault_name>`
- systemd: `<project>-build-index@<vault_name>.timer` など5ユニットをバンドルごとに有効化
- `COMPOSE_PROFILES=tools`（SearXNG と Open WebUI）はサーバー全体で1つのバンドルだけ有効にする。
