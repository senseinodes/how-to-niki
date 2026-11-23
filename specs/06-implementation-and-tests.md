# 06. 段階的実装ロードマップ・テスト仕様・ハマりやすいポイント集

---

## 1. 推奨実装ロードマップ（Phase 1 〜 Phase 5）

依存関係が少なくオフラインで単体テストしやすい順に、5フェーズで実装・検証する。

### Phase 1: PKM バンドル骨格・OKF スキーマ・決定論的ガーディアン
- **対象ファイル:**
  - `pyproject.toml`, `.python-version`, `.github/workflows/test.yml`
  - `pkm/` テンプレート一式（ディレクトリ、`index.md`、`log.md`、`.gitignore`、`.stignore`、`AGENTS.md`、`00_System/specs/okf_local_profile.md`）
  - `pkm/00_System/schemas/okf_v03.py`
  - `pkm/00_System/scripts/{pkm_config,gate_okf,lint_wiki,verify_note,autocommit}.py`
- **完了条件（DoD）:**
  - 正常な Silver / Gold ノートが `gate_okf.py` を通過し、`--fix` で未知キーと並び順を壊さず `x-pkm-medal` が記入される。
  - 以下のエラーを正確に検出できる: Bronze への `type` 付与、カテゴリ番号範囲外、禁止文字ファイル名、旧キー残存、不正タグ/重複/予約語タグ、不正アクター、リンク切れ・バンドル外パス、Gold の直置き・未承認、エージェントによる `x-pkm-my-comment` 改変や `verified` 改ざん。
  - Gold の検証失効がエラーではなく `review_pending` になり、`verify_note.py` で承認すると `review_cleared` に復帰する。
  - `autocommit.py` がゲート不合格ファイルだけステージから外して残りをコミットできる。

### Phase 2: 推論サーバー基盤 & LLM クライアント
- **対象ファイル:**
  - `deploy/inference/`（`config.yaml`, `launchd plist` または `systemd unit`, `smoke_test.sh`, `README.md`）
  - `pkm/00_System/scripts/gemma_client.py`
- **完了条件（DoD）:**
  - 設定ファイル検証テスト（YAML 構文、`ttl: 300`、Parallel Group、モデル ID 整合性）が通る。
  - `gemma_client.py` の自己修復ループテスト（1回目ゲート不合格 → エラー JSON 差し戻し → 2回目合格で `--fix` & lineage 記録、全試行失敗で元ファイル復元＆`logs/rejected/` 保存）が通る。
  - 実機で `smoke_test.sh --wait-unload` が通り、テキスト・画像・768次元 Embedding・LLM+Embed 同時稼働・5分後アンロードを確認できる。

### Phase 3: 全層統合検索ビルダー・検索・タグ統計・ナレッジグラフ
- **対象ファイル:**
  - `pkm/00_System/scripts/{build_index,search,tag_stats,run_graphify}.py`
- **完了条件（DoD）:**
  - 擬似バイナリ（PDF/Office/画像）に対するサイドカー生成・SHA256 差分スキップ・元ファイル削除時のサイドカー掃除・Office `dcterms:modified` 優先が動作する。
  - `pkm.sqlite` への `FTS5 (trigram)` + `sqlite-vec (768d)` 差分登録、3文字未満の `LIKE` フォールバック、RRF＋トラストティア重み順（Gold > Silver = 再レビュー待ち Gold > Bronze）が動作する。
  - 推論機オフライン時でも `build_index.py` は FTS 登録まで完了し、`search.py` は FTS フォールバックで結果を返す。
  - `index.md`（ルートフロントマター保持）と `log.md`（日付降順）が変更時のみ決定論的に書き出される。
  - `run_graphify.py` が合格 Silver/Gold のみから `graph.json` と `manifest.json` を生成し、入力不変時はスキップする。

### Phase 4: トリアージ Web アプリ (`rss-triage-app`) ＆ 自律ワーカー (`hermes.py`)
- **対象ファイル:**
  - `pkm/00_System/scripts/hermes.py`
  - `apps/rss_triage/`（`Dockerfile`, `README.md`, `rss_triage/*.py`, `templates/*.html`, `static/*`）
- **完了条件（DoD）:**
  - `hermes.py` が 10分経過済み未処理 Bronze を選び、計画 → SearXNG 調査 → Silver 生成＋自己修復 → `hermes.sqlite` 記録を実行でき、既引用 Bronze や `x-pkm-my-comment` の保護ルールを守る。
  - `rss-triage-app` の全5画面・全エンドポイントが動作し、テスト用モック（`State` の `chat` / `embedder` / `http` / `clock` 差し替え）で以下が検証済み:
    - RSS/Atom 巡回・条件付き GET・本文抽出・Few-shot スコアリング・7日自動アーカイブ
    - Keep（`type` なし Bronze 書き出し）・Dismiss・Undo・一括 Dismiss・Restore
    - 非 RSS サイトの AI セレクタ推定・プレビュー・再推定
    - Podcast 遅延文字起こし（巡回時に音声 DL しないこと、`<podcast:transcript>` 優先、未搭載時の `whisper` 呼び出し＆一時音声削除）
    - 統合検索・関連ノート・再レビュー差分＆承認・ダッシュボード集計
    - Share 確認ガードと私的ソース/`28_me` の送信拒否
    - PWA マニフェスト・Service Worker・`/share-target` クリップ
  - `docker build` で `sqlite-vec` ロード確認が通る。

### Phase 5: 常駐サーバーデプロイ構成・運用仕上げ
- **対象ファイル:**
  - `docker-compose.yml`
  - `deploy/server/`（`pkm.env.example`, `searxng/settings.yml`, `systemd/*`, `README.md`）
  - ルート `README.md`
- **完了条件（DoD）:**
  - `docker-compose.yml` と systemd ユニット・`searxng/settings.yml` の整合性テストが通る。
  - `uv run pytest` が CI（GitHub Actions）含め全件パスする。

---

## 2. テスト設計方針と必須テストスイート

すべての単体・結合テストは **ネットワーク・実推論サーバー・実 GitHub に依存せずオフラインで高速完結** させる（依存注入で `chat`, `embedder`, `extractor`, `http`, `runner`, `clock` を差し替える設計にする）。

| ディレクトリ | テストファイル例 | 主な検証観点 |
| :--- | :--- | :--- |
| `tests/schemas/` | `test_okf_v03.py` | MUST/SHOULD/任意キー、旧キー拒否、タイムスタンプ正規化、順序非依存トラストティア、パス解決（ルート基準/ファイル基準/外部拒否/デコード）、保護フィールド比較 |
| `tests/scripts/` | `test_pkm_config.py`, `test_gate_okf.py`, `test_lint_wiki.py`, `test_verify_note.py`, `test_autocommit.py`, `test_gemma_client.py`, `test_build_index.py`, `test_tag_stats.py`, `test_hermes.py`, `test_run_graphify.py` | ルール1〜7網羅、生 YAML 編集の未知キー/順序保持、git 除外ファイルの lineage 非記録、`--save-db`、自己修復リトライ＆ロールバック、サイドカー差分＆Office日付、FTS5+vec+3文字未満LIKE、RRF重み付け、Hermes 接続断時の非記録スキップ、グラフの Bronze 除外 |
| `tests/apps/` | `test_rss_triage_core.py`, `test_rss_triage_web.py`, `test_rss_triage_pages.py`, `test_rss_triage_sites.py`, `test_rss_triage_podcast.py` | 巡回/採点/Keep/Undo/Archive、FastAPI TestClient による全5画面・HTMX 応答・OOB 削除、AI セレクタ、Podcast 遅延文字起こし＆一時ファイル消去、Share OAuth 署名・私的ガード、PWA share_target |
| `tests/deploy/` | `test_inference_config.py`, `test_server_compose.py` | `llama-swap` config の `ttl: 300`・グループ設定・全モデルID、`docker-compose.yml` のボリューム・環境変数・`PKM_ENV_FILE=/nonexistent`・SearXNG JSON フォーマット有効化 |

---

## 3. 実装・運用でハマりやすいポイント集（Gotchas）

オリジナル構築・実機運用で判明した落とし穴。実装時に必ず先回りして対策すること。

1. **`pkm.sqlite` と `rss_triage.sqlite` を絶対に同じファイルにしない:**
   `build_index.py --rebuild` は `pkm.sqlite`（および `-wal`/`-shm`）をファイルごと削除して再作成する。アプリ状態が同じ DB にあると登録ソースや Keep/Dismiss 履歴が全消失する。
2. **`sqlite-vec` の拡張ロード確認:**
   Python 標準 `sqlite3` で `conn.enable_load_extension(True); sqlite_vec.load(conn)` が動くかは OS/ビルド依存。Dockerfile のビルド時 `RUN` とサーバーセットアップ手順の両方で必ずスモークテストする。
3. **日本語全文検索と FTS5 trigram の3文字制限:**
   SQLite の `unicode61` は日本語を分かち書きできないため `tokenize='trigram'` を使う。ただし trigram は **1〜2文字の検索語（例: `AI`, `設計`）にヒットしない** ため、`len(term) < 3` の語だけ `LIKE` 条件へ振り分ける実装が必須。
4. **推論モデルの思考トークン（`reasoning_content`）による空応答:**
   Gemma 4 等の思考対応モデルは `max_tokens` が小さいと思考だけで枠を使い切り `choices[0].message.content` が空になる。
   - 高速・大量判定モデル（`gemma4-fast`）は `--reasoning-budget 0` で思考を切る。
   - 思考有効モデル（`gemma4-review`）は `max_tokens` を十分大きく（2048以上）するか未指定にする。
5. **`whisper-server` を常駐デーモン起動すると HTTP 500 になる問題:**
   `launchd` / `systemd` 起動時はカレントディレクトリが `/` になることがある。`--convert` 使用時は必ず `--tmp-dir /tmp` を明示する。
6. **`EmbeddingGemma` のプロンプト接頭辞:**
   素のテキストをそのまま埋め込むと検索精度が大きく落ちる。文書は `title: <title> | text: <text>`、クエリは `task: search result | query: <query>` を必ず付与する。
7. **YAML タイムスタンプの自動型変換:**
   PyYAML は引用符なしの ISO 日時を `datetime.datetime` に変換する。バリデータは `str` と `datetime` / `date` の両方を受け付け、LLM へのプロンプトでは引用符付き RFC 3339 文字列を指示して差分ノイズを減らす。
8. **コンテナ内の `pkm_config.py` と `.env` 探索:**
   Docker コンテナ内で `PKM_ENV_FILE` を未設定のままにすると、`pkm_config.py` がホスト向け `.env` を探しに行く可能性がある。Compose の `environment` で `PKM_ENV_FILE: /nonexistent` を設定して明示的に無効化する。
9. **コンテナ内の `git` 実行と `safe.directory`:**
   ホスト所有の `/srv/pkm` をコンテナへバインドマウントして `git diff` / `git status` を叩く際、UID 不一致等で dubious ownership エラーになりやすい。コンテナの `user:` をホスト所有者に合わせつつ、`git -c safe.directory=<root>` を付ける。
10. **トラストティア判定で `verified[-1]` を使わない:**
    人間承認後に `process:lint-wiki` が追記されると末尾が `process:` になり誤って失効扱いになる。必ず「`at >= generated.at` の全イベントの中に `human:` が含まれるか」で判定する。
11. **書式修正・リンク修復での `generated.at` 更新禁止:**
    意味の変わらない修正で `generated.at` を進めると、承認済み Gold が意図せず「再レビュー待ち」に落ちてしまう。
12. **私的カテゴリ（`28_me`）の多層防御:**
    `.gitignore` に書くだけでは不十分。`lineage.jsonl` / `log.md`（Git コミットされる）へのメタデータ漏洩を防ぐ `git check-ignore` 連携と、Web UI 側での Share ブロック（パス判定＋ソースの `private` フラグ）をセットで実装する。
13. **Syncthing での `.git` 同期禁止:**
    `.git` を複数端末で同期するとコンフリクトファイルが発生してリポジトリが壊れる。`.stignore` で `.git` と `*.sqlite*` を必ず除外し、Git 管理は常時稼働サーバー1台のみで行う。
