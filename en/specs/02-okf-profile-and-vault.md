# 02. PKM Bundle Structure, OKF v0.3 Local Profile & Agent Guidelines

This document defines the directory layout of the PKM bundle (`{{PKM_ROOT}}` = template `pkm/`), the strict schema of **PKM-OKF v0.3 Local Profile v1.1**, and the specification for `AGENTS.md` obeyed by all AI agents.

---

## 1. Medallion Architecture × Johnny.Decimal × OKF v0.3

| LLM Wiki Layer | Medallion | Folder | Format | Zettelkasten Equivalent | Editors |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Raw sources | **Bronze** | `10_Inbox/` | Non-OKF (no `type`) | Fleeting notes | Human / Web app / Web Clipper / Extraction scripts |
| Wiki | **Silver** | `20_Literature/` | OKF v0.3 | Literature notes (structured per source) | Autonomous worker (Hermes) / AI agents / Human |
| Wiki | **Gold** | `50_Knowledge/` | OKF v0.3 (`human:` verification required) | Permanent notes (synthesized across sources) | Human + AI agent dialogue (human-only approval) |
| Schema | — | `00_System/` | Specs, schemas, scripts, DBs, logs | — | System (read-only for agents during normal use) |
| Output | **Output** | `60_Output/` | Non-OKF (derivatives of Gold) | Deliverables (articles, scripts, etc.) | Human + AI agents |
| Assets | — | `90_Assets/` | Git-ignored (large media) | — | Human (synced via Syncthing only) |

---

## 2. Bundle Directory Structure

```text
pkm/
├── AGENTS.md                        # Shared agent code of conduct (non-OKF)
├── index.md                         # [Auto-generated] Bundle root catalog (holds okf_version: "0.3" in frontmatter)
├── log.md                           # [Auto-generated] OKF standard update log generated from lineage.jsonl
├── .gitignore                       # Git exclusion rules
├── .stignore                        # Syncthing exclusion rules
├── 00_System/
│   ├── specs/
│   │   └── okf_local_profile.md     # In-bundle copy of this specification
│   ├── schemas/
│   │   └── okf_v03.py               # Pydantic v2 validation models
│   ├── scripts/                     # Deterministic scripts (03-guardians-and-pipeline.md)
│   ├── graph/                       # [Git/Syncthing-ignored] Knowledge graph output (track .gitkeep only)
│   ├── logs/
│   │   ├── lineage.jsonl            # Audit log of promotions & review-pending state changes
│   │   └── rejected/                # [Git-ignored] Drafts saved after failed self-repair
│   ├── pkm.sqlite                   # [Git/Syncthing-ignored] DB for build_index / lint_wiki
│   ├── rss_triage.sqlite            # [Git/Syncthing-ignored] State DB for rss-triage-app
│   └── hermes.sqlite                # [Git/Syncthing-ignored] Processing state DB for hermes.py
├── 10_Inbox/                        # [Bronze] Non-OKF
│   ├── index.md
│   ├── 11_rss/                      # Articles kept from RSS & non-RSS sites
│   ├── 12_web-clipper/              # Articles clipped via Web Clipper or PWA share menu
│   ├── 13_quick-notes/              # Human handwritten memos & fleeting thoughts
│   ├── 14_podcast/                  # Podcast transcripts ingested via Send to PKM
│   ├── 15_files/                    # PDFs, Office docs, images + auto-generated <name>.extracted.md
│   └── 19_private/                  # [Git-ignored] Kept articles & transcripts from "private" sources (Silver locked to 28_me)
├── 20_Literature/                   # [Silver] OKF v0.3 (category IDs 21–29; 20 is reserved)
│   ├── index.md
│   ├── 21_technology/               # Example standard categories (freely customizable by user)
│   ├── 22_politics-economy/
│   ├── 23_society/
│   ├── 24_life/
│   ├── 25_learning/
│   ├── 26_culture/
│   ├── (27_ spare)
│   ├── 28_me/                       # Private category (excluded from Git / lineage / Share)
│   └── 29_misc/                     # Fallback category
├── 50_Knowledge/                    # [Gold] OKF v0.3 + human: verification (category IDs 51–59; 50 is reserved)
│   ├── index.md
│   ├── 51_concepts/                 # Principles, concepts, mental models
│   └── 52_entities/                 # Verified profiles of people, tools, organizations
├── 60_Output/                       # [Output] Non-OKF
│   ├── index.md
│   ├── 61_youtube/                  # Example output folders (freely customizable by user)
│   └── 62_note/
└── 90_Assets/                       # [Git-ignored] Large media, audio, corpora (track .gitkeep only)
```

### 2.1 Naming Rules & Reserved Names
- **Area folders:** `NN_Name` (no whitespace or `&`). `30_` and `40_` are reserved for future expansion.
- **Category folders:** Must match regex `^(\d{2})_[a-z0-9]+(?:-[a-z0-9]+)*$`.
  - `20_Literature`: `21`–`29` (`20` is reserved). Do not create subfolders under a category (use `tags` for finer taxonomy).
  - `50_Knowledge`: `51`–`59` (`50` is reserved).
- **Note filenames:**
  - Forbidden characters: `[\s&#%?<>\"|*]` (error).
  - Recommended format: lowercase kebab-case slug `^[a-z0-9]+(?:-[a-z0-9]+)*\.md$` (mismatch produces a warning). Do not assign Johnny.Decimal numbers to individual notes (avoids the 99-items-per-category limit).
- **Reserved filenames:** `index.md` and `log.md` are reserved at every level (cannot be used as notes).
- **Note-attached images:** Place in `_assets/` inside the same category folder as the note and reference via relative paths such as `./_assets/diagram.png` (tracked in Git; do NOT put in `90_Assets/`).
- **No MoC (Map of Content):** Auto-generated `index.md` files at every level serve as catalogs. If you need a curated overview, write a regular `concept` note.

### 2.2 Category Definitions (Include in Template)
Category descriptions live in two places, and `hermes.py` injects both into its prompt:

1. **Area `index.md` catalog lines** (`20_Literature/index.md`): Pre-populate the template with descriptive lines in the format below. `build_index.py` preserves the label and description while updating only the count, and `hermes.py` parses them with regex `^\*\s+\[[^\]]+\]\((?P<dir>\d\d_[a-z0-9-]+)/index\.md\)\s*-\s*(?P<desc>.+?)(?:（\d+ 件）)?\s*$` (**this format is a contract between the two scripts**; if you localize the count suffix, keep both scripts in sync):
   ```markdown
   * [Technology](21_technology/index.md) - IT, AI, software, hardware, networking（0 件）
   ```
2. **The `### 20_Literature のカテゴリ` section in `AGENTS.md`**: Document boundary guidelines in a table (§7).

Example standard categories:

| Folder | What Goes Here | Boundary Guidelines |
| :--- | :--- | :--- |
| `21_technology` | IT, AI, LLMs, software, hardware, networking, security | If the focus is an IT company's earnings or M&A, put it in 22 |
| `22_politics-economy` | Politics, government, diplomacy, geopolitics, macroeconomics, markets, corporate management | Incidents or scandals arising from policy execution go in 23 |
| `23_society` | Incidents, accidents, disasters, labor, welfare, education, healthcare news | Practical everyday know-how you personally apply goes in 24 |
| `24_life` | Career, productivity, personal finance, investing, housing, health | Actionable personal-life knowledge rather than general news |
| `25_learning` | Study, languages, science, math, statistics, history, learning methods | Computer science / IT academic topics go in 21 |
| `26_culture` | Sports, entertainment, anime, games, music, film, books | Game development engineering goes in 21 |
| `27_` | Spare (do not create) | — |
| `{{PRIVATE_CATEGORY}}` (`28_me`) | Information about yourself or your family | Git-ignored. Silver notes from `10_Inbox/19_private/` are locked here |
| `29_misc` | Anything that fits nowhere else (**required**: fallback when `hermes.py` planning fails) | When unsure, pick the closest topic and supplement with `tags` |

### 2.3 Bundle Root `index.md` (OKF v0.3 §8.1)
Write `okf_version` ONLY in the frontmatter of the root `index.md`.

```yaml
---
okf_version: "0.3"
title: "{{USER_ID}} PKM"
description: "Personal LLM Wiki grown from RSS, Podcasts, and web clips"
---
```

`build_index.py` regenerates only content below the marker `<!-- build_index.py: この行より下は自動生成。手で編集しない -->` (if localized, keep the marker identical in `index.md` and `build_index.py`), preserving the frontmatter and any handwritten heading/description above the marker. Subdirectory `index.md` files have no frontmatter.

---

## 3. Frontmatter Schema Definition (Shared by Silver & Gold)

Validator implementation: `00_System/schemas/okf_v03.py` (Pydantic v2 with `ConfigDict(extra="allow")` to preserve unknown keys).

### 3.1 MUST Keys (Missing or Invalid Fails the Gate)

| Key | Type / Constraints | Description |
| :--- | :--- | :--- |
| `type` | `"knowledge"` \| `"concept"` \| `"entity"` | OKF structured file discriminator (`moc` is retired). |
| `title` | `str` (1–150 chars) | Note heading, catalog title, and search title. |
| `description` | `str` (1–400 chars) | 1–3 sentence summary used in catalog descriptions and search snippets. |
| `generated` | `{ by: str, at: Timestamp }` | Provenance of the last semantically meaningful content change.<br>• `by`: `<producer>/<version>` (`^[a-z0-9][a-z0-9._-]*/[a-z0-9][a-z0-9._-]*$`) or `human:<id>` (`^human:[a-z0-9][a-z0-9._-]*$`)<br>• `at`: RFC 3339 string with mandatory offset (e.g., `"2026-11-20T09:15:00+09:00"`) or date-only (treated as UTC 00:00). Native YAML `datetime` is accepted, but quoted strings are recommended.<br>• Do NOT update for formatting fixes, whitespace tweaks, link repairs, or writing `x-pkm-medal`. |

### 3.2 SHOULD Keys (Missing Triggers Only a Warning in `lint_wiki.py`)

| Key | Type / Constraints | Description |
| :--- | :--- | :--- |
| `tags` | `list[str]` (0–15 items) | Each item matches `^[a-z0-9]+(?:-[a-z0-9]+)*$` (lowercase kebab-case). Duplicates are errors. Reserved words `bronze`, `silver`, `gold` are **errors**. |
| `sources` | `list[SourceItem]` | Array of primary sources. Each item:<br>• `resource` (required, `min_length=1`): absolute URL or in-bundle path<br>• `id`, `title`, `author`, `last_modified` (optional, OKF standard)<br>• `x-pkm-accessed-at` (optional, `Timestamp`) |
| `x-pkm-my-comment` | `Optional[str]` | The human's own primary reaction/reflection. **Editable by humans only.** Agent edits or deletions fail the gate. (Optional, so excluded from `lint_wiki.py`'s missing-SHOULD warnings.) |

### 3.3 Additional Mandatory Rules for Gold (`50_Knowledge/`)

| Key | Type / Constraints | Description |
| :--- | :--- | :--- |
| `verified` | `list[VerificationEvent]` (single dict allowed when 1 entry) | Each item `{ by: str, at: Timestamp }`. `by` is `<producer>/<version>` or `<scheme>:<id>` (`human:`, `process:`, `team:`, etc.). **Append-only.** |
| `sources` | As above | **Must include at least 1 existing Silver or Gold note inside the bundle (excluding self)** — deterministically prevents placing notes directly into Gold. |

### 3.4 Optional OKF Standard Keys, Local Extension Keys & Forbidden Keys

- **Optional OKF Standard Keys:**
  - `status`: `"draft"` \| `"stable"` \| `"deprecated"` (defaults to `"stable"`)
  - `superseded_by`: `Optional[str]` (allowed ONLY when `status == "deprecated"`; must resolve to an existing path inside the bundle)
  - `aliases`: `list[str]` (standard Obsidian key for alias resolution)
- **Local Extension Keys (`x-pkm-*`):**
  - `x-pkm-medal`: `"silver"` \| `"gold"`. **Written exclusively by the gate (`gate_okf.py --fix`) based on path.** LLMs must never write it. Mismatch with path is an error.
  - `x-pkm-lang`: `"ja"` \| `"en"` (defaults to `"ja"`).
  - `x-pkm-my-comment`: Human only.
  - `sources[].x-pkm-accessed-at`: Source access timestamp.
- **Forbidden (Legacy) Keys in Notes (Gate Returns Error + Remediation Instructions):**
  - Top-level note keys: `okf_version`, `jd_id`, `medal`, `lang`, `my_comment`
  - Inside `sources[]`: `url`, `path`, `accessed_at`

---

## 4. Derived Values, Trust Tiers & Gold State Transitions

### 4.1 Values Derived from Path
- `medal`: `20_Literature` → `"silver"`, `50_Knowledge` → `"gold"`
- `jd_id`: Leading 2 digits of the parent category folder (e.g., `21_technology` → `"21"`)

### 4.2 Trust Tier Determination (OKF v0.3 §5.3 — Order-Independent)
1. **Current Verifications:** All events in `verified` satisfying `event.at >= generated.at`.
2. Evaluation rules (**NEVER inspect only the last array element `verified[-1]`**):
   - At least one `human:` event in current verifications → **`human-reviewed`**
   - No `human:`, but at least one other actor (`process:`, `team:`, `<producer>/<version>`) → **`machine-confirmed`**
   - Zero current verifications → **`unverified`**

### 4.3 Gold Note State Transitions

| State | Condition | Gate Handling |
| :--- | :--- | :--- |
| **Approved** | `50_Knowledge/` and `trust_tier == "human-reviewed"` | Pass (`review_pending = False`) |
| **Review Pending** | `50_Knowledge/` and `trust_tier != "human-reviewed"` and at least 1 historical `human:` verification exists | **Pass (not an error; file left untouched).** Sets `review_pending = True`, records `review_pending` in `lineage.jsonl`, and surfaces it in `index.md` and the Web UI badge |
| **Unapproved Placement** | `50_Knowledge/` and 0 historical `human:` verifications | **Error** (human approval is mandatory for promotion) |

After re-reviewing, when the human appends a new `{ by: human:{{USER_ID}}, at: <now> }` via `verify_note.py` or the UI "Approve" button and passes the gate, the note automatically returns to Approved and `review_cleared` is recorded in `lineage.jsonl`.

**Lineage Recording Timing (Designed Against Missed Events):**
- `build_index.py` (every 30 minutes) detects state changes and records `review_pending`. Even if an agent edits a Gold note directly without running the gate, the transition is caught.
- `verify_note.py` runs the gate (with lineage recording enabled) **before** appending the approval so any unrecorded `review_pending` is logged first. This guarantees a `review_pending` → `review_cleared` pair is always recorded even if the human approves before `build_index.py` runs.
- When a Gold note passes the gate for the first time (= initial human approval), `promoted` (`medal: "gold"`, with `at` set to the latest `human:` verification timestamp) is recorded.

---

## 5. Path Resolution Rules (OKF v0.3 §6)

Resolve `sources[].resource`, `superseded_by`, and body Markdown links `[text](target)` / `![alt](target)` (excluding code fences and inline code) in the following order:

1. Starts with a URI scheme (`^[A-Za-z][A-Za-z0-9+.-]*:`) → `"url"` (external URL; exempt from local existence check)
2. Fragment-only starting with `#` → same-file anchor (exempt)
3. Percent-decode (`urllib.parse.unquote`) and strip `#fragment`
4. Leading `/` → relative to bundle root: `(pkm_root / path.lstrip("/")).resolve()`
5. Otherwise → relative to the file's own directory: `(file_path.parent / path).resolve()`
6. Resolved path escapes `pkm_root` → `"external"` (error)
7. Inside `pkm_root` but does not exist → error

---

## 6. The 7 Rules of the Deterministic Gate (`gate_okf.py`)

| # | Rule Name | Validation Logic | On Failure |
| :--- | :--- | :--- | :--- |
| 1 | **Bronze Contamination Guard** | No file under `10_Inbox/` may have a `type` key | Error |
| 2 | **Placement, Naming & Legacy Key Check** | Located in a category folder under `20_Literature/` or `50_Knowledge/`; category ID in range (21–29 / 51–59); no forbidden chars in filename; `x-pkm-medal` matches placement; no legacy keys; not a reserved filename (`index.md`/`log.md`) | Error (`x-pkm-medal` alone is auto-written by `--fix` when no other errors exist) |
| 3 | **Gold Approval & Synthesis Check** | Gold must cite ≥ 1 Silver/Gold note in `sources` and must not be unapproved | Error for unapproved or direct placement / `review_pending` (warning) for expired verification |
| 4 | **Actor Naming Rules** | `generated.by` is `<producer>/<version>` or `human:<id>`; `verified[].by` is a valid actor format | Error |
| 5 | **Link & Reference Existence** | Every target of `sources[].resource`, `superseded_by`, and body Markdown links must exist inside the bundle | Error |
| 6 | **Tag Validation** | Lowercase kebab-case, ≤ 15 tags, no duplicates, no reserved words (`bronze`/`silver`/`gold`) | Error |
| 7 | **Human Authorship Protection** | Diffing against `git show HEAD:./<name>`, reject if `generated.by` is not `human:` and `x-pkm-my-comment` was altered/deleted, or if the existing prefix of `verified` was altered, deleted, or reordered | Error |

---

## 7. Mandatory Contents of `AGENTS.md` (Agent Code of Conduct)

Place `AGENTS.md` at the bundle root so every agent (Claude Code, Antigravity, Hermes, Open WebUI, etc.) follows these explicit rules:

1. **Core Principles:** Separation of probabilistic and deterministic layers, mandatory execution of `gate_okf.py`, immutability of human-authored fields.
2. **Folder & Role Matrix:** Read/write permissions per area (including `10_Inbox/19_private/`), category definitions and boundary guidelines for `20_Literature/` (§2.2). Because `hermes.py` automatically extracts from the heading `### 20_Literature のカテゴリ` to the next `##` / `###` heading (or EOF) and injects it into its prompt, **do not rename this heading** (unless you update `hermes.py` to match).
3. **Actor Naming Table:**
   - Autonomous worker: `hermes/gemma4-12b` (or active model name)
   - Interactive agents: `claude-code/<model>`, `antigravity/<model>`, etc.
   - Human: `human:{{USER_ID}}`
   - Scripts: `process:gate-okf`, `process:build-index`, `process:lint-wiki`
4. **Allowed vs. Forbidden Keys Table** (handling of `x-pkm-medal`, `okf_version`, `jd_id`, `x-pkm-my-comment`, `verified`).
5. **Silver → Gold Synthesis Procedure** (never move Silver; synthesize a new note; only the human verifies).
6. **Auto-Generated Artifacts — Do Not Edit List** (`index.md`, `log.md`, `lineage.jsonl`, `*.extracted.md`, `00_System/graph/`).
7. **Rules for Combining with Official OKF Agent Skills (`okf-generate` / `okf-consume`):**
   - Precedence diff table where the local profile overrides generic official skills (restrict `type` to 3 values, `description` 1–3 sentences, skills must not write `index.md`/`log.md`, mandatory `gate_okf.py` run, footnote syntax `[^<id>]`).
8. **Summary of Prohibitions.**

---

## 8. Mandatory Exclusions in `.gitignore` and `.stignore`

### `pkm/.gitignore`
- `90_Assets/*` (`!90_Assets/.gitkeep`)
- `20_Literature/{{PRIVATE_CATEGORY}}/` and `10_Inbox/19_private/` (private compartments; exclude the whole directory because even the category `index.md` contains note titles — track the template's `index.md` via `git add -f`)
- `00_System/graph/*` (`!00_System/graph/.gitkeep`)
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
