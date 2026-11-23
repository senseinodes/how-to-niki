# 02. PKM バンドル構造・OKF v0.3 Local Profile・エージェント規範

この文書は、PKM バンドル（`{{PKM_ROOT}}` = テンプレートの `pkm/`）のディレクトリ設計、**PKM-OKF v0.3 Local Profile v1.1** の厳密なスキーマ定義、および全 AI エージェントが守る `AGENTS.md` の仕様を定義する。

---

## 1. メダリオン構造 × Johnny.Decimal × OKF v0.3

| LLM Wiki の層 | メダリオン | フォルダ | 形式 | Zettelkasten 対応 | 編集主体 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Raw sources | **Bronze** | `10_Inbox/` | 非 OKF（`type` なし） | Fleeting notes | 人間 / Web アプリ / Web Clipper / 抽出スクリプト |
| Wiki | **Silver** | `20_Literature/` | OKF v0.3 | Literature notes（ソース毎の構造化） | 自律ワーカー（Hermes） / AI エージェント / 人間 |
| Wiki | **Gold** | `50_Knowledge/` | OKF v0.3（`human:` 検証必須） | Permanent notes（複数ソースの合成） | 人間 + AI エージェント対話（承認は人間のみ） |
| Schema | — | `00_System/` | 仕様・スキーマ・スクリプト・DB・ログ | — | システム（通常運用時はエージェント読取専用） |
| Output | **Output** | `60_Output/` | 非 OKF（Gold からの派生物） | 成果物（記事・台本等） | 人間 + AI エージェント |
| Assets | — | `90_Assets/` | Git 除外（大容量メディア） | — | 人間（Syncthing のみ同期） |

---

## 2. バンドルディレクトリ構造

```text
pkm/
├── AGENTS.md                        # エージェント共通行動規範（非OKF）
├── index.md                         # [自動生成] バンドルルート目録（フロントマターに okf_version: "0.3" を保持）
├── log.md                           # [自動生成] lineage.jsonl から生成する OKF 標準更新ログ
├── .gitignore                       # Git 除外設定
├── .stignore                        # Syncthing 除外設定
├── 00_System/
│   ├── specs/
│   │   └── okf_local_profile.md     # 本仕様書のバンドル内コピー
│   ├── schemas/
│   │   └── okf_v03.py               # Pydantic v2 バリデーションモデル
│   ├── scripts/                     # 決定論的スクリプト群（03-guardians-and-pipeline.md）
│   ├── graph/                       # [Git/Syncthing除外] ナレッジグラフ出力（.gitkeep のみ管理）
│   ├── logs/
│   │   ├── lineage.jsonl            # 昇格・再レビュー状態変化の監査ログ
│   │   └── rejected/                # [Git除外] 自己修復失敗時の下書き保存先
│   ├── pkm.sqlite                   # [Git/Syncthing除外] build_index / lint_wiki 用 DB
│   ├── rss_triage.sqlite            # [Git/Syncthing除外] rss-triage-app 状態 DB
│   └── hermes.sqlite                # [Git/Syncthing除外] hermes.py 処理状態 DB
├── 10_Inbox/                        # 【Bronze】非OKF
│   ├── index.md
│   ├── 11_rss/                      # RSS・非RSSサイトから Keep した記事
│   ├── 12_web-clipper/              # Web Clipper・PWA 共有メニューから取り込んだ記事
│   ├── 13_quick-notes/              # 人間の手書きメモ・思考の断片
│   ├── 14_podcast/                  # Send to PKM した Podcast 文字起こし
│   └── 15_files/                    # PDF・Office・画像 ＋ 自動生成された <name>.extracted.md
├── 20_Literature/                   # 【Silver】OKF v0.3（カテゴリ番号 21〜29、20は予約）
│   ├── index.md
│   ├── 21_technology/               # 標準カテゴリ例（利用者が自由にカスタマイズ可）
│   ├── 22_politics-economy/
│   ├── 23_society/
│   ├── 24_life/
│   ├── 25_learning/
│   ├── 26_culture/
│   ├── (27_ 予備)
│   ├── 28_me/                       # 私的カテゴリ（Git/lineage/Share から除外）
│   └── 29_misc/                     # フォールバックカテゴリ
├── 50_Knowledge/                    # 【Gold】OKF v0.3 + human:検証（カテゴリ番号 51〜59、50は予約）
│   ├── index.md
│   ├── 51_concepts/                 # 原則・概念・メンタルモデル
│   └── 52_entities/                 # 人物・ツール・組織の確定プロフィール
├── 60_Output/                       # 【Output】非OKF
│   ├── index.md
│   ├── 61_youtube/                  # 成果物フォルダ例（利用者が自由にカスタマイズ可）
│   └── 62_note/
└── 90_Assets/                       # [Git除外] 大容量メディア・音声・コーパス（.gitkeep のみ管理）
```

### 2.1 命名規則と予約名
- **エリアフォルダ:** `NN_Name`（空白・`&` 禁止）。`30_` / `40_` は将来拡張用に予約。
- **カテゴリフォルダ:** 正規表現 `^(\d{2})_[a-z0-9]+(?:-[a-z0-9]+)*$` に一致。
  - `20_Literature`: `21`〜`29`（`20` は予約）。カテゴリ直下にサブフォルダは作らない（細分類は `tags` で行う）。
  - `50_Knowledge`: `51`〜`59`（`50` は予約）。
- **ノートファイル名:**
  - 禁止文字: `[\s&#%?<>\"|*]`（エラー）。
  - 推奨形式: 英小文字ケバブケースのスラッグ `^[a-z0-9]+(?:-[a-z0-9]+)*\.md$`（不一致は警告）。ノート単位には Johnny.Decimal 番号を振らない（1カテゴリ99件制限を回避）。
- **予約ファイル名:** `index.md` / `log.md` は全階層で予約（ノートとして使用不可）。
- **ノート添付画像:** ノートと同じカテゴリフォルダ配下の `_assets/` に配置し、`./_assets/diagram.png` のように相対パス参照する（Git 管理対象。`90_Assets/` には入れない）。
- **MoC（Map of Content）禁止:** 各階層の自動生成 `index.md` が目録を担う。キュレーション概観が必要なら通常の `concept` ノートとして作成する。

### 2.2 バンドルルート `index.md`（OKF v0.3 §8.1）
`okf_version` はルート `index.md` のフロントマターにのみ記述する。

```yaml
---
okf_version: "0.3"
title: "{{USER_ID}} PKM"
description: "RSS・Podcast・Webクリップから育てる個人用 LLM Wiki"
---
```

`build_index.py` は `<!-- build_index.py: この行より下は自動生成。手で編集しない -->` マーカーより下のみを再生成し、フロントマターとマーカーより上の手書き見出し・説明を保持する。サブディレクトリの `index.md` はフロントマターを持たない。

---

## 3. フロントマター・スキーマ定義（Silver / Gold 共通）

バリデータ実装: `00_System/schemas/okf_v03.py`（Pydantic v2、`ConfigDict(extra="allow")` により未知キーを保持）。

### 3.1 MUST キー（欠落・不正はゲートでエラー）

| キー | 型・制約 | 説明 |
| :--- | :--- | :--- |
| `type` | `"knowledge"` \| `"concept"` \| `"entity"` | OKF 構造化ファイルの識別子（`moc` は廃止）。 |
| `title` | `str`（1〜150文字） | ノート見出し・目録・検索タイトル。 |
| `description` | `str`（1〜400文字） | 1〜3文の要約。目録の説明・検索スニペットに使用。 |
| `generated` | `{ by: str, at: Timestamp }` | 内容の最終的な意味のある変更来歴。<br>• `by`: `<producer>/<version>`（`^[a-z0-9][a-z0-9._-]*/[a-z0-9][a-z0-9._-]*$`）または `human:<id>`（`^human:[a-z0-9][a-z0-9._-]*$`）<br>• `at`: オフセット必須の RFC 3339 文字列（例: `"2026-11-20T09:15:00+09:00"`）または日付のみ（UTC 00:00 扱い）。YAML ネイティブ `datetime` も許容するが引用符付き文字列を推奨。<br>• 書式修正・空白・リンク修復・`x-pkm-medal` 記入だけでは更新しない。 |

### 3.2 SHOULD キー（欠落は `lint_wiki.py` の警告のみ）

| キー | 型・制約 | 説明 |
| :--- | :--- | :--- |
| `tags` | `list[str]`（0〜15個） | 各要素 `^[a-z0-9]+(?:-[a-z0-9]+)*$`（小文字ケバブケース）。重複エラー。予約語 `bronze`, `silver`, `gold` は**エラー**。 |
| `sources` | `list[SourceItem]` | 一次情報源配列。各要素:<br>• `resource`（必須, min_length=1）: 絶対 URL またはバンドル内パス<br>• `id`, `title`, `author`, `last_modified`（任意・OKF 標準）<br>• `x-pkm-accessed-at`（任意・`Timestamp`） |
| `x-pkm-my-comment` | `Optional[str]` | 人間自身の一次感想・考察。**人間のみ記述・編集可**。エージェントによる変更・削除はゲートエラー。（任意なので `lint_wiki.py` の欠落警告対象からは除外） |

### 3.3 Gold 配置時（`50_Knowledge/`）の追加必須条件

| キー | 型・制約 | 説明 |
| :--- | :--- | :--- |
| `verified` | `list[VerificationEvent]`（1件なら単一 dict も可） | 各要素 `{ by: str, at: Timestamp }`。`by` は `<producer>/<version>` または `<scheme>:<id>`（`human:`, `process:`, `team:` 等）。**追記のみ許可**。 |
| `sources` | 上記 | **バンドル内の既存 Silver または Gold ノート（自身を除く）を1件以上含むこと**（Gold 直置き禁止を決定論的に保証）。 |

### 3.4 OKF 標準の任意キー・ローカル拡張キー・禁止キー

- **OKF 標準任意キー:**
  - `status`: `"draft"` \| `"stable"` \| `"deprecated"`（省略時 `"stable"`）
  - `superseded_by`: `Optional[str]`（`status == "deprecated"` のときのみ許可。バンドル内の実在パスであること）
  - `aliases`: `list[str]`（Obsidian 標準キー・名寄せ用）
- **ローカル拡張キー（`x-pkm-*`）:**
  - `x-pkm-medal`: `"silver"` \| `"gold"`。**ゲート（`gate_okf.py --fix`）のみが配置から記入**。LLM は書かない。配置と不一致ならエラー。
  - `x-pkm-lang`: `"ja"` \| `"en"`（省略時 `"ja"`）。
  - `x-pkm-my-comment`: 人間のみ。
  - `sources[].x-pkm-accessed-at`: 出典アクセス日時。
- **ノートに書いてはいけない禁止（旧仕様）キー（ゲートでエラー＋修正手順を返却）:**
  - ノート直下: `okf_version`, `jd_id`, `medal`, `lang`, `my_comment`
  - `sources[]` 内: `url`, `path`, `accessed_at`

---

## 4. 導出値・トラストティア・Gold 状態遷移

### 4.1 パスからの導出値
- `medal`: `20_Literature` → `"silver"`, `50_Knowledge` → `"gold"`
- `jd_id`: 親カテゴリフォルダ先頭の2桁数字（例: `21_technology` → `"21"`）

### 4.2 トラストティア判定（OKF v0.3 §5.3・順序非依存）
1. **現行の検証（Current Verifications）:** `verified` のうち `event.at >= generated.at` を満たすイベント群。
2. 判定ルール（**配列の末尾要素 `verified[-1]` は絶対に見ない**）:
   - 現行の検証に `human:` が1件以上ある → **`human-reviewed`**
   - 現行の検証に `human:` はないが、他のアクター（`process:`, `team:`, `<producer>/<version>`）がある → **`machine-confirmed`**
   - 現行の検証が0件 → **`unverified`**

### 4.3 Gold ノートの状態遷移

| 状態 | 条件 | ゲートの扱い |
| :--- | :--- | :--- |
| **承認済み** | `50_Knowledge/` かつ `trust_tier == "human-reviewed"` | 合格（`review_pending = False`） |
| **再レビュー待ち** | `50_Knowledge/` かつ `trust_tier != "human-reviewed"` かつ過去に `human:` 検証が1件以上ある | **合格（エラーにしない・ファイル無変更）**。`review_pending = True` とし、`lineage.jsonl` へ `review_pending` を記録、`index.md` と Web UI バッジに表示 |
| **未承認配置** | `50_Knowledge/` かつ `human:` 検証履歴が0件 | **エラー**（昇格には人間承認が必須） |

再レビュー後に人間が `verify_note.py` または UI の「承認する」ボタンで新しい `{ by: human:{{USER_ID}}, at: <now> }` を追記してゲートを通すと、自動的に承認済みへ復帰し `lineage.jsonl` へ `review_cleared` が記録される。

---

## 5. パス解決規則（OKF v0.3 §6）

`sources[].resource`、`superseded_by`、本文 Markdown リンク `[text](target)` / `![alt](target)`（コードフェンス・インラインコード内は除外）を以下の順で解決する。

1. URI スキーム（`^[A-Za-z][A-Za-z0-9+.-]*:`）で始まる → `"url"`（外部 URL、ローカル存在確認対象外）
2. `#` のみのフラグメント → 同一ファイル内アンカー（対象外）
3. パーセントデコード（`urllib.parse.unquote`）し、`#fragment` を除去
4. `/` 始まり → バンドルルート基準 `(pkm_root / path.lstrip("/")).resolve()`
5. それ以外 → ファイル自身のディレクトリ基準 `(file_path.parent / path).resolve()`
6. 解決結果が `pkm_root` 外に出る → `"external"`（エラー）
7. `pkm_root` 内だが実在しない → エラー

---

## 6. 決定論的ゲート（`gate_okf.py`）の7大検証ルール

| # | ルール名 | 検証内容 | 失敗時の扱い |
| :--- | :--- | :--- | :--- |
| 1 | **Bronze 汚染ガード** | `10_Inbox/` 配下のファイルに `type` キーが存在しないこと | エラー |
| 2 | **配置・命名・旧キー検証** | `20_Literature/` または `50_Knowledge/` のカテゴリフォルダ配下にあり、カテゴリ番号が範囲内（21〜29 / 51〜59）、ファイル名に禁止文字なし、`x-pkm-medal` が配置と一致、旧仕様キーなし、予約名（`index.md`/`log.md`）でない | エラー（`x-pkm-medal` のみ他にエラーがない場合に `--fix` で自動記入） |
| 3 | **Gold 承認・合成条件** | Gold は Silver/Gold を `sources` に1件以上含み、未承認配置でないこと | 未承認・直置きはエラー / 検証失効は `review_pending`（警告） |
| 4 | **アクター命名規則** | `generated.by` が `<producer>/<version>` または `human:<id>`、`verified[].by` が有効なアクター形式 | エラー |
| 5 | **リンク・参照実在検証** | `sources[].resource`、`superseded_by`、本文 Markdown リンクの参照先がバンドル内に実在すること | エラー |
| 6 | **タグ検証** | ケバブケース・15個以内・重複なし・予約語（`bronze`/`silver`/`gold`）禁止 | エラー |
| 7 | **人間記述の保護** | `git show HEAD:./<name>` と比較し、`generated.by` が `human:` でないのに `x-pkm-my-comment` が変更・削除された、または `verified` の既存プレフィックスが変更・削除・並べ替えされた | エラー |

---

## 7. `AGENTS.md`（エージェント行動規範）に必ず含める項目

バンドルルートに `AGENTS.md` を配置し、Claude Code / Antigravity / Hermes / Open WebUI 等すべてのエージェントに以下を明記する。

1. **基本原則:** 確率論と決定論の分離、必ず `gate_okf.py` を通すこと、人間記述の不変性。
2. **フォルダと役割一覧:** 各エリアの読み書き権限、`20_Literature/` のカテゴリ定義と境界の目安（`hermes.py` が `### 20_Literature のカテゴリ` 節を自動抽出してプロンプトに組み込むため、**この見出し構成を維持すること**）。
3. **アクター命名表:**
   - 自律ワーカー: `hermes/gemma4-12b`（または使用モデル名）
   - 対話エージェント: `claude-code/<model>`, `antigravity/<model>` 等
   - 人間: `human:{{USER_ID}}`
   - スクリプト: `process:gate-okf`, `process:build-index`, `process:lint-wiki`
4. **書いてよいキー・書いてはいけないキー表**（`x-pkm-medal`, `okf_version`, `jd_id`, `x-pkm-my-comment`, `verified` の扱い）。
5. **Silver → Gold の合成手順**（Silver は移動せず新ノートを作成、検証は人間のみ）。
6. **自動生成物の編集禁止リスト**（`index.md`, `log.md`, `lineage.jsonl`, `*.extracted.md`, `00_System/graph/`）。
7. **OKF 公式 Agent Skills（`okf-generate` / `okf-consume`）との併用ルール:**
   - 汎用の公式スキルよりローカルプロファイルが優先される差分表（`type` の3値限定、`description` 1〜3文、`index.md`/`log.md` はスキルで書かない、`gate_okf.py` を必須実行、脚注は `[^<id>]` 形式）。
8. **禁止事項サマリー。**

---

## 8. `.gitignore` と `.stignore` の必須除外項目

### `pkm/.gitignore`
- `90_Assets/*`（`!90_Assets/.gitkeep`）
- `20_Literature/{{PRIVATE_CATEGORY}}/`（私的カテゴリを使う場合）
- `00_System/graph/*`（`!00_System/graph/.gitkeep`）
- `*.extracted.md`
- `10_Inbox/15_files/**/*.{pdf,docx,xlsx,pptx,png,jpg,jpeg,webp}`
- `*.sqlite`, `*.sqlite3`, `*.sqlite-wal`, `*.sqlite-shm`, `*.sqlite-journal`, `*.db*`
- `00_System/logs/rejected/`
- `.obsidian/workspace*.json`, `.trash/`, `lost+found/`, `__pycache__/`, `*.pyc`, `.stfolder/`, `.claude/worktrees/`, `.claude/settings.local.json`

### `pkm/.stignore`
- `.git`
- `(?d)*.sqlite`, `(?d)*.sqlite-wal`, `(?d)*.sqlite-shm`
- `00_System/graph`
- `.obsidian/workspace*.json`
- `(?d)__pycache__`
- `.claude/worktrees`
