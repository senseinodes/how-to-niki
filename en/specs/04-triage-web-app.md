# 04. Triage & Unified Search Web App Specification (`apps/rss_triage/`)

Specification for a lightweight web application unifying daily information intake (RSS / non-RSS websites / Podcasts), AI first-pass triage, full-layer PKM hybrid search, Gold re-review approval, operational dashboard, and PWA mobile web clipping.

---

## 1. Technical Constraints & Core Design

- **Backend:** Python 3.13 (`python:3.13-slim` + `git` + `tzdata`) + FastAPI (`create_app` factory pattern) + Uvicorn + Jinja2 + httpx + feedparser + trafilatura + BeautifulSoup4.
- **Frontend:** HTMX (vendored locally at `static/vendor/htmx.min.js`) + Vanilla CSS (CSS Variables + Grid / Flexbox, light & dark themes) + minimal Vanilla JS (`static/app.js`). Zero build tools.
- **Reusing PKM Scripts:** Do not duplicate PKM scripts into the Docker image; append the container-mounted `{{PKM_ROOT}}/00_System/scripts/` to `sys.path` and import them dynamically (`pkm.py`).
- **Database Separation:**
  - `00_System/rss_triage.sqlite`: App's own read-write DB (WAL mode, lock-protected).
  - `00_System/pkm.sqlite`: Opened **read-only** (`file:<path>?mode=ro`); the app must stay running cleanly even if missing or corrupted.
  - `00_System/hermes.sqlite`: Opened read-only only during dashboard aggregation.
  - `00_System/links.sqlite`: Read and written only through `related_links.py` functions from the Links screen (the count badge opens it read-only). The screen still renders (empty) if the script or DB is missing.
  - Never write `index.md`, `log.md`, or `lineage.jsonl` directly from the app (call `verify_note.py` for approvals and `related_links.py` for linking / undoing).
- **Timezone:** Screen timestamps and dashboard weekly boundaries are determined via `zoneinfo.ZoneInfo` from the `TZ` env var (default `Asia/Tokyo`, via `settings.local_tz()`). Install `tzdata` in the image; if unavailable, fall back to UTC+9 for `Asia/Tokyo` and UTC otherwise. Never hardcode offsets.
- **Scheduler:** Inside FastAPI's `lifespan`, launch a background loop ticking every 15 minutes (`SCHEDULER_TICK = 900s`) that crawls only sources whose `interval_min` has elapsed (disable via `NIKI_SCHEDULER=0`). On startup, reset any interrupted Podcast `running` states back to `queued` and start the transcription worker.

---

## 2. App State DB Schema (`00_System/rss_triage.sqlite`)

```sql
CREATE TABLE IF NOT EXISTS app_meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);

CREATE TABLE IF NOT EXISTS sources (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    kind TEXT NOT NULL DEFAULT 'rss',        -- rss | site | podcast
    name TEXT NOT NULL,
    url TEXT NOT NULL UNIQUE,
    site_url TEXT,
    enabled INTEGER NOT NULL DEFAULT 1,
    private INTEGER NOT NULL DEFAULT 0,      -- 1 = private source (Share forbidden)
    interval_min INTEGER NOT NULL DEFAULT 120,
    selector TEXT,                           -- CSS selector JSON for non-RSS sites
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
    content TEXT,                            -- trafilatura-extracted body (Markdown)
    image_urls TEXT NOT NULL DEFAULT '[]',   -- JSON array (max 10)
    char_count INTEGER NOT NULL DEFAULT 0,
    score INTEGER,                           -- Interest score 0–100 (NULL if unscored); weighted sum of axes
    summary TEXT,                            -- 3-line summary JSON array
    axes TEXT,                               -- Per-axis values (JSON; NULL = legacy direct score)
    reason TEXT,                             -- Scoring rationale (1–2 sentences)
    is_breaking INTEGER NOT NULL DEFAULT 0,  -- Breaking news (kept out of the score)
    score_error TEXT,                        -- Retry prefix 'x1:...' etc. (max 3 attempts)
    scored_at TEXT,
    status TEXT NOT NULL DEFAULT 'unread',   -- unread | kept | dismissed | archived
    comment TEXT NOT NULL DEFAULT '',        -- Primary reaction (written to x-pkm-my-comment on Keep)
    kept_path TEXT,                          -- Relative Bronze path written on Keep
    acted_at TEXT,
    audio_url TEXT,                          -- Podcast enclosure URL
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

## 3. Module-by-Module Functional Specification

### 3.1 Feed Fetching & OPML (`feeds.py`)
- Conditional GET via `ETag` / `If-Modified-Since` (skip processing on `304 Not Modified`).
- Auto-detect Podcasts when an enclosure has `audio/*` or `.mp3`/`.m4a`, recording `audio_url`, `itunes_duration`, and `<podcast:transcript>` (`podcast_transcript` / `rel="transcript"`). **Never download audio files.**
- OPML 2.0 import (flattens nested `<outline>` tags, recognizes `type="podcast"`) and export (excludes non-RSS sites).

### 3.2 Full-Text Extraction & `robots.txt` (`extract.py`)
- Per-origin cached `RobotsCache` (allows on 404 etc., denies all on 401/403, sends an explicit User-Agent).
- Extract article bodies via `trafilatura.extract(..., output_format="markdown", include_images=True, include_links=False, favor_precision=True)` (40,000-char cap) + metadata image extraction.
- Discard extracted text if it is shorter than the feed summary.

### 3.3 Non-RSS Sites & AI Selector Estimation (`sites.py`)
- Classify trend aggregator hosts (`TREND_HOSTS`, e.g., Togetter / posfie) as `item.kind = "trend"`.
- **HTML Simplification (`simplify_html`):** Strip `script/style/noscript/svg/iframe/head/link/meta/form/picture/source` and comments, keep only `class/id/href/datetime` attributes (truncated to 80 chars each), and cap at 14,000 chars.
- **AI Estimation (`estimate_selector`):** Send simplified HTML to `gemma4-fast` to infer CSS selector JSON `{"item": "...", "title": "...", "link": "...", "date": "..."}` (on re-estimation, pass the broken previous selector as a hint).
- Before registration, always let the human review and tweak extracted results on a **test-fetch preview screen**.
- If a crawl yields 0 matches or a syntax error, set `last_status = "selector-mismatch"` and display a "Re-estimate selector with AI" button in the sources list.

### 3.4 Per-Axis Interest Score & 3-Line Summary (`scoring.py`)
When an LLM assigns 0–100 directly, scores cluster around 65–85 and thresholds stop working. So **the LLM returns only discrete per-axis values, and code combines them into 0–100**.

- **Axes and rubrics:** `relevance` (match with interests, 1–5), `originality` (originality / primary-source value, 1–5), `utility` (usefulness / information density, 1–5), `noise_penalty` (0–3). The system prompt defines each level by observable evidence (presence of primary data, code, numbers, concrete trade-offs, etc.).
- **Breaking news is a separate flag:** `is_breaking` (critical CVEs, breaking changes or major releases of key tools, sudden policy/API changes — information whose value drops sharply within a week). It is never mixed into the score; it drives a badge and a "Breaking" filter.
- **Interest baseline:** the top tags attached to Silver / Gold notes in `pkm.sqlite` over the last 30 days (`tag_stats` `30d`, with counts; default `NIKI_SCORE_TAG_LIMIT=20`) are put in the prompt. Overly broad tags are removed via `NIKI_SCORE_STOP_TAGS` (comma-separated, case-insensitive). With no tags yet, the prompt says to score relevance from a general technologist's perspective centered on 3. **Keep / Dismiss history is not used** (Dismiss also means "already known", "duplicate", or "not now", which causes mislearning).
- **Output JSON (in this key order — writing the summary and reason first anchors the scores to that evidence):** `{"summary": ["line 1", "line 2", "line 3"], "reason": "1–2 sentences", "axes": {"relevance": 1-5, "originality": 1-5, "utility": 1-5, "noise_penalty": 0-3}, "is_breaking": true|false}` (each summary line ≤ 60 chars, in `{{PRIMARY_LANG}}` even for foreign-language articles). The body is truncated to 4,000 chars. `temperature=0.2`, `max_tokens=800`.
- **Parsing:** values are clamped into range. A missing `noise_penalty` defaults to 0; any other missing axis is a parse error.
- **Combination:** `score = clamp(0, 100, round(Σ w_k × axes_k − w_noise × noise_penalty))`. Default weights `NIKI_SCORE_WEIGHTS=relevance=8,originality=6,utility=6,noise_penalty=15` (max 5/5/5/0 → 100, average 3/3/3/0 → 60, each noise point −15).
- **Changing weights:** at startup, compare with `app_meta.score_weights`; if they changed, recompute every item from its stored `axes` **without calling the LLM** (`sync_weights`). Manually: `python -m rss_triage.crawler --rescore`. Legacy items without `axes` are left as is.
- On parse failure, prompt once to re-output valid JSON; cap consecutive parse failures at 3 per item (`x3:...`). If the inference server is unreachable, stop scoring for this crawl run and defer to the next cycle.

### 3.5 Crawl Orchestration (`crawler.py`)
1. Insert a start row into `crawl_runs`.
2. Crawl enabled sources that are due (`interval_min` elapsed) or when `force=True` (cap initial fetch at latest 30 items to avoid an unread flood).
3. Run `fill_content()` on newly fetched `article` / `trend` items.
4. Score up to 200 unscored unread items via `score_pending()`.
5. Auto-archive unread items untouched for `NIKI_ARCHIVE_DAYS` (default 7 days) (`status='archived'`, never deleted).
6. Also provide a one-shot CLI: `python -m rss_triage.crawler --once [--force] [--no-score]`, plus `--rescore` to only recompute scores with the current weights.

### 3.6 Triage & Bronze Writing (`triage.py`, `inbox.py`)
- **Keep:** Save non-Podcast items to `10_Inbox/11_rss/YYYY-MM-DD-<slug>.md` (use `open(..., "x")` so files are never overwritten; append `-2`, `-3` on collision).
  - **Articles from private sources (`sources.private == 1`) are saved to `10_Inbox/19_private/`** (determined by `inbox.inbox_dir(kind, private)`: Git-ignored, locked to `28_me` in Hermes, no SearXNG lookup).
  - Frontmatter: `title`, `source`, `author?`, `published?`, `created`, `x-pkm-source-name`, `x-pkm-collector: rss-triage-app`, `x-pkm-score?`, `x-pkm-summary?`, `x-pkm-my-comment?` (**never include a `type` key**).
- **Dismiss / Bulk Dismiss:** Update status to `dismissed` and pass the `actions` ID(s) to the toast's "Undo" button.
- **Undo / Restore:** Revert Dismiss / Archive. On Restore, set the original action's `actions.undone = 1` so it drops out of dashboard aggregates (this week's Keep / Dismiss counts).

### 3.7 Podcast Lazy Transcription Worker (`podcast.py`)
- Clicking `[Send to PKM]` sets `transcript_status = 'queued'`. A lock-protected background thread processes one episode at a time.
- Priority order:
  1. `transcript_url` present → fetch and parse by format (Podcasting 2.0 JSON `segments[]`, WebVTT / SRT stripping timestamps while keeping `<v Speaker>`, HTML/plain text). Set `transcript_method = "published"`.
  2. `audio_url` present → stream download into a `tempfile.TemporaryDirectory` (600MB cap) → `POST ${MAC_MINI_URL}/v1/audio/transcriptions` (`model="whisper"`, `language="ja"` (`{{PRIMARY_LANG}}`), `response_format="json"`, header `Authorization: Bearer ${PKM_API_KEY}`, 3600s read / 10s connect timeout) → **guaranteed deletion of the audio file when the temp dir exits**. Set `transcript_method = "whisper"`.
- Save result to `10_Inbox/14_podcast/YYYY-MM-DD-<slug>.md` (or `10_Inbox/19_private/` for private sources) with show notes + transcript body, `x-pkm-transcript` / `x-pkm-duration-sec`, and no `type` key; update `status = 'kept'`, `transcript_status = 'done'`.
- On failure, record `transcript_status = 'failed'` and `transcript_error`, allowing retry from the UI.

### 3.8 External Sharing (`share.py`)
- Destinations:
  - **Hatena Bookmark:** `POST https://bookmark.hatenaapis.com/rest/1/my/bookmark` (self-implemented OAuth 1.0a HMAC-SHA1 signing). Enabled only when all 4 `HATENA_*` variables are set.
  - **X:** Official CLI `xurl -X POST /2/tweets -d '{"text": ...}'` (280-char limit check). Enabled only when `NIKI_XURL` is executable.
- **Safety Guards:**
  - Reject unless confirmation dialog passes `confirm="yes"`.
  - Unconditionally refuse sharing if `sources.private == 1` OR `kept_path` starts with any prefix in `PRIVATE_PREFIXES = ("10_Inbox/19_private/", "20_Literature/{{PRIVATE_CATEGORY}}/")` (centralized in `share.is_private_path`, enforced in both dialog rendering and POST submission). Because private keeps go into `19_private/`, the path check provides defense-in-depth independent of the source flag.

### 3.9 PWA Mobile Share Clipping (`clip.py`)
- Configure `share_target` (`GET /share-target?title=&text=&url=`) in `manifest.webmanifest` and a minimal Service Worker (`/sw.js`, caching static assets only while always fetching page HTML from the network).
- Because Android share sheets often embed the URL inside `text`, extract the URL via `pick_url(url, text)`.
- On confirmation form submission (with optional primary comment), extract the body via `trafilatura` and save to `10_Inbox/12_web-clipper/YYYY-MM-DD-<slug>.md` (`x-pkm-collector: rss-triage-app/share-target`, no `type` key).

---

## 4. All 6 Screens UI Specification & HTTP Endpoints

### 4.1 Shared Layout & UX
- **Left Navigation (bottom tab bar below 768px):** Triage (unread count badge) / Search / Links (undecided candidate count badge) / Re-review (warning-colored badge counting review-pending plus awaiting-first-approval Gold, refreshed via `hx-trigger="load, every 5m"`) / Dashboard / Sources.
- **Header:** Last crawl time, next crawl time, inference server status (caches `llama-swap` `/running` for 60s: `busy` / `idle` / `down`), light/dark theme toggle (persisted in `localStorage`).
- **Keyboard Shortcuts (disabled while an input is focused):**
  - `j` / `k`: next / previous card, `Enter`: open article body, `s`: Keep / Send to PKM, `d`: Dismiss, `u`: Undo latest toast, `/`: focus search or navigate to `/search`, `?`: toggle shortcut help dialog.
- **Mobile Responsive (`< 768px`):** Switch 2-pane layout (list | detail) into 1-pane switching (list ⇄ detail + Back button).

### 4.2 Overview of the 6 Screens
1. **Triage (`/triage`):**
   - Left pane: kind filter (All / RSS & Web / Trends / Podcast), score filter (All / 50+ / 80+ / Breaking), Unread/Dismissed toggle, "Bulk Dismiss < 50", "Crawl Now", score-descending card list (breaking badge, comment field, Keep / Send to PKM / Share / Dismiss, polling for Podcast transcription status).
   - Right pane: article metadata, per-axis values (relevance / originality / utility out of 5, noise penalty highlighted if any) and the scoring rationale, 3-line summary, extracted body, **Related Notes (up to 5)** (lazy-loaded via HTMX from `/items/{id}/related` to prevent duplicating existing Silver/Gold knowledge).
2. **Unified Search (`/search`):**
   - Query box, mode selector (Hybrid / Full-Text / Semantic), layer checkboxes (Gold / Silver / Bronze), category filter (dynamically generated from existing folders).
   - Result cards: layer & trust-tier badges, path, `<mark>`-highlighted snippet, match sources (`fts`/`vec`), modified date, generator/verifier, "Open in Obsidian (`obsidian://open?vault=<vault>&file=<path>`)".
3. **Links (`/links`):** the human decides which Silver-to-Silver candidates (03 §13) to link.
   - Filters: All (undecided) / Same category / Cross-category / Possible conflict / On hold (with counts).
   - Candidate card: similarity (or "shared tags" without embeddings), shared tags, "includes a new Silver", cross-category marker, the LLM relation label (agree / complement / possible conflict / weakly related) with its one-sentence gist, and both notes (title, category, description, Obsidian link).
   - Actions: "Link" ("Link as opposing view" for possible conflicts) / Reject (never suggested again) / Hold ("Release hold" in the On hold view) / "Compare bodies" (first 4,000 chars of both side by side). Link, reject, and hold return OOB removal plus a toast with "Undo". A gate failure shows the errors inside the card and reverts.
   - Sidebar: this week's links and rejections, number of orphan Silver notes, and Gold material candidates (groups of 3+ Silver notes connected by body links, with a copyable `gold_prompt` memo for an AI agent). Before the first graph exists, it says candidates are built after the nightly graph run.
4. **Re-review (`/review`):**
   - **Awaiting first approval:** Gold notes that have never had a `human:` approval. They fail the gate and therefore are not in `pkm.sqlite`, so the app scans `50_Knowledge/` directly and runs the gate (results cached by mtime and size). If the only error is `okf_v03.MISSING_HUMAN_VERIFICATION`, the note is approvable; otherwise the other errors are shown as "must be fixed before approval". "View content" shows the whole note as added lines.
   - **Review pending:** list of Gold notes pending re-review (expired verification, last update, days waiting, "View Diff", "Open in Obsidian").
   - Diff view (`/review/diff`): `git diff` from the commit immediately preceding the last human verification timestamp to the working tree, with an "Approve" button below.
   - Approve (`POST /review/approve`): invokes `verify_note.py` (writing `verified.at` with the local `TZ` offset); on pass returns OOB removal + toast; on failure displays error details and rolls back. The same button approves notes awaiting first approval.
   - Sidebar: manual CLI approval command and recent `lineage.jsonl` history (latest 15 entries).
5. **Dashboard (`/dashboard`):**
   - Counts by layer (Bronze / Silver / Gold / Review Pending + weekly delta, excluding sidecars).
   - 8-week bar chart of new notes by layer (weeks starting Monday in local `TZ`).
   - This week's funnel (fetched → Keep / Keep rate / Dismiss → Silver promotions → Gold syntheses, Podcast transcriptions, Hermes self-repair counts).
   - Top 10 tag rankings (HTMX partial swap for `week` / `30d` / `all`, period-over-period delta, clicking a tag jumps to `/search`).
   - Per-folder count table (dynamically generated from existing folders).
   - System health: inference server, Whisper queue, Hermes status & failure count, latest `lint_wiki` run, `pkm.sqlite` chunk / unembedded counts, failing sources list, Git uncommitted count & last commit timestamp, Syncthing device connection status (when configured).
6. **Sources (`/sources`):**
   - OPML import & export.
   - Add source form (direct add for RSS / Podcast; non-RSS sites go through `/sources/site/preview` to verify AI selectors before saving).
   - Sources table (enable/disable toggle, private flag, interval edit, delete, weekly item count & average score, failure-specific actions: `selector-mismatch` → "Re-estimate selector with AI", `http-404` → "Check URL").

### 4.3 HTTP Route Table

| Method | Path | Function |
| :--- | :--- | :--- |
| `GET` | `/` | 302 redirect to `/triage` |
| `GET` | `/healthz` | `200 ok` (for Docker HEALTHCHECK) |
| `GET` | `/triage` | Triage screen (partial list HTML when `HX-Target=item-list`) |
| `GET` | `/items/{id}` | Detail pane HTML (redirects to `/triage?item={id}` if non-HTMX) |
| `GET` | `/items/{id}/related` | Related Silver/Gold notes (max 5) partial HTML |
| `GET` | `/items/{id}/card` | Single-card OOB redraw (for Podcast status polling) |
| `POST` | `/items/{id}/comment` | Save draft primary comment |
| `POST` | `/items/{id}/keep` | Write Bronze file & OOB remove card |
| `POST` | `/items/{id}/send` | Enqueue Podcast transcription & wake worker |
| `POST` | `/items/{id}/dismiss` | Dismiss & return toast with Undo button |
| `POST` | `/items/bulk-dismiss` | Bulk-dismiss items below score threshold (default < 50) |
| `POST` | `/undo` | Undo a list of `actions` IDs |
| `POST` | `/items/{id}/restore` | Restore Dismissed/Archived item to unread |
| `GET` | `/items/{id}/share` | Share confirmation dialog HTML |
| `POST` | `/items/{id}/share` | Execute Share (`confirm=yes` required; private guard) |
| `GET` | `/status` | Header status + review-pending badge partial HTML |
| `POST` | `/crawl` | Trigger background forced crawl |
| `GET` | `/search` | Unified search screen (partial results when `HX-Target=results`) |
| `GET` | `/links` | Links screen (`?view=all\|same\|cross\|conflict\|held`) |
| `GET` | `/links/{cid}/compare` | Side-by-side bodies of a candidate pair (partial HTML) |
| `POST` | `/links/{cid}/link` | Link via `related_links.link` (with "— 対立する見方" for possible conflicts) & OOB removal |
| `POST` | `/links/{cid}/unlink` | Undo a link (toast "Undo") |
| `POST` | `/links/{cid}/reject`, `/links/{cid}/hold`, `/links/{cid}/undo` | Reject / hold / clear decision |
| `GET` | `/review` | Awaiting-first-approval + review-pending lists + lineage history |
| `GET` | `/review/diff?path=...` | `git diff` + Approve button HTML |
| `POST` | `/review/approve` | Approve via `verify_note.py` |
| `GET` | `/dashboard` | Dashboard screen (tag table partial when `HX-Target=tag-ranking`) |
| `GET` | `/sources` | Sources management screen |
| `POST` | `/sources` | Add RSS / Podcast source |
| `POST` | `/sources/site/preview` | Non-RSS AI selector estimation & test-fetch preview |
| `POST` | `/sources/site` | Register non-RSS site or update its selector |
| `POST` | `/sources/{id}/reselect` | Re-estimate selector preview for an existing non-RSS site |
| `POST` | `/sources/{id}/toggle` | Toggle source enabled/disabled |
| `POST` | `/sources/{id}/edit` | Edit source name, crawl interval, and private flag |
| `POST` | `/sources/{id}/delete` | Delete source |
| `POST` | `/sources/import` | Upload OPML |
| `GET` | `/sources/export.opml` | Download OPML |
| `GET/POST` | `/share-target` | PWA share-menu URL clip confirmation & save |
| `GET` | `/manifest.webmanifest`, `/sw.js` | PWA manifest & Service Worker |
