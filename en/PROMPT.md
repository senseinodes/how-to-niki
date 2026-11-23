# Build Prompts for AI Agents (`PROMPT.md`)

Copy the prompts in this file, replace the `{{...}}` placeholders with values for your environment, and pass them to your AI coding agent (Claude Code, Antigravity, Cursor, etc.).

Rather than asking the agent to build all phases at once, working **one Phase at a time: implement → pass tests → next Phase** produces the most reliable results.

---

## 0. Initial Kickoff (Architecture Alignment & Environment Confirmation)

Start by having the agent read the entire specification set and align on all assumptions.

```text
Please read the entire specification set under `how-to-niki/` (`README.md` and `en/specs/01` through `06`).
Based on this specification, I want to build my own autonomous PKM & triage system in the current repository.

My environment parameters are as follows:
- PROJECT_NAME: {{PROJECT_NAME}}
- USER_ID: {{USER_ID}} (human:{{USER_ID}})
- LINUX_USER: {{LINUX_USER}}
- VAULT_NAME: {{VAULT_NAME}}
- PKM_ROOT: {{PKM_ROOT}}
- REPO_DIR: {{REPO_DIR}}
- INFERENCE_HOST: {{INFERENCE_HOST}} (hostname/IP only)
- APP_HOST: {{APP_HOST}} (hostname/IP only)
- TIMEZONE: {{TIMEZONE}}
- PRIMARY_LANG: {{PRIMARY_LANG}}
- PRIVATE_CATEGORY: {{PRIVATE_CATEGORY}}
- TOPOLOGY: {{A: 2-machine setup / B: Single-machine setup}}
- CATEGORIES: {{Standard layout / custom list}}
- OUTPUT_DIRS: {{Standard layout / custom list}}
- ENABLE_SHARE: {{None / Hatena / X / Both}}

Do not write any code yet. First, present:
1. The overall directory layout tailored to these environment parameters and any technology selection differences (if any)
2. An implementation plan for Phases 1–5, plus the hardware and external setup tasks I (the human) need to perform in each Phase
3. A list of clarifying questions if anything needs confirmation
```

---

## 1. Phase 1: PKM Bundle Skeleton, OKF Schema & Deterministic Guardians

```text
Please implement Phase 1.
Reference specifications:
- `how-to-niki/en/specs/01-architecture.md` (strictly follow Do's & Don'ts)
- `how-to-niki/en/specs/02-okf-profile-and-vault.md`
- `how-to-niki/en/specs/03-guardians-and-pipeline.md` (`pkm_config.py`, `okf_v03.py`, `gate_okf.py`, `lint_wiki.py`, `verify_note.py`, `autocommit.py`)
- `how-to-niki/en/specs/06-implementation-and-tests.md` (Phase 1 DoD and test items)

Deliverables:
1. `pyproject.toml` (managed with uv, pinned versions) and `.github/workflows/test.yml`
2. Complete `pkm/` bundle template (directory skeleton, `index.md` at each level, `log.md`, `.gitignore`, `.stignore`, `AGENTS.md`, `00_System/specs/okf_local_profile.md`)
3. `pkm/00_System/schemas/okf_v03.py`
4. `pkm/00_System/scripts/`: `pkm_config.py`, `gate_okf.py`, `lint_wiki.py`, `verify_note.py`, `autocommit.py`
5. Unit test suites under `tests/schemas/` and `tests/scripts/`

Once complete, run `uv run pytest`, verify all tests pass, and report the results.
```

---

## 2. Phase 2: Inference Server Infrastructure (`llama-swap` + `llama.cpp` + `whisper.cpp`)

```text
Please implement Phase 2.
Reference specifications:
- `how-to-niki/en/specs/05-inference-and-deploy.md` (Inference Server section)
- `how-to-niki/en/specs/03-guardians-and-pipeline.md` (`gemma_client.py`)
- `how-to-niki/en/specs/06-implementation-and-tests.md` (Phase 2 DoD)

Deliverables:
1. Inference server configuration tailored to my topology ({{TOPOLOGY}}) under `deploy/inference/`: `config.yaml`, daemon unit (`plist` or `systemd`), `smoke_test.sh`, `README.md`
   - Must include automatic idle unloading (`ttl: 300`) and Parallel Groups (LLM swap, concurrent Embedding/Whisper)
2. `pkm/00_System/scripts/gemma_client.py` (chat, 768-dim Embedding prompt prefixes, `gate_okf.py` self-repair loop)
3. Configuration validation tests under `tests/deploy/` and `tests/scripts/test_gemma_client.py`

Once complete, run `uv run pytest` to verify all tests pass, and summarize the setup commands I need to run on the inference node.
```

---

## 3. Phase 3: Full-Layer Hybrid Search Builder, Search CLI, Tag Stats & Knowledge Graph

```text
Please implement Phase 3.
Reference specifications:
- `how-to-niki/en/specs/03-guardians-and-pipeline.md` (`build_index.py`, `search.py`, `tag_stats.py`, `run_graphify.py`)
- `how-to-niki/en/specs/06-implementation-and-tests.md` (Phase 3 DoD and Gotchas)

Deliverables:
1. `pkm/00_System/scripts/build_index.py` (MarkItDown sidecar extraction, SQLite FTS5 trigram + sqlite-vec incremental sync, all-level `index.md` generation, lineage recording via `gate_okf.record_lineage`, `log.md` generation)
2. `pkm/00_System/scripts/search.py` (RRF hybrid search, trust-tier weighting, LIKE fallback for terms shorter than 3 chars, review-pending listing)
3. `pkm/00_System/scripts/tag_stats.py` (period-based tag rankings with period-over-period deltas)
4. `pkm/00_System/scripts/run_graphify.py` (deterministic `graph.json` generation restricted to gate-passing Silver/Gold notes + optional Graphify CLI integration)
5. Corresponding test suite under `tests/scripts/` (must run completely offline using mocked extractors and embedders)

Once complete, run `uv run pytest` to verify all tests pass.
```

---

## 4. Phase 4: Triage Web App (`rss-triage-app`) & Autonomous Silver Promotion Worker (`hermes.py`)

```text
Please implement Phase 4.
Reference specifications:
- `how-to-niki/en/specs/01-architecture.md` (Do's & Don'ts: Vanilla CSS + HTMX + Jinja2, no Node.js builds, 3-DB separation)
- `how-to-niki/en/specs/03-guardians-and-pipeline.md` (`hermes.py`)
- `how-to-niki/en/specs/04-triage-web-app.md` (all modules, all 5 screens, PWA, Share, AI selectors, Podcast lazy transcription)
- `how-to-niki/en/specs/06-implementation-and-tests.md` (Phase 4 DoD)

Deliverables:
1. `pkm/00_System/scripts/hermes.py` and `tests/scripts/test_hermes.py`
2. Full `apps/rss_triage/` package (`Dockerfile`, `README.md`, `rss_triage/*.py`, `templates/*.html`, `static/app.css`, `static/app.js`, `static/manifest.webmanifest`, `static/sw.js`, `static/icon.svg`, `static/vendor/htmx.min.js`)
   - Do not use any CSS framework or JS build tool; implement light/dark theme support, mobile single-pane switching below 768px, and keyboard shortcuts
   - Open `pkm.sqlite` read-only and store app state in a separate `rss_triage.sqlite` file
   - Save Keep items and transcripts from "private" sources into `10_Inbox/19_private/`, and implement isolation on both the Hermes and Share sides (04 §3.6–3.8)
3. Full test suite under `tests/apps/` (core, web routes, all 5 screens, non-RSS sites, Podcast lazy transcription, Share protection, PWA)

Once complete, run `uv run pytest` to verify all tests pass.
```

---

## 5. Phase 5: Always-On Server Deployment (Docker Compose, systemd, SearXNG, Open WebUI)

```text
Please implement Phase 5.
Reference specifications:
- `how-to-niki/en/specs/05-inference-and-deploy.md` (Always-On Server, Docker Compose, systemd, sync, backup)
- `how-to-niki/en/specs/06-implementation-and-tests.md` (Phase 5 DoD)

Deliverables:
1. `docker-compose.yml` (`rss-triage-app`, `hermes-agent`, `searxng`, `open-webui`; `tools` profile separation, per-bundle project support)
2. Under `deploy/server/`:
   - `pkm.env.example`
   - `searxng/settings.yml` (JSON API enabled)
   - Complete `systemd/` unit set (**all `@` templates**: `build-index@`, `lint-wiki@`, `autocommit@`, `graphify@`, `compose@`, using `/srv/%i` and `/etc/{{PROJECT_NAME}}/%i.env`)
   - `README.md` (fixed mount, bundle expansion & `git init`, copying updated scripts to the bundle, Python/uv environment, official Docker repo, systemd enablement, Syncthing/Git operations, disabling system sleep)
3. Compose / configuration consistency tests under `tests/deploy/` (including checks that every variable in `pkm.env.example` is passed to containers via Compose and that all systemd units are templates)
4. Final update to the root `README.md`

Once complete, run `uv run pytest` to verify all tests pass, and output a deployment checklist for the actual server.
```
