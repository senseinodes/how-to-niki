# 01. System Architecture, Design Philosophy & Constraints

## 1. Core Concept & Problems Solved

### 1.1 Core Concept
Cleanly separate the **probabilistic LLM (semantic understanding, summarization, structuring, dialogue)** from **deterministic code control (schema validation, derived value calculation, index generation, audit logging)** to build a personal knowledge pipeline (LLM Wiki) that can be operated long-term without depending on paid external LLM APIs or paid cloud databases.

### 1.2 Three Problems Solved
1. **Information Overload & Rising LLM API Costs:**
   Process high-volume inputs — RSS feeds, web articles, Podcasts, PDFs, etc. — with local LLMs, bringing ongoing API cost to zero.
2. **Preventing LLM Wiki "Decay":**
   Leaving file placement, indexing, and history maintenance to LLMs causes format drift, broken links, and unverified information to accumulate. We prevent this systematically:
   - Separate values the LLM should write (semantic content) from values computed by code (layer, category ID, trust tier).
   - Route every LLM output through a Pydantic gate (`gate_okf.py`); on failure, return errors to the LLM for self-repair.
   - Generate catalogs (`index.md`), update logs (`log.md`), and audit logs (`lineage.jsonl`) exclusively by code (`build_index.py` picks up state transitions every 30 minutes and records them in `lineage.jsonl`, so nothing is missed even if a human or agent forgets to run the gate).
3. **Eliminating Memory Fabrication Risk from Automatic Chat Memory:**
   Avoid auto-memory systems derived from conversation logs (which risk conflating other people's articles with your own experiences and require heavy always-on graph databases). Instead, treat **only gate-validated OKF Markdown files** as the Single Source of Truth.

---

## 2. Strict Technical Constraints (Do's & Don'ts)

Constraints that AI agents must NOT deviate from on their own during implementation.

### Don'ts
1. **No CSS Frameworks or JS Build Tools:**
   Build the Web UI with **FastAPI + Jinja2 + HTMX + Vanilla CSS (CSS Variables / Grid / Flexbox) + minimal Vanilla JS**. Do NOT introduce Node.js, Tailwind CSS, React/Vue, Vite, Webpack, or any frontend build pipeline. Vendor HTMX locally (`static/vendor/htmx.min.js`) with no CDN dependency.
2. **No Heavy External Databases or Separate Memory Systems:**
   Do NOT spin up external DB containers such as PostgreSQL, MySQL, Neo4j, Qdrant, Chroma, or Elasticsearch. Consolidate all persistence and search into **SQLite 3 (`FTS5` + `sqlite-vec`)**.
3. **Never Merge the 3 SQLite Files:**
   Because their purposes and lifecycles differ, always keep them as separate files inside `00_System/`:
   - `pkm.sqlite`: Search index, tag stats, and lint results (can be deleted and recreated from scratch via `build_index.py --rebuild`)
   - `rss_triage.sqlite`: Web app state (sources, items, action history, crawl runs — survives a `pkm.sqlite` rebuild)
   - `hermes.sqlite`: Processing history, retry state, and self-repair counts of the autonomous Silver worker
4. **Never Allow OKF Frontmatter in the Bronze Layer:**
   Files under `10_Inbox/` must never contain a `type` key.
5. **Never Let LLMs Write Derived Values, and Never Move Silver Notes:**
   - Only the gate writes `x-pkm-medal`. Never write `jd_id` or `okf_version` into notes (`okf_version` belongs exclusively in the root `index.md`).
   - Never move a Silver note into `50_Knowledge/`. A Gold note is synthesized as a **new note** that cites Silver notes in `sources`.
6. **No Whitespace, `&`, etc. in File or Folder Names:**
   To avoid breaking CommonMark links, shell commands, and Docker mounts, disallow characters matching `[\s&#%?<>\"|*]`.
7. **No Permanent Memory Hoarding on Shared Inference Hosts:**
   Configure automatic idle unloading (`ttl: 300`) on the inference server so memory is freed whenever idle.

### Do's
1. **Separation of Probabilistic & Deterministic Logic + Self-Repair Loop:**
   Every Silver / Gold note generated or updated by an LLM must pass `gate_okf.py`; on failure, feed the JSON error details straight back to the LLM to regenerate (default max 3 attempts).
2. **Raw YAML Line Editing to Preserve Unknown Keys & Key Order:**
   Per OKF v0.3's unknown-key preservation rule, whenever a script modifies frontmatter (`x-pkm-medal` in `gate_okf.py --fix`, appending `verified` in `verify_note.py`), do NOT re-dump a Pydantic model — perform **raw YAML line editing** so all other keys and their ordering remain byte-for-byte intact.
3. **Protecting Human Authorship:**
   `gate_okf.py` compares against `git show HEAD:<path>` and rejects any non-`human:` change that modifies or deletes `x-pkm-my-comment`, or alters, deletes, or reorders existing `verified` events.
4. **Lazy Transcription for Podcasts:**
   Never download audio files during RSS crawling. Perform first-pass triage using only `<title>` and `<description>` (show notes), and fetch `<podcast:transcript>` or run `whisper.cpp` transcription ONLY for episodes where the human clicks `[Send to PKM]`.
5. **Multi-Format Sidecar Extraction:**
   For PDFs, Office documents, and images under `10_Inbox/`, generate `<filename>.extracted.md` using `MarkItDown` (with local VLM integration) and track changes by SHA256 without modifying the original binary.
6. **Isolation of Private Compartments (Bronze `10_Inbox/19_private/` & Silver `20_Literature/28_me/`):**
   - Kept articles and Podcast transcripts from sources flagged "private" in the web app are saved to `10_Inbox/19_private/` instead of `11_rss/` or `14_podcast/`.
   - Both folders are excluded via `.gitignore` (never pushed to GitHub, while still synced via Syncthing), and `gate_okf.py` / `build_index.py` check `git check-ignore` so they are never logged to `lineage.jsonl` or `log.md`.
   - `hermes.py` bypasses LLM placement planning for Bronze items from `19_private/`, locks their target category to `28_me`, and never sends query terms to SearXNG (external search engine).
   - The web app blocks Share on both `sources.private == 1` and path prefixes `10_Inbox/19_private/` / `20_Literature/28_me/`.

---

## 3. Overall System Architecture

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ Client Devices (PC / Smartphone / Obsidian / AI Coding Agents)              │
│  • Claude Code / Antigravity / etc. (Gold synthesis, writing, OKF Skills)   │
│  • Obsidian + Web Clipper (Markdown reading, manual editing, clipping)      │
│  • Browser / PWA (rss-triage-app & Open WebUI, mobile share-menu clipping)  │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │ LAN / Tailscale VPN + Syncthing
┌────────────────────────────────────▼────────────────────────────────────────┐
│ Always-On Server (Linux 24/7 / Docker Compose + systemd timers)             │
│  • Docker Compose:                                                          │
│      - rss-triage-app : FastAPI + HTMX + Vanilla CSS + rss_triage.sqlite    │
│      - hermes-agent   : Bronze watch → SearXNG research → Silver + repair   │
│      - searxng        : Metasearch engine (JSON API enabled)                │
│      - open-webui     : Weekly review & long-form dialogue UI               │
│  • systemd timers (host /opt/<project>/.venv):                              │
│      - build_index.py (every 30m) : sidecars, pkm.sqlite, lineage, index/log│
│      - autocommit.py  (02:30)     : gated auto git commit & push            │
│      - lint_wiki.py   (03:30)     : broken links, orphans, SHOULD → DB      │
│      - run_graphify.py(04:00)     : Silver/Gold-only knowledge graph        │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │ HTTP :8080 (OpenAI-compatible / On-Demand)
┌────────────────────────────────────▼────────────────────────────────────────┐
│ Inference Server (GPU / Apple Silicon / On-Demand)                          │
│  • llama-swap (:8080, ttl: 300, Parallel Groups):                           │
│      - llm group (swap):      gemma4-review (large) / gemma4-fast (med+VLM) │
│      - embed group (parallel): gemma-embed (768-dim Embedding)              │
│      - audio group (parallel): whisper (whisper.cpp /v1/audio/transcriptions)│
└─────────────────────────────────────────────────────────────────────────────┘
```

### Critical Coupling Point (Single Script Location)
- Deterministic scripts (`00_System/scripts/`) and schemas (`00_System/schemas/`) live **inside the PKM bundle (`{{PKM_ROOT}}/00_System/`)**.
- Both host systemd timers and Docker containers (`rss-triage-app` / `hermes-agent`) mount the bundle (`/pkm`) and execute **the exact same script code**, preventing spec drift between host and containers.
- Note that the "canonical source" of scripts is `pkm/00_System/` in the dev repository (template & test target), while what runs in production is the copy in the bundle. After updating scripts in the dev repo, copy them to `{{PKM_ROOT}}/00_System/` and commit on the bundle side as well (see 05 §2.1).

---

## 4. Hardware Topology Patterns (Choose for Your Environment)

The original setup uses a 2-machine topology — an always-on low-power Linux box (Intel N100, 32GB) + an inference node (M4 Mac mini, 32GB) — but you can pick from the following options based on your hardware:

| Pattern | Setup | Changes & Notes |
| :--- | :--- | :--- |
| **A: 2-Machine Setup (Recommended / Original)** | Always-on Linux (low-power PC / mini PC / Raspberry Pi, etc.) + GPU machine (Apple Silicon Mac or NVIDIA GPU Linux/Windows) | Keeps 24/7 electricity low while connecting to the GPU node over HTTP only during inference. Designed so `build_index.py` (FTS-only) and Web UI browsing keep working even when the GPU machine is asleep. |
| **B: Single-Machine Setup** | One always-on Linux or Mac with a GPU and sufficient RAM (32GB+ recommended) | Runs `llama-swap` and Docker on the same machine. Set `MAC_MINI_URL` to the **host's LAN or Tailscale IP** (e.g., `http://192.168.1.10:8080`). Because the env file is shared between systemd (host) and Compose (containers), do NOT use `localhost` (points to the container itself) or `host.docker.internal` (unresolvable from a Linux host and requires `extra_hosts: host-gateway` in containers). |
| **C: Hybrid Cloud API Setup (Reference / Unsupported)** | Always-on Linux (or VPS) + OpenAI-compatible cloud API | Outside specification scope because it departs from the "zero API cost" premise. Changes if you try it: set `PKM_API_KEY` (shared by `gemma_client`, MarkItDown, whisper, and Open WebUI) and `PKM_MODEL_*`; adapt the EmbeddingGemma-specific prefixes (03 §3.1) and 768 dimensions (`pkm_config.EMBED_DIM`) to your embedding model; provide alternatives for llama-swap's `/running` endpoint (header inference indicator) and whisper's `/v1/audio/transcriptions`. |

### Model Sizing Guide (by Memory)
You can tune `llama-swap` models to fit your inference node's VRAM / unified memory (model IDs on the app side are abstracted behind environment variables).

| Unified Memory / VRAM | `gemma4-review` (deep dialogue) | `gemma4-fast` (fast + VLM) | `gemma-embed` | `whisper` |
| :--- | :--- | :--- | :--- | :--- |
| **32GB+ (Standard)** | Gemma 4 26B-A4B Q4_0 + mmproj (64K ctx) | Gemma 4 12B Q4_0 + mmproj (32K ctx) | EmbeddingGemma-300M Q8_0 (768d) | large-v3-turbo |
| **16GB** | Gemma 4 12B Q4_0 + mmproj (32K ctx) | Gemma 4 4B–12B Q4_0 + mmproj (16K ctx) | EmbeddingGemma-300M Q8_0 (768d) | medium / large-v3-turbo |

---

## 5. Network, Sync & Backup Strategy

### 5.1 Network & Ports
Never expose services to the public internet; access them only within your home LAN or a private VPN such as **Tailscale**.

| Source | Destination | Protocol / Port | Purpose |
| :--- | :--- | :--- | :--- |
| Client / Smartphone | Always-On Server | `HTTP :8000` (increment per bundle) | `rss-triage-app` Web UI / PWA |
| Client | Always-On Server | `HTTP :3000` | `Open WebUI` (weekly review & dialogue) |
| Server (App / Hermes / Scripts) | Inference Server | `HTTP :8080` | `llama-swap` (Chat / Embed / Whisper) |
| Server (Hermes / Open WebUI) | Docker internal `http://searxng:8080` (`:8888` from host) | `HTTP :8080` (inter-container) / `:8888` (published) | `SearXNG` metasearch JSON API |
| Server / Client / NAS | Mesh P2P | `TCP/UDP :22000` | `Syncthing` bidirectional file sync |

### 5.2 Dual Sync (Syncthing × Git) Division of Responsibilities
**Clearly separating file sync (Syncthing) from version control (Git)** prevents conflicts.

- **Syncthing (Real-Time P2P Sync):**
  - Scope: Always-on server ⇄ Workstation PC / Mobile ⇄ Backup NAS (no sync needed on a dedicated inference node).
  - Exclusions (`.stignore`): `.git`, `*.sqlite*`, `00_System/graph`, `.obsidian/workspace*.json`, `__pycache__`, `.claude/worktrees`.
  - **Important:** Never sync `.git` via Syncthing. Concurrent commits from multiple devices fork and corrupt history, so **centralize Git management and auto-commits on the single always-on server**.
- **Git / GitHub Private Repository (History Preservation & Gate Baseline):**
  - Scope: OKF Markdown, scripts, configs, `lineage.jsonl`, `index.md`, `log.md`.
  - Exclusions (`.gitignore`): `90_Assets/*`, private compartments (`20_Literature/28_me/`, `10_Inbox/19_private/`), `00_System/graph/*`, `*.extracted.md`, original binaries in `10_Inbox/15_files/`, `*.sqlite*`, `00_System/logs/rejected/`, `lost+found/`, `.stfolder/`.
  - **Important:** Because `gate_okf.py` Rule 7 (protecting human authorship) diffs against `git show HEAD:<path>`, `autocommit.py` runs nightly to automatically commit and push only gate-passing files.

### 5.3 Multi-Bundle Extensibility
If you want separate PKM bundles (e.g., work vs. personal), you can run multiple instances off the same repository code by separating only the following:
- Data path: `/srv/<vault_name>` (fixed layout because all systemd units are `@` templates referencing `/srv/%i`)
- Env file: `/etc/<project>/<vault_name>.env` (change `PKM_ROOT`, `NIKI_VAULT`, `NIKI_APP_PORT`)
- Compose project: `docker compose -p <project>-<vault_name>`
- systemd: enable the 5 units per bundle, e.g., `<project>-build-index@<vault_name>.timer`
- Enable `COMPOSE_PROFILES=tools` (SearXNG and Open WebUI) on only ONE bundle across the entire server.
