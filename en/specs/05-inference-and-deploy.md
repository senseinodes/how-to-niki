# 05. Inference Server, Always-On Server, Deployment & Environment Variables

---

## 1. Inference Server (`llama-swap` + `llama.cpp` + `whisper.cpp`)

To maximize GPU memory bandwidth, run the inference stack **natively on the host** rather than in Docker (Homebrew + `launchd` on Apple Silicon Mac; build/packages + `systemd` on Linux GPU machines).

### 1.1 Exposed Model IDs & Group Design (`config.yaml`)

| Model ID | Backing Model | Context | Group | Concurrency Policy |
| :--- | :--- | :--- | :--- | :--- |
| `gemma4-review` | Gemma 4 26B-A4B Q4_0 + mmproj Q8_0 | 65536 (KV `q8_0`) | `llm` | `swap: true` (swaps with `gemma4-fast` inside group), `exclusive: false` (does not kill Embed / Whisper in other groups) |
| `gemma4-fast` | Gemma 4 12B Q4_0 + mmproj Q8_0 | 32768 | `llm` | **Disable thinking via `--reasoning-budget 0`** (faster short judgments/OCR/structuring & prevents empty responses) |
| `gemma-embed` | EmbeddingGemma-300M QAT Q8_0 | 2048 (`--embedding --pooling mean`) | `embed` | `swap: false`, `exclusive: false`, `persistent: true` (runs concurrently with LLM) |
| `whisper` | `whisper-server` (`ggml-large-v3-turbo.bin`) | — | `audio` | `swap: false`, `exclusive: false`, `persistent: true` (runs concurrently with LLM/Embed) |

### 1.2 Mandatory `llama-swap` Configuration Requirements
- **Ports:** Expose ONLY `llama-swap` on `0.0.0.0:8080`. Each backend server binds `127.0.0.1:${PORT}` (`startPort: 10001`) with explicit `proxy: http://127.0.0.1:${PORT}` (avoids `localhost` resolving to IPv6 `::1`).
- **Automatic Idle Unload:** `globalTTL: 300` and `ttl: 300` on every model (terminates processes after 5 minutes of inactivity to return RAM/VRAM; `persistent: true` only prevents eviction by other groups while still honoring `ttl`).
- **Cold-Load Timeout:** `healthCheckTimeout: 300`.
- **Common Chat Flags:** `--n-gpu-layers 99 --no-webui --jinja --flash-attn on --temp 1.0 --top-k 64 --top-p 0.95 --parallel 1`.
- **Mandatory `whisper-server` Flags:**
  ```text
  --host 127.0.0.1 --port ${PORT}
  --model ${models}/ggml-large-v3-turbo.bin
  --request-path /v1/audio/transcriptions --inference-path ""
  --language ja
  --convert
  --tmp-dir /tmp
  ```
  - Set `--language` to `{{PRIMARY_LANG}}` (`ja` in the original). Do NOT put `#` comments inside `cmd` because they get passed as CLI arguments.
  - Set `checkEndpoint: /v1/audio/transcriptions/` (`whisper-server` has no `/health` endpoint).
  - `--convert` requires `ffmpeg` to be installed (converts mp3/m4a → 16kHz wav).
  - **`--tmp-dir /tmp` is mandatory:** When launched by `launchd` or `systemd`, the working directory may be `/`, causing HTTP 500 errors when creating temporary files.
- **macOS-Specific Note:** If large models fail to load on a 32GB Mac, raise the GPU wired memory limit with `sudo sysctl iogpu.wired_limit_mb=24576`.
- **Daemonization:**
  - macOS: `LaunchAgents` (`com.{{PROJECT_NAME}}.llama-swap.plist`, `RunAtLoad=true`, `KeepAlive=true`, include `/opt/homebrew/bin` in `PATH`).
  - Linux: `systemd` unit with equivalent arguments and environment variables.
- **Smoke Test (`smoke_test.sh`):**
  1. `GET /v1/models`
  2. `gemma4-fast` text completion (verify non-empty `content`)
  3. `gemma4-fast` image input (base64 `image_url`)
  4. `gemma-embed` returns 768 dimensions AND `/running` shows both `gemma4-fast` and `gemma-embed` simultaneously (verifying Parallel Groups)
  5. `gemma4-review` text completion (`max_tokens: 2048`+ so reasoning + answer fit, and it swaps out `gemma4-fast`)
  6. With `--wait-unload`: wait 330 seconds and verify `/running` is empty

---

## 2. Always-On Server (Docker Compose & systemd)

### 2.1 Directory Layout & Prerequisites
| Path | Contents |
| :--- | :--- |
| `{{PKM_ROOT}}` (e.g., `/srv/pkm`) | Live PKM bundle (if on dedicated storage, mount via `/etc/fstab` using `UUID=...` + `noatime,nofail`; never rely on desktop `/media/` automounts) |
| `{{REPO_DIR}}` (e.g., `/opt/{{PROJECT_NAME}}`) | Dev repository + host Python virtualenv (`.venv` via `uv sync --locked --no-dev`) |
| `/etc/{{PROJECT_NAME}}/{{VAULT_NAME}}.env` | Per-bundle env file (shared by systemd `EnvironmentFile` and Docker Compose `--env-file`) |

- **Keep `{{PKM_ROOT}}` at `/srv/{{VAULT_NAME}}`.** All systemd units are `@` templates referencing `/srv/%i`.
- **Bundle Deployment & Git Initialization (separate private repo from the dev repo):**
  ```bash
  git clone <dev-repo> /tmp/{{PROJECT_NAME}}-template
  cp -a /tmp/{{PROJECT_NAME}}-template/pkm/. {{PKM_ROOT}}/
  cd {{PKM_ROOT}} && git init -b main && git add -A && git commit -m "init bundle"
  # Optional: create a private GitHub repo, then git remote add origin ... && git push -u origin main
  ```
  Gate Rule 7 (protecting human authorship) and `autocommit.py` assume the bundle is a Git repository (without Git, protection silently stops working).
- **Propagating Script Updates:** After modifying `pkm/00_System/{scripts,schemas,specs}/` in the dev repo, copy them into `{{PKM_ROOT}}/00_System/` and commit on the bundle side. Both containers and host timers execute the bundle's copy.
- **Propagating Agent Instruction Files:** Likewise, after updating `pkm/AGENTS.md`, `CLAUDE.md`, or `GEMINI.md`, copy them to the bundle root and commit (if you have extended `AGENTS.md` on the bundle side, `diff` first). Verify by launching `claude` in `{{PKM_ROOT}}` and checking that `/memory` lists `CLAUDE.md` and `AGENTS.md`. Launch AI agents at the bundle root (over SSH too — `search.py` and `graph.json` exist only on the server).
- **Host Python Environment:** Run `uv sync --locked --no-dev` in `{{REPO_DIR}}` and verify `sqlite-vec` loads cleanly (06 §3 Gotcha 2).
- Install Docker from the **official Docker apt repository** (`docker-ce` + `docker-compose-plugin`); avoid the snap package due to `/srv` mount restrictions and auto-refresh restarts.
- Disable OS auto-suspend for 24/7 operation (`systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target`).

### 2.2 `docker-compose.yml` Layout (4 Services)

Project name: `name: {{PROJECT_NAME}}-${NIKI_VAULT:-pkm}`

| Service | Image / Build | Port | Profile | Role |
| :--- | :--- | :--- | :--- | :--- |
| `rss-triage-app` | `apps/rss_triage/Dockerfile` (`python:3.13-slim` + `git` + `tzdata` + `uv`) | `${NIKI_APP_PORT:-8000}:8000` | Always | Triage, search, re-review, and dashboard Web UI. `user: "${NIKI_UID}:${NIKI_GID}"`, mount `${PKM_ROOT}:/pkm`, `PKM_ROOT=/pkm`, `HOME=/tmp`, `PKM_ENV_FILE=/nonexistent`. Verify `sqlite-vec` load at build time + `/healthz` HEALTHCHECK. |
| `hermes-agent` | Same image as `rss-triage-app` | None | Always | `command: ["python", "/pkm/00_System/scripts/hermes.py", "--watch"]`, `healthcheck: { disable: true }`. Uses `SEARXNG_URL=${HERMES_SEARXNG_URL:-http://searxng:8080}`. |
| `searxng` | `${SEARXNG_IMAGE:-searxng/searxng:<pinned-tag>}` | `${NIKI_SEARXNG_PORT:-8888}:8080` | `tools` | Metasearch. Mount `deploy/server/searxng/settings.yml` `:ro` (`search.formats: [html, json]`, `server.limiter: false`, `server.method: GET`). Requires `SEARXNG_SECRET`. |
| `open-webui` | `${OPEN_WEBUI_IMAGE:-ghcr.io/open-webui/open-webui:<pinned-tag>}` | `${NIKI_WEBUI_PORT:-3000}:8080` | `tools` | Weekly review & long-form dialogue UI. `OPENAI_API_BASE_URL=${MAC_MINI_URL}/v1`, `OPENAI_API_KEY=${PKM_API_KEY:-local}`, `ENABLE_OLLAMA_API=false`, `DEFAULT_MODELS=${PKM_MODEL_REVIEW:-gemma4-review}`, web search engine wired to `http://searxng:8080/search?q=<query>`, `ANONYMIZED_TELEMETRY=false`. |

> **Multi-bundle tip:** Enable `COMPOSE_PROFILES=tools` on only ONE bundle per server; leave it empty on a second bundle to avoid duplicate SearXNG / Open WebUI containers. On the second bundle, point `HERMES_SEARXNG_URL` at the first bundle's published SearXNG URL (e.g., `http://<host>:8888`), since `http://searxng:8080` only resolves within the same Compose project.

> **Pass environment variables to containers explicitly:** `--env-file` only interpolates `${VAR}` expressions inside `docker-compose.yml` — it does NOT inject variables into containers. Do NOT use `env_file:` (that would leak host-specific values like `PKM_ROOT=/srv/...` into containers). Instead, enumerate each variable with defaults under `environment:` in `rss-triage-app` and `hermes-agent`: `MAC_MINI_URL`, `PKM_MODEL_REVIEW/FAST/EMBED`, `PKM_LLM_TIMEOUT`, `PKM_API_KEY`, `TZ` (`${TZ:-Asia/Tokyo}`, all 4 services); app-only `NIKI_VAULT_NAME`, `NIKI_REVIEWER`, `NIKI_CRAWL_INTERVAL`, `NIKI_CRAWL_ON_START`, `NIKI_ARCHIVE_DAYS`, `NIKI_FETCH_FULLTEXT`, `NIKI_USER_AGENT`, `NIKI_SCORE_WEIGHTS`, `NIKI_SCORE_TAG_LIMIT`, `NIKI_SCORE_STOP_TAGS`, `HATENA_*`, `NIKI_XURL`, `SYNCTHING_*`; Hermes-only `SEARXNG_URL`, `HERMES_INTERVAL`, `HERMES_LIMIT`. Enforce via tests that every variable in `pkm.env.example` is referenced by Compose (06 §2; host-only variables used solely by systemd — `PKM_DB`, `GRAPHIFY_CMD`, `COMPOSE_PROFILES`, `NIKI_LINKS_*` — go in an exclusion list).

### 2.3 systemd Units & Timers (System Units — No Login Required)

Make **all 5 units `@` templates**, enabled per bundle with `%i` = `{{VAULT_NAME}}` (e.g., `sudo systemctl enable --now {{PROJECT_NAME}}-build-index@pkm.timer`). Shared settings:

```ini
[Unit]
# Do not run before the disk is mounted (systemd disallows inline comments)
RequiresMountsFor=/srv/%i
[Service]
Type=oneshot
User={{LINUX_USER}}
Group={{LINUX_USER}}
EnvironmentFile=/etc/{{PROJECT_NAME}}/%i.env
WorkingDirectory=/srv/%i
ExecStart={{REPO_DIR}}/.venv/bin/python /srv/%i/00_System/scripts/<script>.py
```

| Unit | Schedule | Command & Settings |
| :--- | :--- | :--- |
| `{{PROJECT_NAME}}-compose@.service` | On boot (`multi-user.target`) | `WorkingDirectory={{REPO_DIR}}`, `docker compose -p {{PROJECT_NAME}}-%i --env-file /etc/{{PROJECT_NAME}}/%i.env up -d --build --remove-orphans` (`Requires=docker.service`, `Type=oneshot`, `RemainAfterExit=yes`, `ExecStop=... down`, `TimeoutStartSec=900`; no `User=`) |
| `{{PROJECT_NAME}}-build-index@.timer` | Every 30 min (`*:0/30`, `RandomizedDelaySec=60`, `Persistent=true`) | `build_index.py` (`Nice=10`, `IOSchedulingClass=idle`, `TimeoutStartSec=2h`; also records lineage here) |
| `{{PROJECT_NAME}}-autocommit@.timer` | Daily `02:30` | `autocommit.py` (leaves gate-failing files unstaged with exit 1; push failure exits 3 so systemd alerts on failure) |
| `{{PROJECT_NAME}}-lint-wiki@.timer` | Daily `03:30` | `lint_wiki.py --save-db` (persists results to `pkm.sqlite` for the dashboard) |
| `{{PROJECT_NAME}}-graphify@.timer` | Daily `04:00` (`RandomizedDelaySec=5min`) | `run_graphify.py` → `related_links.py build` (two `ExecStart=` lines in the same `Type=oneshot` service; if graph building fails, no candidates are built and the previous night's remain. `Nice=15`, `TimeoutStartSec=4h`) |

**Nightly Job Ordering Rationale:** `02:30` auto-commit (lock daytime human edits into `HEAD`) → `03:30` Wiki health lint → `04:00` build knowledge graph from finalized Silver/Gold notes.

---

## 3. Environment Variable Reference (`/etc/{{PROJECT_NAME}}/{{VAULT_NAME}}.env`)

| Variable | Required | Default / Example | Purpose |
| :--- | :---: | :--- | :--- |
| `PKM_ROOT` | Required | `/srv/{{VAULT_NAME}}` | PKM bundle path on host (overridden to `/pkm` inside containers by Compose) |
| `MAC_MINI_URL` | Required | `http://{{INFERENCE_HOST}}:8080` | Base URL of inference server (`llama-swap`); must be reachable from both host and containers (see 01 §4 Pattern B) |
| `PKM_DB` | Optional | `$PKM_ROOT/00_System/pkm.sqlite` | Search & lint SQLite path |
| `PKM_LLM_TIMEOUT` | Optional | `300` | LLM response timeout (seconds) |
| `PKM_MODEL_REVIEW` / `FAST` / `EMBED` | Optional | `gemma4-review` / `gemma4-fast` / `gemma-embed` | Model ID overrides (match `llama-swap` `config.yaml`; also sets Open WebUI default model) |
| `PKM_API_KEY` | Optional | `local` | OpenAI-compatible API key (`llama-swap` has no auth; needed only behind an auth proxy) |
| `TZ` | Optional | `{{TIMEZONE}}` (default `Asia/Tokyo`) | Container timezone, Web UI timestamp display, and weekly boundaries |
| `NIKI_VAULT` | Optional | `pkm` | Compose project name suffix |
| `NIKI_APP_PORT` | Optional | `8000` | Published port for `rss-triage-app` |
| `NIKI_UID` / `NIKI_GID` | Recommended | `1000` / `1000` | Container runtime UID/GID (match bundle owner) |
| `NIKI_VAULT_NAME` | Optional | `pkm` | Vault name for Obsidian `obsidian://open` links |
| `NIKI_REVIEWER` | Recommended | `human:{{USER_ID}}` | Actor name used by the Re-review screen "Approve" button |
| `NIKI_CRAWL_INTERVAL` | Optional | `120` | Default source crawl interval (minutes) |
| `NIKI_CRAWL_ON_START` | Optional | `1` | Run one crawl immediately on startup (`0` to disable) |
| `NIKI_ARCHIVE_DAYS` | Optional | `7` | Days before unread items auto-archive |
| `NIKI_FETCH_FULLTEXT` | Optional | `1` | Extract full article text from web pages (`0` uses feed summary only) |
| `NIKI_USER_AGENT` | Optional | Empty (app default UA) | Custom User-Agent for crawling |
| `NIKI_SCHEDULER` | Optional | `1` | Set `0` to disable background scheduler (testing only; not passed in Compose) |
| `NIKI_SCORE_WEIGHTS` | Optional | Empty (= `relevance=8,originality=6,utility=6,noise_penalty=15`) | Weights for combining the interest score; on change, scored items are recomputed at next startup without the LLM |
| `NIKI_SCORE_TAG_LIMIT` | Optional | `20` | Number of interest tags (top of last 30 days) passed to the scoring prompt |
| `NIKI_SCORE_STOP_TAGS` | Optional | Empty | Overly broad tags to drop from interest tags (comma-separated, e.g., `tech,news,memo`) |
| `NIKI_LINKS_PER_NOTE` / `NIKI_LINKS_MIN_SIM` | Optional | `3` / `0.72` | Link candidates per note and minimum similarity (read by host-side `related_links.py`) |
| `NIKI_LINKS_CLASSIFY_LIMIT` | Optional | `30` | Max pairs whose relation the LLM labels per night (same) |
| `HERMES_INTERVAL` / `HERMES_LIMIT` | Optional | `300` / `5` | Hermes polling interval (seconds) & max items per cycle |
| `HERMES_SEARXNG_URL` | Optional | `http://searxng:8080` | SearXNG URL for Hermes (injected into container as `SEARXNG_URL`) |
| `COMPOSE_PROFILES` | Optional | `tools` (one bundle only) | Controls startup of SearXNG + Open WebUI |
| `SEARXNG_SECRET` | Required w/ `tools` | `openssl rand -hex 32` | SearXNG signing secret |
| `NIKI_SEARXNG_PORT` / `NIKI_WEBUI_PORT` | Optional | `8888` / `3000` | Published ports for SearXNG / Open WebUI |
| `SEARXNG_IMAGE` / `OPEN_WEBUI_IMAGE` | Recommended | Pinned date/version tag | Pin production container image tags |
| `GRAPHIFY_CMD` | Optional | `graphify {input} --output {output}` | External Graphify CLI command (if unset, generates `graph.json` only) |
| `HATENA_CONSUMER_KEY` / `SECRET` / `ACCESS_TOKEN` / `ACCESS_SECRET` | Optional | — | Hatena Bookmark OAuth 1.0a credentials (enabled when all 4 are set) |
| `NIKI_XURL` | Optional | `xurl` (`/opt/xurl/xurl` when mounted) | Path to official X CLI (`docker-compose.override.yml` mounts binary and `~/.xurl` → `/tmp/.xurl`) |
| `SYNCTHING_URL` / `SYNCTHING_API_KEY` | Optional | `http://<host-ip>:8384` | Displays Syncthing device connection status on the dashboard |

---

## 4. Client Setup (Workstation PC / Mobile)

1. **Obsidian + Obsidian Web Clipper:**
   - Open the Syncthing-synced `~/pkm/` folder as an Obsidian Vault.
   - Set attachment location to "In subfolder under current folder: `_assets`" (matches relative paths `./_assets/...`).
   - Configure Web Clipper destination to `10_Inbox/12_web-clipper/` (must NOT include a `type` key).
2. **AI Coding Agents (Claude Code / Antigravity / etc.):**
   - Install the official OKF Agent Skills (`okf-generate` / `okf-consume` from `open-knowledge-format/open-knowledge-format`).
   - Launch the agent from the bundle root so it loads `AGENTS.md` and synthesizes Gold notes / outputs with the local profile taking precedence.
3. **Mobile PWA:**
   - Open `http://{{APP_HOST}}:8000/` (`NIKI_APP_PORT`) over Tailscale and tap "Add to Home Screen".
   - Clip URLs directly to `/share-target` from your mobile browser's share menu.
