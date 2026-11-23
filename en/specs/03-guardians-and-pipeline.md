# 03. Deterministic Scripts & Autonomous Pipeline Specification (`00_System/scripts/`)

Defines the specifications for all 11 scripts in `pkm/00_System/scripts/` and `pkm/00_System/schemas/okf_v03.py`.
All scripts must run on Python 3.12–3.14 using only the core dependencies in `pyproject.toml` (`pydantic`, `pyyaml`, `openai`, `sqlite-vec`, `markitdown[pdf,docx,xlsx,pptx]`) — they must not import web-app dependencies.

---

## 1. Script Inventory & Responsibilities

| File | Role | Write Targets |
| :--- | :--- | :--- |
| `pkm_config.py` | Shared configuration loader for all scripts | None |
| `gemma_client.py` | OpenAI-compatible LLM / Embedding client + self-repair loop | Target note & `logs/rejected/` (only during `generate`) |
| `gate_okf.py` | OKF v0.3 deterministic gate (Rules 1–7) | `x-pkm-medal` on `--fix`, `00_System/logs/lineage.jsonl` |
| `lint_wiki.py` | Checks links, `[[wikilink]]`, orphan Gold notes, SHOULD keys | `pkm.sqlite` (`lint_runs` / `lint_issues`) on `--save-db` |
| `verify_note.py` | Human-only Gold approval script (appends `verified` + runs gate) | Target Gold note, `lineage.jsonl` |
| `build_index.py` | Sidecar extraction, `pkm.sqlite` sync, all-level `index.md`, lineage logging, `log.md` generation | `*.extracted.md`, `pkm.sqlite`, `index.md`, `lineage.jsonl`, `log.md` |
| `search.py` | Full-layer hybrid search CLI (FTS5 + sqlite-vec + RRF) | None (read-only) |
| `tag_stats.py` | Period-based tag ranking CLI for Silver / Gold | None (read-only) |
| `hermes.py` | Bronze watcher → SearXNG research → autonomous Silver generation worker | `20_Literature/`, `00_System/hermes.sqlite`, `lineage.jsonl` |
| `run_graphify.py` | Nightly knowledge graph batch restricted to gate-passing Silver / Gold | `00_System/graph/` |
| `autocommit.py` | Scheduled gated `git commit` & `push` | Git repository |

---

## 2. Shared Configuration (`pkm_config.py`)

### Resolution Priority
`CLI arguments` > `Environment variables` > `.env file` > `Defaults`

### Environment Variables & Defaults
| Variable | Default | Description |
| :--- | :--- | :--- |
| `PKM_ROOT` | Two levels above the script (`Path(__file__).resolve().parents[2]`) | Bundle root |
| `MAC_MINI_URL` | `http://localhost:8080` | OpenAI-compatible inference server URL. The `api_base` property appends `/v1`. **Keep this exact variable name regardless of hardware** (referenced by Compose, Open WebUI, and all scripts) |
| `PKM_API_KEY` | `local` | OpenAI-compatible API key (`Config.api_key`). `llama-swap` does not authenticate, so keep the default unless placing an authenticating proxy in front. Shared by `gemma_client`, MarkItDown, whisper (`Authorization: Bearer`), and Open WebUI |
| `PKM_DB` | `$PKM_ROOT/00_System/pkm.sqlite` | Search & lint DB path |
| `PKM_LLM_TIMEOUT` | `300.0` | LLM response timeout in seconds (accounts for cold model loading) |
| `PKM_MODEL_REVIEW` | `gemma4-review` | Primary reasoning model ID |
| `PKM_MODEL_FAST` | `gemma4-fast` | Fast / structured-output / VLM model ID |
| `PKM_MODEL_EMBED` | `gemma-embed` | Embedding model ID (`EMBED_DIM = 768`) |
| `PKM_ENV_FILE` | `./.env` → `~/.config/pkm/env` | `.env` file path (set to `/nonexistent` inside containers to disable host `.env` discovery) |

The `.env` parser handles `KEY=VALUE`, `export KEY=VALUE`, `# comments`, and surrounding quotes, and never overwrites already-set environment variables (`setdefault`).

---

## 3. Inference Client & Self-Repair Loop (`gemma_client.py`)

### 3.1 `GemmaClient`
- Wraps `openai.OpenAI(base_url=cfg.api_base, api_key=cfg.api_key, timeout=Timeout(cfg.timeout, connect=10.0), max_retries=2)`.
  - **Why separate the connect timeout to 10s:** Prevents systemd timers from hanging for 300s when the inference node is asleep or the VPN is disconnected.
- `chat(messages, model=None, temperature=None, max_tokens=None, **extra) -> str`:
  - Raises `GemmaError` if `content` is empty (e.g., when reasoning tokens exhausted `max_tokens`).
- `embed_documents(texts, titles=None)` / `embed_query(text)`:
  - Calls `/v1/embeddings` in batches of 16 and raises `GemmaError` if vector dimensionality != `EMBED_DIM` (768).
  - **EmbeddingGemma-Specific Prompt Prefixes (Critical):**
    - Document: `f"title: {title or 'none'} | text: {text}"`
    - Query: `f"task: search result | query: {text}"`
- `list_models() -> list[str]`: Returns model IDs from `/v1/models`.

### 3.2 Self-Repair Loop (`generate_okf_note`)
1. If the target file already exists, back up its original text.
2. Loop up to `max_attempts` times (default 3):
   - Strip code fences (` ```markdown ... ``` `) from the `client.chat(convo)` output and write it to the target path.
   - Validate via `gate_okf.run_gate([target], root, fix=False, use_git=use_git, write_lineage=False)` (never pollute `lineage.jsonl` with failed drafts).
   - If `status == "pass"` and `kind == "okf"`, run the gate one final time with `fix=True, write_lineage=True` and return `RepairResult(ok=True, attempts=attempt, ...)`.
   - On failure, append the draft as `assistant` and a bounce-back `user` message embedding `gate_okf`'s `errors` / `warnings` as JSON, then retry.
3. If all attempts fail:
   - Restore the target file to its original content (or delete it if newly created).
   - Save the final draft to `00_System/logs/rejected/<YYYYMMDDTHHMMSSZ>-<filename>` and return `RepairResult(ok=False, rejected_draft=...)`.

### 3.3 CLI
- `gemma_client.py check`: Verifies that `/v1/models` exposes all 3 models (review / fast / embed) and that `embed_query("疎通確認")` returns 768 dimensions (exit code 0/1).
- `gemma_client.py chat "<prompt>" [--model ...]`: One-shot chat.
- `gemma_client.py generate --source <bronze> --to <silver> [--actor ...] [--max-attempts 3]`: One-shot Silver generation.

---

## 4. Deterministic Gate (`gate_okf.py`)

See [02-okf-profile-and-vault.md](02-okf-profile-and-vault.md) §6 for full rule details.

### 4.1 Processing Specification
- During bulk scans, skip hidden dot-directories/files, top-level `00_System` and `90_Assets`, and root `log.md`.
- Per-file decision flow:
  1. Root `index.md` → validate with `BundleIndexMeta`.
  2. Under `10_Inbox/` → `check_bronze_frontmatter()` (error if `type` key exists).
  3. `"type" not in raw` → `kind="non-okf"`, `status="skipped"` (`README.md`, `AGENTS.md`, `60_Output/`, etc.).
  4. Reserved filename (`index.md` / `log.md`) in a subdirectory containing `type` → error.
  5. Validate with `PKMFrontmatter.model_validate(raw)`. Even if Pydantic raises errors, also run placement checks (path validation) and combine all errors so the LLM can self-repair in a single round-trip.
  6. `validate_file_context()` (separate `x-pkm-medal` mismatch as a `--fix` candidate).
  7. `find_broken_body_links()` (existence of body Markdown links and images).
  8. When `use_git=True`, fetch the previous version via `git show HEAD:./<name>` and run `check_protected_fields(old_raw, raw)`.
  9. **Only if there are no other errors**, when `--fix` is set, write `x-pkm-medal: silver|gold` via raw YAML line editing (`set_medal_in_text`).
- **Appending to `lineage.jsonl` (`write_lineage=True` and passing Silver / Gold only):**
  - Extract event evaluation into `lineage_events(path, result, meta, state, now)` and writing into `record_lineage(root, gated, now)`, shared by `run_gate` and `build_index.py`.
  - **Mutual exclusion:** `record_lineage` opens `lineage.jsonl` in append mode, acquires `fcntl.flock(LOCK_EX)`, and **re-reads current state after acquiring the lock** before deciding and writing events (prevents interleaved writes or duplicate `promoted` events when the Hermes container and host `build_index.py` run concurrently).
  - Never record anything for files where `git check-ignore -q` returns 0 (e.g., `20_Literature/28_me/`).
  - First time a Silver / Gold note passes (once per path): `{"at", "event": "promoted", "by": "process:gate-okf", "path", "title", "medal": "silver"|"gold", "jd_id", "generated": {by, at}, "sources": [...]}`
    - For Gold, passing requires `human:` verification, so this doubles as the "initial approval" record. Set `at` to the latest `human:` verification timestamp and include `"verified": {by, at}` (keeps `log.md` dates accurate even when backfilling existing Gold notes).
  - When Gold transitions into `review_pending`: `{"at", "event": "review_pending", "by": "process:gate-okf", "path", "title", "generated": {by, at}, "expired_verification": {by, at}}`
  - When Gold recovers from `review_pending` to `human-reviewed`: `{"at", "event": "review_cleared", "by": "process:gate-okf", "path", "title", "verified": {by, at}}`

### 4.2 CLI & Exit Codes
- `gate_okf.py [paths...] [--root DIR] [--fix] [--json] [--no-git] [--no-lineage] [-v]`
- Exit codes: `0` = all passed (including warnings / review-pending), `1` = errors found, `2` = argument / root error.

---

## 5. Wiki Linter (`lint_wiki.py`)

Never modifies any files. Performs cross-bundle health inspection.

### 5.1 Checks & Severities
| Code | Target & Condition | Severity |
| :--- | :--- | :--- |
| `frontmatter` | Frontmatter YAML parse failure | `error` |
| `broken-link` | Broken or out-of-bundle Markdown link / image.<br>**Exemption:** `10_Inbox/11_rss/`, `10_Inbox/12_web-clipper/`, `10_Inbox/14_podcast/`, and `10_Inbox/19_private/` frequently inherit broken external links, so missing targets are not flagged there (`13_quick-notes` and `15_files` ARE checked) | `error` for OKF notes, `warning` for non-OKF |
| `broken-wikilink` | Broken or out-of-bundle `[[target]]` / `![[embed]]` (Obsidian-style shortest-path & extensionless resolution) | `error` for OKF notes, `warning` for non-OKF |
| `ambiguous-wikilink` | `[[name]]` matches multiple same-named files (still ambiguous even after same-folder preference) | `warning` |
| `broken-source` | Missing or out-of-bundle target in `sources[].resource` / `superseded_by` | `error` for OKF notes, `warning` for non-OKF |
| `orphan` | **Gold note** never referenced by any other note (links from `index.md` do not count; Silver notes are normally unreferenced until cited by Gold, so excluded) | `warning` |
| `missing-should` | Silver / Gold missing or empty `tags` or `sources` (`x-pkm-my-comment` is optional and never warned) | `warning` |

### 5.2 SQLite Persistence (`--save-db`)
Creates and writes the tables below in `pkm.sqlite` for the dashboard, retaining only the latest 30 runs (`KEEP_RUNS = 30`).

```sql
CREATE TABLE IF NOT EXISTS lint_runs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    at TEXT NOT NULL,
    files INTEGER NOT NULL,
    notes INTEGER NOT NULL,
    errors INTEGER NOT NULL,
    warnings INTEGER NOT NULL,
    by_code TEXT NOT NULL DEFAULT '{}'
);
CREATE TABLE IF NOT EXISTS lint_issues (
    run_id INTEGER NOT NULL REFERENCES lint_runs(id) ON DELETE CASCADE,
    path TEXT NOT NULL,
    severity TEXT NOT NULL,
    code TEXT NOT NULL,
    message TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS lint_issues_run ON lint_issues(run_id);
```

### 5.3 CLI & Exit Codes
- `lint_wiki.py [--root DIR] [--json] [--strict] [--save-db]`
- Exit codes: `0` = no errors, `1` = errors found (or warnings under `--strict`), `2` = argument error.

---

## 6. Human-Only Gold Approval Script (`verify_note.py`)

Must never be called by an LLM or AI agent. Executed solely by the human via CLI or the Web UI "Approve" button.

1. Verify `by` starts with `human:` and target is an existing file under `50_Knowledge/`.
2. **Pre-approval gate:** Before appending, run `gate_okf.run_gate([rel], root, use_git=..., write_lineage=True)` once so any `review_pending` not yet picked up by `build_index.py` is recorded first (without this, `review_cleared` would not be written after approval).
3. **Raw YAML line editing (`append_verification_text`):** Append `- by: <by>\n  at: "<RFC3339>"\n` to `verified` (matching indentation for block style; converting flow style or empty values to block style; includes a safety assertion confirming all keys other than `verified` match identically and the existing event prefix is untouched).
4. Run `gate_okf.run_gate([rel], root, use_git=..., write_lineage=True)`.
5. If passed, finalize (records `review_cleared` when recovering from review-pending, or Gold `promoted` on initial approval). If failed, roll back to the original file content and exit `1`.

CLI: `verify_note.py <path> --by human:{{USER_ID}} [--root DIR] [--no-git] [--json]` (exit codes 0/1/2)

---

## 7. Deterministic Builder (`build_index.py`)

### 7.1 Processing Steps (5 Stages)

1. **Sidecar Extraction (`extract_sidecars`):**
   - Target extensions under `10_Inbox/`: `.pdf`, `.docx`, `.xlsx`, `.pptx`, `.png`, `.jpg`, `.jpeg`, `.webp`.
   - Output path: `<filename>.extracted.md` in the same directory (non-OKF: must NOT contain `type`).
   - Extract text / OCR via `MarkItDown(llm_client=OpenAI(base_url=cfg.api_base, api_key=cfg.api_key), llm_model=cfg.model_fast)`.
   - Record the source file's SHA256 in `x-pkm-source-sha256` and skip re-extraction if unchanged.
   - Source modified date: For Office (OOXML) files, prefer `dcterms:modified` from `docProps/core.xml` (valid only if year ≥ 1980), falling back to file `mtime`. Store in `x-pkm-source-modified` and `title: "YYYY-MM-DD <filename>"` (never rename the file itself).
   - Automatically remove orphaned sidecars whose source file was deleted (only when `x-pkm-extracted-from` matches).
   - A failure on one file must not halt the batch (record in `extract_failed`).

2. **SQLite Incremental Sync (`pkm.sqlite`):**
   - Target areas: All `.md` files under `10_Inbox`, `20_Literature`, `50_Knowledge` (excluding reserved `index.md` / `log.md`).
   - For Silver / Gold, index ONLY notes passing `gate_okf.gate_file(..., fix=False, use_git=False)` (Single Source of Truth). Record failing notes in `skipped_failing_notes` and skip them.
   - **Chunking (`chunk_markdown`):** Split by heading hierarchy (e.g., `"Design > Search"`, ignoring `#` inside code fences); sections exceeding `CHUNK_CHARS = 1200` chars (safely within EmbeddingGemma's 2048-token limit) are further split by paragraphs → character count.
   - **Trust Tier Weights:**
     - `Gold (human-reviewed & not review_pending)` = `1.5`
     - `Silver` / `Gold (review_pending)` = `1.2`
     - `Bronze` = `1.0`
   - **SQLite Schema (WAL mode, `PRAGMA foreign_keys = ON`):**

```sql
CREATE TABLE IF NOT EXISTS meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);
CREATE TABLE IF NOT EXISTS files (
    path TEXT PRIMARY KEY,
    sha256 TEXT NOT NULL,
    layer TEXT NOT NULL,             -- bronze | silver | gold
    kind TEXT NOT NULL,              -- bronze | sidecar | note
    medal TEXT,
    jd_id TEXT,
    title TEXT NOT NULL,
    description TEXT NOT NULL DEFAULT '',
    tags TEXT NOT NULL DEFAULT '[]', -- JSON array
    trust_tier TEXT,
    review_pending INTEGER NOT NULL DEFAULT 0,
    lang TEXT NOT NULL DEFAULT 'ja',
    weight REAL NOT NULL,
    generated_by TEXT,
    generated_at TEXT,
    indexed_at TEXT NOT NULL,
    first_seen_at TEXT               -- First-indexed timestamp (for weekly & tag period stats; immutable on updates)
);
CREATE INDEX IF NOT EXISTS files_first_seen ON files(first_seen_at);

CREATE TABLE IF NOT EXISTS chunks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    path TEXT NOT NULL REFERENCES files(path) ON DELETE CASCADE,
    ord INTEGER NOT NULL,
    heading TEXT NOT NULL DEFAULT '',
    text TEXT NOT NULL,
    embedded INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX IF NOT EXISTS chunks_path ON chunks(path);

CREATE VIRTUAL TABLE IF NOT EXISTS chunks_fts USING fts5(title, heading, text, tokenize='trigram');
CREATE VIRTUAL TABLE IF NOT EXISTS vec_chunks USING vec0(embedding float[768] distance_metric=cosine);
```

   - **Incremental Updates & Fault Tolerance:**
     - If `files.sha256` is unchanged, update metadata only and keep existing chunks.
     - Embed `chunks.embedded = 0` chunks in batches of 32 via `gemma-embed`, insert into `vec_chunks`, and commit. If the inference server is unreachable, log the error and finish cleanly so FTS search stays usable (remaining chunks will be backfilled on the next run).

3. **All-Level `index.md` Generation (`write_indexes`):**
   - Target areas: `10_Inbox`, `20_Literature`, `50_Knowledge`, `60_Output`, and root.
   - **Write only when content changed (`write_if_changed`)** to avoid noisy Syncthing/Git diffs.
   - Root `index.md`: Preserve frontmatter (`okf_version: "0.3"`, etc.) and handwritten section above the auto-gen marker; render area listing and "Review Pending" list below.
   - Area `index.md`: List existing category folders (`NN_kebab-case`) with item counts (`（N 件）` in the original template). Preserve existing labels and descriptions.
   - Category `index.md`:
     - Silver / Gold: `## ノート` (Notes), `## タグ別` (By Tag, `### <tag>`), `## 再レビュー待ち` (Review Pending, when applicable).
     - Bronze / Output: `## ファイル` (Files; wrap paths containing spaces or parentheses in `<...>`).

4. **Lineage Recording (`gate_okf.record_lineage`):**
   - Pass all gated `(path, result, meta)` Silver / Gold tuples from Step 2 to record initial passes (`promoted`) and Gold review-pending transitions (`review_pending` / `review_cleared`). Direct Gold edits by agents are caught here.
   - Run **before** `log.md` generation so the same run reflects them in `log.md`. Disable with `--no-lineage`.

5. **Root `log.md` Generation (`render_log`):**
   - Read `00_System/logs/lineage.jsonl` and output under `# Directory Update Log` grouped by `## YYYY-MM-DD` (descending):
     - `promoted` (Silver) → `* **Creation**: [title](path) generated by <generated.by>`
     - `promoted` (Gold) → `* **Creation**: [title](path) (Gold) approved by <verified.by>`
     - `review_cleared` → `* **Verification**: ...`, `review_pending` → `* **Review Pending**: ...`

### 7.2 CLI
`build_index.py [--root DIR] [--db PATH] [--no-extract] [--no-embed] [--no-index-md] [--no-log] [--no-lineage] [--rebuild] [--json]` (exit codes: `0` = OK, `1` = one or more sidecar extractions failed)

---

## 8. Full-Layer Hybrid Search CLI (`search.py`)

- **Full-Text Search (`fts_search`):** Whitespace-delimited AND search.
  - Terms ≥ 3 chars: `chunks_fts MATCH '"term1" AND "term2"'`, ordered by `bm25(chunks_fts, 5.0, 2.0, 1.0)` (title > heading > text).
  - **Terms < 3 chars (trigram workaround):** Combine with `LIKE '%term%' ESCAPE '\'` against `c.text` and `f.title`.
- **Vector Search (`vec_search`):** Embed the query via `gemma-embed` (capped at 60s timeout) and query `vec_chunks WHERE embedding MATCH ? AND k = ?` (fetch `k = limit * 4` when layer filtering and filter afterward). If the inference server is unreachable, return a warning and automatically fall back to FTS-only.
- **Fusion (`fuse`):**
  - Over the top 50 candidates (`CANDIDATES = 50`) from each method, compute Reciprocal Rank Fusion (`RRF_K = 60`: `1 / (60 + rank + 1)`) and multiply by `files.weight` (Gold 1.5 / Silver & review-pending Gold 1.2 / Bronze 1.0).
  - Deduplicate multiple chunks from the same file by keeping the highest-scoring chunk and merging `via` (`["fts", "vec"]`).
- **CLI:** `search.py [query] [--root DIR] [--db PATH] [--mode hybrid|fts|vec] [--layer bronze|silver|gold] (repeatable) [-n 10] [--review-pending] [--json]` (exit codes: 0 = hits found / 1 = no hits / 2 = error).

---

## 9. Tag Rankings (`tag_stats.py`)

- Opens `pkm.sqlite` read-only (`file:<path>?mode=ro`) and aggregates tags over `files` rows where `kind = 'note' AND layer IN ('silver', 'gold')` grouped by `first_seen_at`.
- Periods:
  - `week`: Last 7 days vs. the preceding 7 days (`delta = current - previous`)
  - `30d`: Last 30 days vs. the preceding 30 days
  - `all`: All time (no previous-period delta)
- Ties break alphabetically by tag name. CLI: `tag_stats.py [--period week|30d|all] [--limit 10] [--json]`.

---

## 10. Autonomous Silver Generation Worker (`hermes.py`)

A self-contained Python worker with no external agent framework dependency (actor name: `hermes/gemma4-12b`).

### 10.1 State DB (`00_System/hermes.sqlite`)
```sql
CREATE TABLE IF NOT EXISTS hermes_items (
    path TEXT PRIMARY KEY,
    sha256 TEXT NOT NULL,
    status TEXT NOT NULL,          -- ok | failed | skipped
    silver TEXT,
    category TEXT,
    attempts INTEGER NOT NULL DEFAULT 0,
    repairs INTEGER NOT NULL DEFAULT 0,
    researched INTEGER NOT NULL DEFAULT 0,
    error TEXT,
    rejected TEXT,
    at TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS hermes_items_at ON hermes_items(at);
```

### 10.2 Processing Flow
1. **Candidate Selection (`find_candidates`):** Scan `.md` files under `10_Inbox/`, excluding:
   - `index.md` / `log.md` and hidden paths
   - Files already referenced in `sources[].resource` of any note in `20_Literature/` (avoids duplicating notes the human already turned into Silver manually)
   - Files modified within `SETTLE_SECONDS` (default 600s; prevents colliding with active edits/writes)
   - Files already recorded in `hermes_items` with the same `sha256` (automatically retries if edited)
   - Files whose frontmatter contains a `type` key, or empty files
2. **Placement Planning (`plan_note`):**
   - **For private Bronze (`10_Inbox/19_private/`), skip LLM placement planning:** lock category to `{{PRIVATE_CATEGORY}}` (`28_me`), derive the slug from the filename by stripping the Keep date prefix (`YYYY-MM-DD-`, falling back to `note-<sha10>`), and produce no search `query`.
   - Inject existing `20_Literature/` categories (descriptions read from `20_Literature/index.md` per 02 §2.2) and the `### 20_Literature のカテゴリ` section of `AGENTS.md` (boundary guidelines, up to the next `##`/`###` heading or EOF) into the system prompt.
   - Ask `gemma4-fast` to output JSON `{"category": "...", "slug": "...", "query": "..."}` (fallbacks: `29_misc` and `note-<sha10>`).
   - Resolve filename collisions by appending `-2`, `-3`, etc.
3. **Background Research (`searxng_search`):**
   - When `SEARXNG_URL` and `plan.query` are present and the item is **NOT private Bronze**, fetch top 5 results from `/search?q=<query>&format=json&language=ja` (20s timeout). On failure, log a warning and proceed with Silver generation. Never send terms from private Bronze to an external search engine.
   - Prompt-injection defense: explicitly instruct the model to ignore instructions embedded in search results and to add the corresponding URL to `sources` whenever adding facts not in the Bronze note.
4. **Generation + Self-Repair:**
   - Pass any Bronze `x-pkm-my-comment` only as a hint of user interest, and **forbid writing `x-pkm-my-comment` into the Silver note** (human-only key).
   - Execute `gemma_client.generate_okf_note(..., max_attempts=3)`.
   - On pass: record `status='ok'`, `repairs = attempts - 1` in `hermes.sqlite`.
   - On failure: record `status='failed'` and the `rejected` draft path.
   - **On inference server connection error:** Do NOT record as failed; abort the current batch immediately and retry next cycle.

### 10.3 CLI
- `hermes.py --once [--limit 5] [--dry-run]`
- `hermes.py --watch [--interval 300] [--limit 5]` (execution mode for the `hermes-agent` Docker container; `--interval` / `--limit` default to `HERMES_INTERVAL` / `HERMES_LIMIT`)
- Common flags: `--root DIR`, `--settle 600`, `--model <id>` (default `gemma4-fast`), `--searxng <url>` (default `SEARXNG_URL`; empty disables research), `--no-git`
- `hermes.py --status` (all-time & last-7-day ok/failed/repair counts and list of failures)
- `hermes.py --retry <10_Inbox/...>` (deletes the row from the state DB so it is re-queued)

---

## 11. Nightly Knowledge Graph Batch (`run_graphify.py`)

- **Strict Input Filtering:** Scans ONLY `20_Literature/` and `50_Knowledge/`, selecting only notes that pass `gate_okf.run_gate(..., fix=False, write_lineage=False)` (completely excludes Bronze and failing notes).
- **Change Detection:** Computes `input_sha256` from all passing notes' `(rel_path, sha256)`; if it matches `00_System/graph/manifest.json`, does nothing (`--force` to rebuild).
- **Deterministic Graph (`00_System/graph/graph.json`):** Always generated.
  - Nodes: Notes (`kind: "note"`, `title`, `type`, `medal`, `layer`, `category`, `tags`, `description`) + Tags (`id: "tag:<name>"`, `kind: "tag"`).
  - Edges: `tagged` (note → tag), `source` (`sources[].resource` → note), `link` (body Markdown links & unambiguous `[[wikilink]]` → note).
- **Optional External Graphify CLI Integration:**
  - CLI: `run_graphify.py [--root DIR] [--force] [--graphify-cmd CMD] [--no-git] [--json]` (exit codes: `0` = OK or unchanged, `3` = Graphify failed).
  - If `GRAPHIFY_CMD` (or `--graphify-cmd`, e.g., `graphify {input} --output {output}`) is configured, copies only passing notes into `00_System/graph/input/` and runs the command (3-hour timeout). On failure, clears `input_sha256` in `manifest.json` so it retries the following night (exit code 3).

---

## 12. Scheduled Gated Auto-Commit (`autocommit.py`)

Runs nightly to commit and push changes so `gate_okf.py` Rule 7 (`git show HEAD:<path>`) stays effective.

1. If `MERGE_HEAD`, `rebase-merge`, `rebase-apply`, `CHERRY_PICK_HEAD`, or `REVERT_HEAD` exists, assume a human operation is in progress and do nothing (exit code 0).
2. `git add -A` (respecting `.gitignore`).
3. Validate only staged `.md` files under `20_Literature/` and `50_Knowledge/` (excluding reserved names) via `gate_okf.run_gate(..., use_git=True, write_lineage=False)`.
4. **Unstage failing files with `git reset -q -- <path>`** so invalid edits are never baked into `HEAD`.
5. Commit remaining changes (including deletions) with message `auto: YYYY-MM-DD HH:MM <local offset %z> (N files)` (e.g., `+0900`) plus up to 50 listed paths. Under `--dry-run`, print the message without committing.
   - Does not write lineage (`write_lineage=False`; `build_index.py` owns lineage logging).
6. Unless `--no-push`, if ahead of upstream, run `git push -q` (on failure, keep the commit and exit `3` to retry next time).
7. Exit codes: `0` = OK, `1` = gate failures unstaged (valid changes committed), `2` = not a Git repo, `3` = push failed.
8. CLI: `autocommit.py [--root DIR] [--no-push] [--dry-run]`.
