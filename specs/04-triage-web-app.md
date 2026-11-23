# 04. トリアージ＆統合検索 Web アプリ仕様 (`apps/rss_triage/`)

日々の情報収集（RSS / 非 RSS サイト / Podcast）、AI 一次トリアージ、PKM 全層統合検索、Gold 再レビュー承認、運用ダッシュボード、PWA スマホクリップを1つにまとめた軽量 Web アプリケーション仕様。

---

## 1. 技術制約・基本設計

- **バックエンド:** Python 3.13（`python:3.13-slim` + `git` + `tzdata`）+ FastAPI（`create_app` ファクトリパターン）+ Uvicorn + Jinja2 + httpx + feedparser + trafilatura + BeautifulSoup4。
- **フロントエンド:** HTMX（`static/vendor/htmx.min.js` にローカル同梱）+ Vanilla CSS（CSS Variables + Grid / Flexbox、ライト/ダーク両対応）+ 最小 Vanilla JS（`static/app.js`）。ビルドツール一切不要。
- **PKM スクリプトの再利用:** イメージ内に PKM スクリプトを複製せず、コンテナへマウントした `{{PKM_ROOT}}/00_System/scripts/` を `sys.path` に追加して動的 import する（`pkm.py`）。
- **データベース分離:**
  - `00_System/rss_triage.sqlite`: アプリ自身の読み書き用 DB（WAL モード、ロック保護）。
  - `00_System/pkm.sqlite`: `file:<path>?mode=ro` の**読み取り専用**で開く（存在しない・破損時でもアプリは落ちずに動く）。
  - `00_System/hermes.sqlite`: ダッシュボード集計時のみ読み取り専用で開く。
  - `index.md` / `log.md` / `lineage.jsonl` をアプリから直接書かない（承認時のみ `verify_note.py` を呼ぶ）。
- **タイムゾーン:** 画面の日時表示とダッシュボードの週区切りは `TZ` 環境変数（既定 `Asia/Tokyo`）から `zoneinfo.ZoneInfo` で決める（`settings.local_tz()`）。イメージに `tzdata` を入れ、見つからない場合も `Asia/Tokyo` は UTC+9 に、それ以外は UTC にフォールバックする。オフセットを直書きしない。
- **スケジューラ:** FastAPI の `lifespan` 内で15分刻み（`SCHEDULER_TICK = 900s`）のバックグラウンドループを起動し、ソースごとの `interval_min` を過ぎたものだけ巡回する（`NIKI_SCHEDULER=0` で無効化可）。また起動時に途中で止まった Podcast の `running` 状態を `queued` に戻し、文字起こしワーカーを起動する。

---

## 2. アプリ状態 DB スキーマ (`00_System/rss_triage.sqlite`)

```sql
CREATE TABLE IF NOT EXISTS app_meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);

CREATE TABLE IF NOT EXISTS sources (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    kind TEXT NOT NULL DEFAULT 'rss',        -- rss | site | podcast
    name TEXT NOT NULL,
    url TEXT NOT NULL UNIQUE,
    site_url TEXT,
    enabled INTEGER NOT NULL DEFAULT 1,
    private INTEGER NOT NULL DEFAULT 0,      -- 1 = 私的ソース（Share 禁止）
    interval_min INTEGER NOT NULL DEFAULT 120,
    selector TEXT,                           -- 非RSSサイト用 CSS セレクタ JSON
    etag TEXT,
    last_modified TEXT,
    last_fetched_at TEXT,
    last_status TEXT,                        -- ok | not-modified | selector-mismatch | robots | http-<code> | error
    last_error TEXT,
    fail_count INTEGER NOT NULL DEFAULT 0,
    last_new INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source_id INTEGER NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    guid TEXT NOT NULL,
    kind TEXT NOT NULL DEFAULT 'article',    -- article | trend | podcast
    url TEXT NOT NULL,
    title TEXT NOT NULL,
    author TEXT,
    published_at TEXT,
    fetched_at TEXT NOT NULL,
    feed_summary TEXT NOT NULL DEFAULT '',
    content TEXT,                            -- trafilatura 抽出本文（Markdown）
    image_urls TEXT NOT NULL DEFAULT '[]',   -- JSON 配列（最大10件）
    char_count INTEGER NOT NULL DEFAULT 0,
    score INTEGER,                           -- 関心スコア 0〜100（未採点 NULL）
    summary TEXT,                            -- 3行要約 JSON 配列
    score_error TEXT,                        -- 失敗回数プレフィックス 'x1:...' 等（最大3回まで試行）
    scored_at TEXT,
    status TEXT NOT NULL DEFAULT 'unread',   -- unread | kept | dismissed | archived
    comment TEXT NOT NULL DEFAULT '',        -- 一次感想（Keep 時に x-pkm-my-comment へ）
    kept_path TEXT,                          -- Keep で書き出した Bronze 相対パス
    acted_at TEXT,
    audio_url TEXT,                          -- Podcast エンクロージャ URL
    duration_sec INTEGER,
    transcript_url TEXT,                     -- <podcast:transcript> URL
    transcript_status TEXT,                  -- queued | running | done | failed
    transcript_error TEXT,
    transcript_method TEXT,                  -- published | whisper
    UNIQUE (source_id, guid)
);
CREATE INDEX IF NOT EXISTS items_status_score ON items(status, score DESC);
CREATE INDEX IF NOT EXISTS items_fetched ON items(fetched_at);

CREATE TABLE IF NOT EXISTS actions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    item_id INTEGER REFERENCES items(id) ON DELETE CASCADE,
    action TEXT NOT NULL,                    -- keep | dismiss | restore | archive | share | send
    prev_status TEXT,
    at TEXT NOT NULL,
    undone INTEGER NOT NULL DEFAULT 0,
    detail TEXT                              -- JSON
);
CREATE INDEX IF NOT EXISTS actions_at ON actions(at);

CREATE TABLE IF NOT EXISTS crawl_runs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    started_at TEXT NOT NULL,
    finished_at TEXT,
    sources INTEGER NOT NULL DEFAULT 0,
    new_items INTEGER NOT NULL DEFAULT 0,
    scored INTEGER NOT NULL DEFAULT 0,
    errors INTEGER NOT NULL DEFAULT 0,
    archived INTEGER NOT NULL DEFAULT 0
);
```

---

## 3. モジュール別機能仕様

### 3.1 フィード取得・OPML (`feeds.py`)
- `ETag` / `If-Modified-Since` による条件付き GET（`304 Not Modified` 時は処理スキップ）。
- エンクロージャに `audio/*` または `.mp3`/`.m4a` があれば Podcast として自動判定し、`audio_url`, `itunes_duration`, `<podcast:transcript>`（`podcast_transcript` / `rel="transcript"`）を記録。**音声ファイルは一切ダウンロードしない。**
- OPML 2.0 のインポート（入れ子 `<outline>` フラット化、`type="podcast"` 認識）とエクスポート（非 RSS サイトは除外）。

### 3.2 本文抽出・robots.txt (`extract.py`)
- オリジン単位で `robots.txt` をキャッシュする `RobotsCache`（404 等は許可、401/403 は全拒否、ユーザーエージェント明示）。
- `trafilatura.extract(..., output_format="markdown", include_images=True, include_links=False, favor_precision=True)` で本文抽出（上限 40,000 文字）＋メタデータ画像抽出。
- フィードの要約より短い抽出結果は採用しない。

### 3.3 非 RSS サイト＆AI セレクタ推定 (`sites.py`)
- Togetter / posfie 等のトレンド系ホスト（`TREND_HOSTS`）は `item.kind = "trend"` として分類。
- **HTML 簡約 (`simplify_html`):** `script/style/noscript/svg/iframe/head/link/meta/form/picture/source` とコメントを除去し、属性を `class/id/href/datetime`（各80文字まで）に絞って最大 14,000 文字に圧縮。
- **AI 推定 (`estimate_selector`):** 簡約 HTML を `gemma4-fast` に渡し、`{"item": "...", "title": "...", "link": "...", "date": "..."}` の CSS セレクタ JSON を推定（再推定時は壊れた旧セレクタをヒントとして渡す）。
- 登録前に必ず**試し取得プレビュー画面**で抽出結果を人間が確認・微調整してから保存する。
- 巡回時に 0 件一致または構文エラーになったら `last_status = "selector-mismatch"` にし、ソース一覧に「AI でセレクタを再推定」ボタンを表示する。

### 3.4 関心スコア学習＆3行要約 (`scoring.py`)
- `actions` テーブルから `undone = 0` の直近 Keep 6件（タイトル＋一次感想）と直近 Dismiss 6件（タイトル）を取得し、Few-shot 例として `gemma4-fast` に渡す。
- 出力 JSON: `{"score": 0〜100, "summary": ["1行目", "2行目", "3行目"], "reason": "1文"}`（各要約60文字以内、英語記事でも主言語で要約）。
- パース失敗時は最大1回 JSON 再出力を促し、連続パースエラーはアイテムごとに3回まで（`x3:...`）で打ち切り。推論サーバー到達不可時は巡回内の採点を中断して次回に回す。

### 3.5 巡回オーケストレーション (`crawler.py`)
1. `crawl_runs` に開始行を記録。
2. 期限到来（`interval_min` 経過）または `force=True` の有効ソースを巡回（初回取得は最新 30 件に制限して未読爆発を防ぐ）。
3. `article` / `trend` の新着は `fill_content()` で本文抽出。
4. 未採点・未読アイテムを最大 200 件まで `score_pending()` で採点。
5. `NIKI_ARCHIVE_DAYS`（既定 7 日）以上操作されていない未読を自動アーカイブ（`status='archived'`、削除しない）。
6. CLI 単発実行も提供: `python -m rss_triage.crawler --once [--force] [--no-score]`。

### 3.6 トリアージ＆Bronze 保存 (`triage.py`, `inbox.py`)
- **Keep:** 非 Podcast アイテムを `10_Inbox/11_rss/YYYY-MM-DD-<slug>.md` に保存（`open(..., "x")` で絶対上書きしない、衝突時は `-2`, `-3`）。
  - **私的ソース（`sources.private == 1`）の記事は `10_Inbox/19_private/` に保存する**（`inbox.inbox_dir(kind, private)` で決める。Git 除外・Hermes は `28_me` 固定・SearXNG 調査なし）。
  - フロントマター: `title`, `source`, `author?`, `published?`, `created`, `x-pkm-source-name`, `x-pkm-collector: rss-triage-app`, `x-pkm-score?`, `x-pkm-summary?`, `x-pkm-my-comment?`（**`type` キーは絶対に含めない**）。
- **Dismiss / 一括 Dismiss:** ステータスを `dismissed` に変更し、`actions` の ID をトーストの「元に戻す（Undo）」ボタンへ渡す。
- **Undo / Restore:** Dismiss / Archive を元に戻す。Restore 時は学習例から外すため `actions.undone = 1` にする。

### 3.7 Podcast 遅延文字起こしワーカー (`podcast.py`)
- `[Send to PKM]` 押下時に `transcript_status = 'queued'` に積む。排他ロック付きバックグラウンドスレッドが1件ずつ処理。
- 優先順位:
  1. `transcript_url` がある → 取得して形式別パース（Podcasting 2.0 JSON の `segments[]`、WebVTT / SRT からタイムスタンプ除去＆ `<v 話者>` 保持、HTML/プレーンテキスト）。`transcript_method = "published"`。
  2. `audio_url` がある → `tempfile.TemporaryDirectory` へストリーミング DL（上限 600MB）→ `POST ${MAC_MINI_URL}/v1/audio/transcriptions`（`model="whisper"`, `language="ja"`（`{{PRIMARY_LANG}}`）, `response_format="json"`, ヘッダー `Authorization: Bearer ${PKM_API_KEY}`, タイムアウト 3600s・接続 10s）→ **一時ディレクトリ削除で元音声を確実消去**。`transcript_method = "whisper"`。
- 結果を `10_Inbox/14_podcast/YYYY-MM-DD-<slug>.md`（私的ソースは `10_Inbox/19_private/`）へ保存（ショーノート＋文字起こし本文、`x-pkm-transcript` / `x-pkm-duration-sec` 付き、`type` なし）し、`status = 'kept'`, `transcript_status = 'done'` に更新。
- 失敗時は `transcript_status = 'failed'` と `transcript_error` を保存し、画面から再試行可能にする。

### 3.8 外部共有 (`share.py`)
- 送信先:
  - **はてなブックマーク:** `POST https://bookmark.hatenaapis.com/rest/1/my/bookmark`（OAuth 1.0a HMAC-SHA1 自前署名）。`HATENA_*` 4変数がすべて揃ったときのみ有効。
  - **X:** 公式 CLI `xurl -X POST /2/tweets -d '{"text": ...}'`（280文字上限チェック）。`NIKI_XURL` が実行可能なときのみ有効。
- **安全ガード:**
  - 確認ダイアログの `confirm="yes"` がなければ拒否。
  - `sources.private == 1` または `kept_path` が `PRIVATE_PREFIXES = ("10_Inbox/19_private/", "20_Literature/{{PRIVATE_CATEGORY}}/")` のいずれかで始まる場合は絶対に送信拒否（判定は `share.is_private_path` に一本化し、Share ダイアログ表示と送信の両方で使う）。私的ソースの Keep は `19_private/` に入るので、パス判定はソースのフラグと独立した二重の防御になる。

### 3.9 PWA スマホ共有クリップ (`clip.py`)
- `manifest.webmanifest` に `share_target`（`GET /share-target?title=&text=&url=`）と最小の Service Worker（`/sw.js`、静的ファイルのみキャッシュ、画面 HTML は常にネットワーク取得）を配置。
- Android の共有は URL を `text` に混ぜてくることがあるため `pick_url(url, text)` で URL を抽出。
- 確認フォームで一次感想を入力して保存すると、記事本文を `trafilatura` で抽出して `10_Inbox/12_web-clipper/YYYY-MM-DD-<slug>.md` に保存（`x-pkm-collector: rss-triage-app/share-target`、`type` なし）。

---

## 4. 全5画面 UI 仕様 & HTTP エンドポイント一覧

### 4.1 共通レイアウト・UX
- **左ナビ（768px 未満は下部タブバー）:** トリアージ（未読件数バッジ）/ 検索 / 再レビュー（警告色バッジ、`hx-trigger="load, every 5m"` で更新）/ ダッシュボード / ソース。
- **ヘッダー:** 最終巡回時刻・次回巡回時刻・推論サーバー状態（`llama-swap` `/running` を 60 秒キャッシュ：`busy` 応答中 / `idle` 待機中 / `down` 到達不可）・ライト/ダークテーマ切替（`localStorage` 永続化）。
- **キーボードショートカット（入力フォーカス時は無効）:**
  - `j` / `k`: 前後のカード選択、`Enter`: 本文を開く、`s`: Keep / Send to PKM、`d`: Dismiss、`u`: 直近トーストの Undo、`/` : 検索へフォーカスまたは `/search` 遷移、`?`: ヘルプダイアログ開閉。
- **スマホ対応（`< 768px`）:** 2ペイン（一覧｜本文）を1ペイン切替（一覧 ⇄ 本文＋戻るボタン）にする。

### 4.2 5画面の概要
1. **トリアージ (`/triage`):**
   - 左ペイン: 種別フィルタ（すべて / RSS・Web / トレンド / Podcast）、スコアフィルタ（全スコア / 50+ / 80+）、未読/Dismiss済み切替、「50未満をまとめて Dismiss」、「今すぐ巡回」、スコア降順カード一覧（一次感想入力欄、Keep / Send to PKM / Share / Dismiss、Podcast の文字起こし状態ポーリング）。
   - 右ペイン: 記事メタ情報、3行要約、本文、**関連ノート（最大5件）**（`/items/{id}/related` を HTMX 遅延読込し、既存 Silver/Gold との重複取り込みを防止）。
2. **統合検索 (`/search`):**
   - クエリ入力、モード切替（ハイブリッド / 全文 / 意味）、層チェックボックス（Gold / Silver / Bronze）、カテゴリフィルタ（実在フォルダから動的生成）。
   - 結果カード: 層・トラストティアバッジ、パス、検索語 `<mark>` ハイライト抜粋、一致方式（`fts`/`vec`）、更新日、生成者・検証者、「Obsidian で開く（`obsidian://open?vault=<vault>&file=<path>`）」。
3. **再レビュー待ち (`/review`):**
   - 対象 Gold 一覧（失効した検証、最終更新、待ち日数、「差分を見る」「Obsidian で開く」）。
   - 差分表示（`/review/diff`）: 最後の人間検証時刻直前の git コミットから作業ツリーまでの `git diff` を表示し、差分下に「承認する」ボタンを配置。
   - 承認押下（`POST /review/approve`）: `verify_note.py` を呼び、合格すれば OOB 削除＋トースト、不合格ならエラー詳細を表示してロールバック。
   - サイドバー: 手動解除コマンドと最近の `lineage.jsonl` 履歴（直近15件）。
4. **ダッシュボード (`/dashboard`):**
   - 層別件数（Bronze / Silver / Gold / 再レビュー待ち＋今週増分、サイドカー除外）。
   - 直近8週間の層別新規ノート棒グラフ（`TZ` のローカル時刻で月曜始まり）。
   - 今週の流れ（取得 → Keep / Keep率 / Dismiss → Silver 昇格 → Gold 合成、Podcast 文字起こし数、Hermes ゲート自己修復回数）。
   - タグ別ランキング上位10（期間切替 `week` / `30d` / `all` の HTMX 部分更新、前期比増減、タグクリックで `/search` 遷移）。
   - フォルダ別件数表（実在フォルダから動的生成）。
   - システム状態: 推論サーバー、Whisper キュー、Hermes 稼働・失敗数、最新 `lint_wiki` 結果、`pkm.sqlite` チャンク/未埋込数、失敗中ソース一覧、Git 未コミット数・最終コミット時刻、Syncthing デバイス接続状態（設定時）。
5. **ソース管理 (`/sources`):**
   - OPML インポート・エクスポート。
   - ソース追加フォーム（RSS / Podcast 直接追加、非 RSS サイトはプレビュー画面 `/sources/site/preview` 経由で AI セレクタ確認後に登録）。
   - ソース一覧（有効/無効トグル、私的フラグ、巡回間隔編集、削除、週次件数・平均スコア、失敗種別ごとのアクション表示：`selector-mismatch` →「AI でセレクタを再推定」、`http-404` →「URL を確認」）。

### 4.3 HTTP ルート一覧

| Method | Path | 機能 |
| :--- | :--- | :--- |
| `GET` | `/` | `/triage` へ 302 リダイレクト |
| `GET` | `/healthz` | `200 ok`（Docker HEALTHCHECK 用） |
| `GET` | `/triage` | トリアージ画面（HTMX `HX-Target=item-list` 時は一覧部分 HTML） |
| `GET` | `/items/{id}` | 本文ペイン HTML（非 HTMX は `/triage?item={id}` へ） |
| `GET` | `/items/{id}/related` | 関連 Silver/Gold ノート（最大5件）部分 HTML |
| `GET` | `/items/{id}/card` | 単一カード OOB 再描画（Podcast 状態更新用） |
| `POST` | `/items/{id}/comment` | 一次感想の途中保存 |
| `POST` | `/items/{id}/keep` | Bronze 保存＆カード OOB 削除 |
| `POST` | `/items/{id}/send` | Podcast 文字起こしキュー投入＆ワーカー起動 |
| `POST` | `/items/{id}/dismiss` | Dismiss＆Undo 付きトースト返却 |
| `POST` | `/items/bulk-dismiss` | 指定スコア未満（既定50未満）を一括 Dismiss |
| `POST` | `/undo` | `actions` ID リストの Undo |
| `POST` | `/items/{id}/restore` | Dismiss/Archived から未読へ復元 |
| `GET` | `/items/{id}/share` | Share 確認ダイアログ HTML |
| `POST` | `/items/{id}/share` | Share 実行（`confirm=yes` 必須・私的ガード） |
| `GET` | `/status` | ヘッダーステータス＋再レビューバッジ更新 HTML |
| `POST` | `/crawl` | バックグラウンド強制巡回開始 |
| `GET` | `/search` | 統合検索画面（HTMX `HX-Target=results` 時は結果部分 HTML） |
| `GET` | `/review` | 再レビュー待ち一覧＋lineage 履歴 |
| `GET` | `/review/diff?path=...` | `git diff`＋承認ボタン HTML |
| `POST` | `/review/approve` | `verify_note.py` 実行による承認 |
| `GET` | `/dashboard` | ダッシュボード画面（HTMX `HX-Target=tag-ranking` 時はタグ表のみ） |
| `GET` | `/sources` | ソース管理画面 |
| `POST` | `/sources` | RSS / Podcast ソース追加 |
| `POST` | `/sources/site/preview` | 非 RSS サイト AI セレクタ推定＆試し取得プレビュー |
| `POST` | `/sources/site` | 非 RSS サイト登録・セレクタ更新 |
| `POST` | `/sources/{id}/reselect` | 既存非 RSS サイトのセレクタ再推定プレビュー |
| `POST` | `/sources/{id}/toggle` | ソース有効/無効切替 |
| `POST` | `/sources/{id}/edit` | ソース名・巡回間隔・私的フラグ編集 |
| `POST` | `/sources/{id}/delete` | ソース削除 |
| `POST` | `/sources/import` | OPML アップロード |
| `GET` | `/sources/export.opml` | OPML ダウンロード |
| `GET/POST` | `/share-target` | PWA 共有メニューからの URL クリップ確認＆保存 |
| `GET` | `/manifest.webmanifest`, `/sw.js` | PWA マニフェスト・Service Worker |
