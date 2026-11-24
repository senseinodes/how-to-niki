# 06. Phased Implementation Roadmap, Test Specifications & Gotchas

---

## 1. Recommended Implementation Roadmap (Phase 1 – Phase 5)

Implement and verify across 5 phases ordered from fewest external dependencies and easiest offline unit testing upward.

### Phase 1: PKM Bundle Skeleton, OKF Schema & Deterministic Guardians
- **Target files:**
  - `pyproject.toml`, `.python-version`, `.github/workflows/test.yml`
  - Complete `pkm/` template (directories, `index.md`, `log.md`, `.gitignore`, `.stignore`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `00_System/specs/okf_local_profile.md`)
  - `pkm/00_System/schemas/okf_v03.py`
  - `pkm/00_System/scripts/{pkm_config,gate_okf,lint_wiki,verify_note,autocommit}.py`
- **Definition of Done (DoD):**
  - Valid Silver / Gold notes pass `gate_okf.py`, and `--fix` writes `x-pkm-medal` without disturbing unknown keys or key ordering.
  - Accurately detects all of the following errors: `type` in Bronze, out-of-range category IDs, forbidden filename characters, legacy keys, invalid/duplicate/reserved tags, invalid actors, broken/out-of-bundle links, direct or unapproved Gold placement, agent modification of `x-pkm-my-comment`, and tampering with `verified`.
  - Expired Gold verification yields `review_pending` (not an error), and approving via `verify_note.py` restores `review_cleared`. Even if `verify_note.py` runs before `review_pending` was logged, both `review_pending` → `review_cleared` are recorded in order.
  - First gate pass of Silver / Gold records `promoted` exactly once (for Gold: `medal: "gold"` and `at` = latest `human:` verification timestamp). Writes to `lineage.jsonl` re-read state under `flock`.
  - `autocommit.py` unstages only gate-failing files and commits the rest.

### Phase 2: Inference Server Infrastructure & LLM Client
- **Target files:**
  - `deploy/inference/` (`config.yaml`, `launchd plist` or `systemd unit`, `smoke_test.sh`, `README.md`)
  - `pkm/00_System/scripts/gemma_client.py`
- **Definition of Done (DoD):**
  - Configuration validation tests pass (YAML syntax, `ttl: 300`, Parallel Groups, model ID consistency).
  - `gemma_client.py` self-repair loop tests pass (attempt 1 fails gate → error JSON returned → attempt 2 passes with `--fix` & lineage recorded; all attempts failing restores original file & saves draft to `logs/rejected/`).
  - On real hardware, `smoke_test.sh --wait-unload` passes, confirming text, vision, 768-dim embedding, concurrent LLM+Embed execution, and unload after 5 minutes.

### Phase 3: Full-Layer Hybrid Search Builder, Search, Tag Stats & Knowledge Graph
- **Target files:**
  - `pkm/00_System/scripts/{build_index,search,tag_stats,run_graphify}.py`
- **Definition of Done (DoD):**
  - Sidecar generation for mock binaries (PDF/Office/images), SHA256 skip, orphan sidecar cleanup on source deletion, and Office `dcterms:modified` priority all work.
  - Incremental `FTS5 (trigram)` + `sqlite-vec (768d)` indexing into `pkm.sqlite`, `LIKE` fallback for terms < 3 chars, and RRF + trust-tier weighting (`Gold > Silver = review-pending Gold > Bronze`) all work.
  - Even when the inference server is offline, `build_index.py` completes FTS indexing and `search.py` falls back to FTS results.
  - `index.md` (preserving root frontmatter) and `log.md` (descending by date) are deterministically written only when changed.
  - `build_index.py` records lineage for gated Silver / Gold notes before generating `log.md` (direct Gold edits by agents produce `review_pending` on the next run; `--no-lineage` disables).
  - `run_graphify.py` builds `graph.json` and `manifest.json` strictly from passing Silver/Gold notes and skips when inputs are unchanged.

### Phase 4: Triage Web App (`rss-triage-app`) & Autonomous Worker (`hermes.py`)
- **Target files:**
  - `pkm/00_System/scripts/hermes.py`, `pkm/00_System/scripts/related_links.py`
  - `apps/rss_triage/` (`Dockerfile`, `README.md`, `rss_triage/*.py`, `templates/*.html`, `static/*`)
- **Definition of Done (DoD):**
  - `hermes.py` selects settled (> 10 min) unprocessed Bronze notes, runs plan → SearXNG research → Silver generation + self-repair → `hermes.sqlite` logging, and respects already-cited Bronze skips and `x-pkm-my-comment` protection.
  - `related_links.py` builds candidates from graph.json and embeddings (shared tags as fallback), excludes already-linked, decided, and private/public pairs, and caches relation labels (no relabeling of unchanged notes, nightly cap, candidates still built when the inference host is offline). `link` appends mutual links to both `## 関連` sections without changing `generated`, runs the gate, and reverts both on failure. `unlink` removes only the appended lines.
  - All 6 screens and all endpoints of `rss-triage-app` work, verified with injected mocks (`chat`, `embedder`, `http`, `clock` on `State`) for:
    - RSS/Atom crawling, conditional GET, full-text extraction, per-axis scoring (interest-tag injection, combination, LLM-free recompute on weight change, breaking flag and filter), 7-day auto-archive
    - Links screen (filters, link / link as opposing view / reject / hold / undo, side-by-side bodies, Gold material memo)
    - Listing Gold awaiting first approval (approvable vs. needs fixes) and approving it
    - Keep (non-OKF Bronze write without `type`), Dismiss, Undo, bulk Dismiss, Restore
    - Non-RSS AI selector estimation, preview, and re-estimation
    - Podcast lazy transcription (no audio DL during crawl, `<podcast:transcript>` priority, `whisper` invocation & temp audio cleanup when absent)
    - Unified search, related notes, re-review diff & approval, dashboard aggregation
    - Share confirmation guard and blocking for private sources / `19_private` / `28_me`
    - Private source Keeps & transcripts landing in `10_Inbox/19_private/`, with Hermes locking them to `28_me` and never calling SearXNG
    - `TZ`-aware date formatting (`zoneinfo`; unknown TZ falls back to UTC)
    - PWA manifest, Service Worker, `/share-target` clipping
  - `docker build` passes the `sqlite-vec` load verification step.

### Phase 5: Always-On Server Deployment & Operational Polish
- **Target files:**
  - `docker-compose.yml`
  - `deploy/server/` (`pkm.env.example`, `searxng/settings.yml`, `systemd/*`, `README.md`)
  - Root `README.md`
- **Definition of Done (DoD):**
  - Consistency tests for `docker-compose.yml`, systemd units, and `searxng/settings.yml` pass (every variable in `pkm.env.example` referenced in Compose, app/LLM config variables passed to containers, Open WebUI honors `PKM_MODEL_REVIEW` / `PKM_API_KEY`, all systemd units are `@` templates using `/srv/%i` and `%i.env`).
  - `uv run pytest` passes 100%, including in CI (GitHub Actions).

---

## 2. Test Design Policy & Mandatory Test Suites

All unit and integration tests must run **fast and completely offline without depending on the network, a live inference server, or GitHub** (use dependency injection to swap `chat`, `embedder`, `extractor`, `http`, `runner`, and `clock`).

| Directory | Example Test Files | Key Verification Areas |
| :--- | :--- | :--- |
| `tests/schemas/` | `test_okf_v03.py` | MUST/SHOULD/optional keys, legacy key rejection, timestamp normalization, order-independent trust tier, path resolution (root-relative / file-relative / external rejection / percent-decoding), protected-field diffing |
| `tests/scripts/` | `test_pkm_config.py`, `test_gate_okf.py`, `test_lint_wiki.py`, `test_verify_note.py`, `test_autocommit.py`, `test_gemma_client.py`, `test_build_index.py`, `test_tag_stats.py`, `test_hermes.py`, `test_run_graphify.py`, `test_related_links.py` | Link candidate generation, exclusion rules, rejecting private pairs, relation-label caching and cap, appending/undoing `## 関連` with rollback on gate failure, `linked`/`unlinked` lineage, Full coverage of Rules 1–7, raw YAML editing preserving unknown keys & order, no lineage for git-ignored files, Gold `promoted` / backfilling unrecorded `review_pending` / `flock` mutual exclusion, lineage recording in `build_index`, `PKM_API_KEY` propagation, Hermes private Bronze handling & extracting the final section of `AGENTS.md`, `--save-db`, self-repair retry & rollback, sidecar diffs & Office dates, FTS5 + vec + `<3`-char LIKE, RRF weighting, Hermes skipping without recording on connection failure, Bronze exclusion in graph |
| `tests/apps/` | `test_rss_triage_core.py`, `test_rss_triage_web.py`, `test_rss_triage_pages.py`, `test_rss_triage_sites.py`, `test_rss_triage_podcast.py` | Crawl / per-axis scoring, combination & rescoring / Keep / Undo / Archive, FastAPI `TestClient` across all 6 screens (including Links and awaiting first approval), HTMX responses & OOB removals, AI selectors, Podcast lazy transcription & temp file cleanup, Share OAuth signing & private guards, PWA `share_target` |
| `tests/deploy/` | `test_inference_config.py`, `test_server_compose.py` | `llama-swap` config `ttl: 300`, Parallel Groups, all model IDs; `docker-compose.yml` volumes, env vars (cross-checked with `pkm.env.example`), `PKM_ENV_FILE=/nonexistent`, SearXNG JSON format enabled; all systemd units templated & nightly job order |

---

## 3. Implementation & Operational Gotchas

Real-world pitfalls uncovered during the original build and live operation. Guard against every one of these proactively during implementation.

1. **Never put `pkm.sqlite` and `rss_triage.sqlite` in the same file:**
   `build_index.py --rebuild` deletes `pkm.sqlite` (plus `-wal`/`-shm`) and recreates it from scratch. Putting app state in the same DB would wipe all subscribed sources and Keep/Dismiss history.
2. **Verify `sqlite-vec` extension loading:**
   Whether Python's standard `sqlite3` supports `conn.enable_load_extension(True); sqlite_vec.load(conn)` depends on OS and Python build flags. Smoke-test it in both the Dockerfile build `RUN` step and the host setup instructions.
3. **Japanese / CJK full-text search & FTS5 trigram's 3-character minimum:**
   SQLite's `unicode61` cannot tokenize Japanese/CJK text, so we use `tokenize='trigram'`. However, trigram **never matches 1–2 character terms (e.g., `AI`, `設計`)**, so routing `len(term) < 3` terms to `LIKE` clauses is mandatory.
4. **Empty responses caused by reasoning tokens (`reasoning_content`):**
   Thinking-enabled models like Gemma 4 can burn through a small `max_tokens` budget on internal reasoning, returning an empty `choices[0].message.content`.
   - Disable thinking on high-volume fast models (`gemma4-fast`) with `--reasoning-budget 0`.
   - For thinking-enabled models (`gemma4-review`), set `max_tokens` sufficiently large (2048+) or leave it unset.
5. **`whisper-server` returning HTTP 500 under a daemon manager:**
   Under `launchd` or `systemd`, the working directory may default to `/`. Always pass `--tmp-dir /tmp` when using `--convert`.
6. **`EmbeddingGemma` prompt prefixes:**
   Embedding raw text degrades retrieval quality severely. Always prepend `title: <title> | text: <text>` for documents and `task: search result | query: <query>` for queries.
7. **YAML automatic timestamp coercion:**
   PyYAML parses unquoted ISO timestamps into `datetime.datetime` objects. Validators must accept both `str` and `datetime`/`date`, while LLM prompts should ask for quoted RFC 3339 strings to reduce diff noise.
8. **`pkm_config.py` `.env` discovery inside containers:**
   If `PKM_ENV_FILE` is left unset inside a Docker container, `pkm_config.py` may look for host `.env` files. Explicitly disable this in Compose `environment:` with `PKM_ENV_FILE: /nonexistent`.
9. **Running `git` inside containers & `safe.directory`:**
   Running `git diff` or `git status` on a host-owned `/srv/pkm` bind mount inside a container often triggers dubious ownership errors on UID mismatch. Match the container `user:` to the host owner AND pass `git -c safe.directory=<root>`.
10. **Never use `verified[-1]` for trust tier evaluation:**
    If `process:lint-wiki` appends an event after human approval, the last element becomes `process:`, falsely marking the note expired. Always check whether ANY event with `at >= generated.at` is a `human:` verification.
11. **Never bump `generated.at` on formatting or link fixes:**
    Advancing `generated.at` for non-semantic edits unintentionally drops approved Gold notes into "review pending".
12. **Defense-in-depth for private compartments (`19_private` / `28_me`):**
    Adding them to `.gitignore` alone is not enough. Implement all four layers together:
    - Route Keeps from private sources to Git-ignored `10_Inbox/19_private/` instead of Git-tracked `11_rss/` (ignoring only Silver still leaks Bronze to GitHub).
    - Wire `git check-ignore` into lineage logging so private metadata never leaks into Git-committed `lineage.jsonl` or `log.md`.
    - Prevent Hermes from sending terms from private Bronze notes to SearXNG (external search engine).
    - Block Share in the Web UI using both the source's `private` flag AND path prefixes (`19_private/` and `28_me/`). Note that checking only `20_Literature/28_me/` would never trigger on Bronze Keeps in `10_Inbox/`.
13. **Never sync `.git` with Syncthing:**
    Syncing `.git` across devices creates conflict files and corrupts the repository. Always exclude `.git` and `*.sqlite*` in `.stignore` and centralize Git operations on the single always-on server.
14. **Compose `--env-file` does not inject variables into containers:**
    `docker compose --env-file X` only expands `${VAR}` references inside the Compose file. Variables present in the env file but omitted from `environment:` are silently ignored (breaking `NIKI_ARCHIVE_DAYS` or `PKM_MODEL_*`). List every needed variable with defaults and guard against omissions with a cross-check test against `pkm.env.example`.
15. **Hardcoded language touchpoints (when changing `{{PRIMARY_LANG}}` from `ja`):**
    The original implementation hardcodes `ja`. When changing it, update all of these together: `whisper-server` `--language` in `llama-swap`, `language` in `podcast.py`, SearXNG `language` in `hermes.py`, `default_lang` / `default_locale` in `searxng/settings.yml`, LLM prompts (`scoring.py`, `hermes.py`, `gemma_client.py`), and the default/allowed values of `x-pkm-lang` in `okf_v03.py`.
16. **Preventing missed lineage records:**
    If lineage is written only on manual gate runs, an agent editing a Gold note without running the gate leaves no `review_pending`, which in turn prevents `review_cleared` from being logged upon approval. Recording state transitions periodically in `build_index.py` and running the gate pre-approval in `verify_note.py` closes the gap. Because multiple processes (Hermes container + host timer) write concurrently, `flock` plus re-reading state after locking is mandatory.
17. **Inference URL in Pattern B (Single-Machine Setup):**
    The env file is shared between systemd (host) and Compose (containers). `localhost` refers to the container itself inside Docker, and `host.docker.internal` does not resolve on a Linux host. Use the host's LAN or Tailscale IP.
18. **Never Let the LLM Assign a 0–100 Score Directly:**
    Scores cluster around 65–85 and the 50+ / 80+ thresholds stop working. Have it return discrete per-axis values (with rubrics), combine them in code, and make it write the summary and rationale before scoring. Keep the axis values so weights can be recomputed later.
19. **Do Not Learn Preferences from Keep / Dismiss:**
    Dismiss also means "already known", "duplicate", or "not now", so using it as few-shot negatives causes mislearning. Base interest on the tags of notes actually kept in the knowledge base (last 30 days).
20. **Gold Awaiting First Approval Is Not in the Search DB:**
    Gold without a `human:` approval fails the gate, so `build_index.py` and `autocommit.py` skip it and it shows up nowhere. The web app runs `50_Knowledge/` through the gate directly to list it, matching on the error constant (`MISSING_HUMAN_VERIFICATION`) rather than duplicating the string.
21. **Appending Links Must Not Change `generated`:**
    Adding a link is a meaning-preserving change like a link repair; bumping `generated.at` would make a Silver note's "last meaningful change" newer even though its content did not change. Pairing a private note with a public one would leave a private path in the public note, so exclude such pairs at the candidate stage.
