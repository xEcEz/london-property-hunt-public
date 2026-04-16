# Flat Hunt Variant — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the `london-property-hunt-public` repo with a whole-unit 1–2 bed flat hunt variant, then configure Raphael's personal hunt (Google Sheet tracker + local cron + remote fallback trigger) for daily automated search around King's Cross.

**Architecture:** Add a second skill prompt (`skill-flats.md`) alongside the existing `skill.md`; users select via `HUNT_MODE=rooms|flats` in their config. Flats variant uses a Google Sheet tracker (not xlsx) so both the local laptop cron (10:00) and a remote Anthropic-hosted fallback trigger (15:00) can read/write the same state. Remote no-ops if local already stamped the sheet.

**Tech Stack:**
- Claude Code skill prompts (markdown)
- Google Sheets API via Drive MCP (`mcp__claude_ai_Google_Drive__*`)
- Gmail MCP (`mcp__claude_ai_Gmail__*`)
- Claude-in-Chrome MCP (local-only, browser extension)
- `crontab` for local scheduling
- Claude Code `CronCreate` for remote triggers
- Python + `openpyxl` (existing — unchanged for rooms variant)

**Source spec:** [`../specs/2026-04-16-flat-hunt-variant-design.md`](../specs/2026-04-16-flat-hunt-variant-design.md)

**Branch/worktree:** currently on `main`. All Phase 1 changes are additive markdown/template edits — safe to commit directly to `main`. If you prefer isolation, create a worktree first (optional).

---

## Phase 1 — Repo: docs and templates (agent-executed)

### Task 1: Add flats-mode fields to `config.example.md`

**Files:**
- Modify: `config.example.md` (append a new "Flats mode" section, keep all existing fields intact)

- [ ] **Step 1: Read current `config.example.md` to confirm it still matches expected baseline**

Run: verify the top of the file starts with `# Property Hunt — Personal Config` and the sections `## About you`, `## Commute`, `## Target areas`, `## Budget` exist.

- [ ] **Step 2: Prepend `HUNT_MODE` to the `## About you` section**

Add immediately after the existing frontmatter block (after line 6 — the `---` separator), before `## About you`:

```markdown
## Mode

```
HUNT_MODE=rooms
```

`HUNT_MODE` selects which skill variant runs:
- `rooms` — shared-room hunt (uses `skill.md`, xlsx tracker)
- `flats` — whole-unit 1–2 bed hunt (uses `skill-flats.md`, Google Sheet tracker)

All fields below that begin with `FLAT_*` or are under a `## Flats mode` section are ignored in `rooms` mode, and vice versa.

---
```

- [ ] **Step 3: Append the `## Flats mode` section at the end of `config.example.md`**

Append after the existing `## Notes` section:

```markdown
---

## Flats mode (only used if `HUNT_MODE=flats`)

### Unit criteria

```
FLAT_BEDROOMS_MIN=1
FLAT_BEDROOMS_MAX=2
FLAT_SIZE_FLOOR_M2=60
FLAT_FURNISHED_PREFERENCE=unfurnished   # unfurnished | furnished | any
```

### Budget

```
FLAT_BUDGET_SWEET_MAX=3200
FLAT_BUDGET_HARD_CAP=3500
FLAT_STRETCH_PENALTY=1
```

### Areas (ordered by priority)

```
FLAT_PRIMARY_AREAS=Islington, Angel, Camden Town, Kentish Town, De Beauvoir Town, Highbury, Canonbury, Clerkenwell
FLAT_SECONDARY_AREAS=Tufnell Park, Holloway, Bloomsbury, Russell Square, Barbican, Finsbury Park, London Bridge, Bermondsey
```

### Scoring weights (overrides — defaults in skill-flats.md)

```
FLAT_WEIGHT_TOP_FLOOR=3
FLAT_WEIGHT_NEW_BUILD=3
FLAT_WEIGHT_CALM=2
FLAT_WEIGHT_BATHTUB=1
FLAT_WEIGHT_LIGHT=1
FLAT_WEIGHT_WOODEN_FLOOR=1
FLAT_TIER_HIGH_THRESHOLD=7
FLAT_TIER_MEDIUM_THRESHOLD=4
```

### Tracker (Google Sheet)

```
FLAT_TRACKER_SHEET_ID=1aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789AbCdEf
FLAT_TRACKER_FLATS_TAB=Flats
FLAT_TRACKER_META_TAB=Meta
```

Create a Google Sheet titled "London Flat Hunt" in the Drive account authenticated with Drive MCP. Paste its ID (from the URL, between `/d/` and `/edit`) above.

### Outreach

```
FLAT_OUTREACH_STYLE=agent   # agent | landlord
FLAT_OUTREACH_AVAILABILITY=weekday evenings and weekends
```
```

- [ ] **Step 4: Verify the modified file**

Run: read `config.example.md` and confirm:
- `HUNT_MODE=rooms` appears near the top (under `## Mode`)
- `## Flats mode` section exists at the end with all `FLAT_*` fields listed

- [ ] **Step 5: Commit**

```bash
git add config.example.md
git commit -m "config: add HUNT_MODE and flats-mode fields

Introduces HUNT_MODE=rooms|flats selector and a new Flats mode
section with fields for unit criteria, budget bands, area lists,
scoring weights, Google Sheet tracker ID, and outreach style.

Rooms mode fields are untouched and remain the default."
```

---

### Task 2: Create `tracker/flats-schema.md`

**Files:**
- Create: `tracker/flats-schema.md`

- [ ] **Step 1: Create the file with the full schema**

Write `tracker/flats-schema.md`:

```markdown
# Flats Tracker — Schema (Google Sheet)

The flats variant of the skill reads and writes a Google Sheet (not an xlsx file). This lets the local laptop cron and the remote Anthropic-hosted fallback trigger share state.

## File location

Stored in the Google Drive account authenticated with the Drive MCP. Sheet ID is set in `config.md` as `FLAT_TRACKER_SHEET_ID`.

## Tabs

1. `Flats` — one row per listing
2. `Meta` — run stamps and skill metadata

---

## `Flats` tab

| # | Column | Type | Notes |
|---|---|---|---|
| 1 | URL | text | Dedup key (hyperlink) |
| 2 | Platform | enum | Rightmove / Zoopla / OpenRent / SpareRoom |
| 3 | Found On | date | First-seen date |
| 4 | Title | text | Listing title |
| 5 | Area | text | Must match a value in `FLAT_PRIMARY_AREAS` or `FLAT_SECONDARY_AREAS` |
| 6 | Price | int | £ pcm |
| 7 | Beds | int | 1 or 2 |
| 8 | Size | number | m² (may be empty if size unknown) |
| 9 | Size Source | enum | stated-text / floorplan / inferred-2bed / inferred-1bed-photos / unknown |
| 10 | Furnished | enum | Furnished / Part / Unfurnished / Unknown |
| 11 | Available From | date | Stated availability |
| 12 | Floor | enum | Top / Mid / Ground / Unknown |
| 13 | Bathtub | enum | Yes / No / Unknown |
| 14 | New/Renovated | enum | Yes / No / Unknown |
| 15 | Calm | enum | Yes / No / Unknown |
| 16 | Light | enum | Good / Avg / Poor / Unknown |
| 17 | Wooden Floor | enum | Yes / No / Unknown |
| 18 | Score | number | 0–11, one decimal |
| 19 | Tier | enum | HIGH / MEDIUM / LOW |
| 20 | Status | enum | NEW 🔴 / CONTACTED / VIEWING / REJECTED / GONE |
| 21 | Reason | text | One-line scoring summary |
| 22 | Needs-verify | text | Comma-separated list of unresolved signals |
| 23 | Notes | text | Free-form |

### Row colours (tier-based)

| Tier | Fill hex |
|---|---|
| HIGH | `#E2EFDA` (green) |
| MEDIUM | `#FFFFC7` (yellow) |
| LOW | `#FCE4D6` (red) |

### Deduplication

Skill loads all values from column 1 (`URL`) into a set at the start of each run. New rows with duplicate URLs are silently dropped — every run is idempotent.

---

## `Meta` tab

| Cell | Purpose |
|---|---|
| `A1` | label: `Key` |
| `B1` | label: `Value` |
| `A2` | `last_local_run` |
| `B2` | ISO date string (`YYYY-MM-DD`) of most recent successful local run |
| `A3` | `last_remote_run` |
| `B3` | ISO date string (`YYYY-MM-DD`) of most recent successful remote run |
| `A4` | `schema_version` |
| `B4` | `1` |

The remote fallback trigger compares `B2` against today's date. If equal → no-op. If not → run flats skill in WebFetch mode, then write today's date to `B3`.

---

## Setup

One-time workbook creation is automated via a small Python script (Google Sheets API) or can be done manually:

1. Create a new Google Sheet titled "London Flat Hunt"
2. Rename the default tab to `Flats`, add a second tab named `Meta`
3. Paste column headers into row 1 of `Flats` (see table above)
4. Fill `Meta!A1:B4` per the cell map above (leave `B2` and `B3` empty initially)
5. Freeze row 1 on `Flats`; enable filter on row 1
6. Copy the Sheet ID from the URL into `config.md` as `FLAT_TRACKER_SHEET_ID`

The skill creates missing tabs and headers on first run if the workbook is empty, but starting from the correct structure avoids first-run friction.
```

- [ ] **Step 2: Verify the file exists and contains the `Meta` tab section**

Run: confirm `tracker/flats-schema.md` exists and contains the strings `last_local_run` and `schema_version`.

- [ ] **Step 3: Commit**

```bash
git add tracker/flats-schema.md
git commit -m "tracker: add Google Sheet schema for flats variant

Documents the 23-column Flats tab and the Meta tab holding run
stamps (last_local_run, last_remote_run) used by the remote
fallback trigger to decide whether to no-op."
```

---

### Task 3: Create `skill-flats.md`

**Files:**
- Create: `skill-flats.md`

- [ ] **Step 1: Create the file with the full skill prompt**

Write `skill-flats.md`:

````markdown
# London Flat Hunt — Claude Code Skill (1–2 bed variant)

> **How to use:** Copy this file's contents into Claude Code as a skill, or run it directly.
> Before running, fill in your personal details in `config.md` (copy from `config.example.md`, set `HUNT_MODE=flats`).
> All `[PLACEHOLDER]` values below are read from your config file.

---

## Skill prompt (paste into Claude Code)

```
You are running [YOUR_NAME]'s London flat hunt (1–2 bed whole-unit rentals). Do this automatically — no user present. Complete all steps fully.

## RUNTIME MODE

- Local runs (Claude-in-Chrome available): use the browser MCP for Rightmove and Zoopla photos/floorplans.
- Remote runs (WebFetch only): use WebFetch for all listings; floorplan extraction may be degraded — mark `Size Source=unknown` and `Needs-verify: size (remote run)` when floorplans can't be loaded.

Detect remote mode by checking if `Claude-in-Chrome` MCP tools are available. If not → WebFetch mode.

## WHO IS [YOUR_NAME]
- [YOUR_AGE]yo [YOUR_NATIONALITY] [YOUR_PROFESSION] at [YOUR_EMPLOYER_TYPE]
- Permanent contract, single, non-smoker, clean, respectful
- Office near King's Cross ([YOUR_WORK_POSTCODE] if known, else "King's Cross area")
- Move-in: ~[MOVE_IN_DATE]

## HARD FILTERS (AUTO-SKIP)

Reject outright if any of:
- Beds < [FLAT_BEDROOMS_MIN] or > [FLAT_BEDROOMS_MAX]
- Rent > £[FLAT_BUDGET_HARD_CAP] pcm
- Area not in [FLAT_PRIMARY_AREAS] ∪ [FLAT_SECONDARY_AREAS]
- Stated size < [FLAT_SIZE_FLOOR_M2] m²
- Obviously dated/unrenovated (tired photos, old fittings, yellowing bathroom)

## SIZE RESOLUTION (priority order)

1. Stated in listing text → use.
2. Stated on floorplan → extract visually.
3. 2-bed + no size → assume ≥60 m², accept, set `Size Source=inferred-2bed`.
4. 1-bed + no size (no text, no floorplan):
   - INCLUDE but cap tier at MEDIUM
   - Add `Needs-verify: size` to `Needs-verify` column
   - If photos show large living room + separate kitchen + proper double bedroom → `Size Source=inferred-1bed-photos`
   - Else → `Size Source=unknown`

## SCORING

Base score (0–11) = sum of signal weights below. Then apply modifiers. Then clamp to 0–11.

Signal weights (defaults; overridable from config):
- Top floor / no neighbor above: [FLAT_WEIGHT_TOP_FLOOR]
- New build or recent refurb (≤5y): [FLAT_WEIGHT_NEW_BUILD]
- Calm street: [FLAT_WEIGHT_CALM]
- Bathtub: [FLAT_WEIGHT_BATHTUB]
- Light/windows: [FLAT_WEIGHT_LIGHT]
- Wooden floor: [FLAT_WEIGHT_WOODEN_FLOOR]

Modifiers:
- Area in [FLAT_PRIMARY_AREAS]: +1
- Area in [FLAT_SECONDARY_AREAS]: −1
- Unfurnished or part-furnished: +0.5
- Rent in stretch band £[FLAT_BUDGET_SWEET_MAX + 1] – £[FLAT_BUDGET_HARD_CAP]: −[FLAT_STRETCH_PENALTY]

Tiers:
- HIGH: score ≥ [FLAT_TIER_HIGH_THRESHOLD]
- MEDIUM: [FLAT_TIER_MEDIUM_THRESHOLD] ≤ score < [FLAT_TIER_HIGH_THRESHOLD]
- LOW: score < [FLAT_TIER_MEDIUM_THRESHOLD]

**Cap:** size-unknown 1-beds (rule 4 above) → tier cannot exceed MEDIUM even if score ≥ HIGH threshold.

## PLATFORMS

Search all four. URLs assembled from config areas + budget. Max price = £[FLAT_BUDGET_HARD_CAP].

- Rightmove: https://www.rightmove.co.uk/property-to-rent/find.html?searchType=RENT&locationIdentifier=[REGION_CODE]&minBedrooms=[FLAT_BEDROOMS_MIN]&maxBedrooms=[FLAT_BEDROOMS_MAX]&maxPrice=[FLAT_BUDGET_HARD_CAP]&propertyTypes=flat&includeLetAgreed=false&sortType=6
- Zoopla: https://www.zoopla.co.uk/to-rent/flats/[AREA_SLUG]/?beds_min=[FLAT_BEDROOMS_MIN]&beds_max=[FLAT_BEDROOMS_MAX]&price_frequency=per_month&price_max=[FLAT_BUDGET_HARD_CAP]&results_sort=newest_listings
- OpenRent: https://www.openrent.co.uk/properties-to-rent/london?term=[AREAS_CSV]&prices_max=[FLAT_BUDGET_HARD_CAP]&bedrooms_min=[FLAT_BEDROOMS_MIN]&bedrooms_max=[FLAT_BEDROOMS_MAX]&isLive=true
- SpareRoom: https://www.spareroom.co.uk/flats-to-rent/london/[AREA_SLUG]?max_rent=[FLAT_BUDGET_HARD_CAP]&min_beds=[FLAT_BEDROOMS_MIN]&max_beds=[FLAT_BEDROOMS_MAX]&sort=posted_date&mode=list

## TRACKER

Google Sheet ID: [FLAT_TRACKER_SHEET_ID]
- Flats tab: [FLAT_TRACKER_FLATS_TAB] (default: `Flats`)
- Meta tab: [FLAT_TRACKER_META_TAB] (default: `Meta`)

Columns (see tracker/flats-schema.md): URL, Platform, Found On, Title, Area, Price, Beds, Size, Size Source, Furnished, Available From, Floor, Bathtub, New/Renovated, Calm, Light, Wooden Floor, Score, Tier, Status, Reason, Needs-verify, Notes.

Dedup: skip if URL already in column 1.
New rows: Status = NEW 🔴, Found On = today (YYYY-MM-DD).

Row colours based on Tier:
- HIGH: #E2EFDA
- MEDIUM: #FFFFC7
- LOW: #FCE4D6

## REMOTE FALLBACK LOGIC (WebFetch mode only)

Before doing anything else in WebFetch mode:
1. Read Meta!B2 (`last_local_run`).
2. If equal to today's YYYY-MM-DD → exit silently. Do not email. Do not scrape.
3. Else → proceed with full search. On success, write today's YYYY-MM-DD to Meta!B3 (`last_remote_run`).

In local mode (Claude-in-Chrome available): on success, write today's YYYY-MM-DD to Meta!B2 (`last_local_run`).

## EMAIL — SELF-CONTAINED, PHONE-READY

Use `gmail_create_draft` (contentType: text/html), To: [YOUR_EMAIL].

Subject: 🏠 London Flat Hunt — {DATE} — {N} new (H:{high}/M:{med}/L:{low})

HTML body sections:

**A — Header:** Date, run source (Local / Remote fallback), platform counts, tier totals.

**B — 🟢 HIGH New Today:** For EVERY HIGH listing, a card sorted by score desc:
- Title as large clickable link | Area | £price | Beds | Size (or "unverified") | Floor | Furnished | Platform
- Score badge (e.g. "8.5 / 11")
- One-line reason: "top floor ✅, new build ✅, on Camden Rd ❌ (loud)"
- Needs-verify flags (if any)
- Green styled box with ready-to-send outreach (see OUTREACH below)

**C — 🟡 MEDIUM New Today:** Condensed card — Title (link) | Area | £ | Beds | Size | Score | short reason | short outreach box.

**D — ⚪ LOW / SKIP:** Bullet list only — title (link), one-line reason.

**E — 🔁 Backlog (uncontacted HIGH, not today):** Up to 8 rows from `Flats` where Status = NEW 🔴, Tier = HIGH, Found On ≠ today. Primary areas first. Each with card + ready-to-send outreach.

**F — Stats:** Totals, area breakdown, days-to-[MOVE_IN_DATE], next run time. End: "⚠️ X days to [MOVE_IN_DATE]. Message at least 3 listings today."

Always send, even on zero new listings — confirms pipeline ran.

After creating draft: navigate to https://mail.google.com/mail/u/[GMAIL_ACCOUNT_INDEX]/#drafts, open the latest draft, click Send. (In remote WebFetch mode, use `gmail_send_message` directly if available; skip the browser navigation.)

## OUTREACH

Save .txt files for HIGH priority to [YOUR_HUNT_DIR]/outreach/YYYY-MM-DD_{slug}.txt

Template (agent-facing, <100 words):
"""
Hi,

I'd like to arrange a viewing of [LISTING TITLE / ADDRESS].

I'm [YOUR_NAME], [YOUR_AGE], [YOUR_NATIONALITY], a [YOUR_PROFESSION] at [YOUR_EMPLOYER_TYPE] on a permanent contract. Single, non-smoker, clean and respectful.

References and proof of funds available on request. Looking to move in [MOVE_IN_DATE]. Available for viewings [FLAT_OUTREACH_AVAILABILITY].

Best,
[YOUR_NAME]
"""

Insert ONE sentence of personalisation between the greeting and the self-intro if (and only if) you have a specific, confident detail from the listing (e.g. "The top-floor dual-aspect living room caught my eye" / "I like that it's on a quiet residential street just off Upper Street"). Don't fabricate. If unsure, omit the personalisation sentence.

## SUCCESS CRITERIA

- Flats tab updated, Meta tab stamped (B2 for local, B3 for remote)
- No listings added that violate hard filters
- No duplicates created (URL dedup works)
- HIGH tier caps respected for size-unknown 1-beds
- Outreach .txt files saved for HIGH priority
- Email sent (always, even if zero new)
```

---

## Notes on the placeholders

| Placeholder | Example value | Source |
|---|---|---|
| `[YOUR_NAME]` | Raphael | config.md |
| `[YOUR_AGE]` | 36 | config.md |
| `[YOUR_NATIONALITY]` | Swiss | config.md |
| `[YOUR_PROFESSION]` | computer engineer | config.md |
| `[YOUR_EMPLOYER_TYPE]` | an AI/gaming startup | config.md |
| `[MOVE_IN_DATE]` | early June 2026 | config.md |
| `[FLAT_*]` | see config.example.md flats mode section | config.md |
| `[FLAT_TRACKER_SHEET_ID]` | Google Sheet URL ID | config.md |
| `[REGION_CODE]` | Rightmove region identifier for London (`REGION%5E87490`) or tighter (per area); leave as-is if unsure | hardcode in skill |
| `[AREA_SLUG]` | zoopla/spareroom URL slug for area (lowercase, underscores) | derived from FLAT_PRIMARY_AREAS |
| `[AREAS_CSV]` | comma-separated areas for OpenRent's `term=` | derived from FLAT_PRIMARY_AREAS |
````

- [ ] **Step 2: Verify the skill file is complete**

Run: confirm `skill-flats.md` contains the strings `HARD FILTERS`, `SIZE RESOLUTION`, `SCORING`, `REMOTE FALLBACK LOGIC`, `OUTREACH`.

- [ ] **Step 3: Commit**

```bash
git add skill-flats.md
git commit -m "skill: add flats variant for 1-2 bed whole-unit rentals

Introduces skill-flats.md as a parallel to skill.md. Adds scoring
model (top floor, new build, calm, bathtub, light, wooden floor),
size resolution rules (including permissive 1-bed inference capped
at MEDIUM), Google Sheet tracker wiring, and remote/local mode
detection for the fallback trigger."
```

---

### Task 4: Update `README.md` with Variants section + scheduling docs

**Files:**
- Modify: `README.md` (add Variants section near the top; replace the current "Schedule it" subsection with local + remote setup)

- [ ] **Step 1: Insert a `## Variants` section immediately after the "Example output" section**

Find the line `---` that closes the "Example output" section (currently around line 12 in the file). After that line, insert:

```markdown
## Variants

This repo ships **two** skill variants. Pick one via `HUNT_MODE` in your `config.md`.

| Variant | Skill file | Tracker | Who should use it |
|---|---|---|---|
| **Rooms** (default) | [`skill.md`](skill.md) | Local xlsx (`openpyxl`) | Renting a room in a shared 2–3 bed flat; SpareRoom-heavy |
| **Flats** | [`skill-flats.md`](skill-flats.md) | Google Sheet (Drive MCP) | Renting a whole 1–2 bed flat; scoring on top-floor / new-build / calm / bathtub / light / wooden floor |

The two variants are independent — they share the same `config.example.md` template and outreach folder convention, nothing else.

---

```

- [ ] **Step 2: Replace the existing `### 4. Schedule it` subsection**

Find the current block starting with `### 4. Schedule it` (around line 83) and ending just before `---` that introduces `## How the search works`. Replace the whole block with:

```markdown
### 4. Schedule it

**Rooms variant** (local xlsx, any single schedule works):

```
/schedule 0 9,18 * * * Run the London property hunt skill
```

**Flats variant** (local cron + remote fallback — recommended so missed days are self-healing):

1. **Local cron** (10:00 London time, uses Claude-in-Chrome for full fidelity):

   ```bash
   crontab -e
   # add:
   0 10 * * * cd ~/hunt && /usr/local/bin/claude "Run the London flat hunt skill" >> ~/hunt/cron.log 2>&1
   ```

2. **Remote fallback trigger** (15:00 London time, no-ops if local already ran):

   In Claude Code:
   ```
   /schedule 0 15 * * * Run the London flat hunt skill (remote fallback mode)
   ```

   The skill reads `Meta!B2` on the tracker sheet and exits silently if the local run already stamped today.

3. **Weekly health check** (optional; Mondays 09:00):

   ```
   /schedule 0 9 * * 1 Check Meta!B2 and Meta!B3 on the flat hunt sheet; if either is more than 2 days stale, send a one-line status email.
   ```

```

- [ ] **Step 3: Verify the README changes**

Run: confirm `README.md` now contains the strings `## Variants`, `Flats variant`, `Meta!B2`, and still contains the original `## How the search works` section afterwards.

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs(readme): add Variants section and scheduling docs

Documents the rooms vs flats split up-front and adds the local +
remote fallback cron pattern for the flats variant."
```

---

### Task 5: Final Phase 1 verification

- [ ] **Step 1: List all files changed in this phase**

Run: `git log --since="1 hour ago" --name-only --format="%h %s"` and confirm four commits are present: config.example.md, tracker/flats-schema.md, skill-flats.md, README.md.

- [ ] **Step 2: Confirm no stray files**

Run: `git status` — should be clean (no untracked, no modified).

- [ ] **Step 3: Re-read `config.example.md` + `skill-flats.md` for consistency**

Cross-check that every `[FLAT_*]` placeholder used in `skill-flats.md` exists as a field in `config.example.md`'s Flats mode section. Any mismatch → fix and add an amendment commit.

---

## Phase 2 — Raphael's personal setup (user-executed)

> These tasks are performed by the user (Raphael) on his own laptop and Google/Anthropic accounts. An agent cannot complete them autonomously — they involve interactive OAuth, browser sign-ins, and local machine access. Steps are included so the work is traceable.

### Task 6: Create the Google Sheet tracker

**Files (external):**
- Create: Google Sheet titled "London Flat Hunt" in Raphael's Google Drive

- [ ] **Step 1: Create the sheet**

Go to [sheets.new](https://sheets.new) while signed in as `rvonaar@gmail.com`. Title: `London Flat Hunt`.

- [ ] **Step 2: Create the `Flats` tab and populate headers**

Rename the default `Sheet1` to `Flats`. Paste these 23 headers into row 1 (A1:W1), in order:

```
URL	Platform	Found On	Title	Area	Price	Beds	Size	Size Source	Furnished	Available From	Floor	Bathtub	New/Renovated	Calm	Light	Wooden Floor	Score	Tier	Status	Reason	Needs-verify	Notes
```

Apply: bold header row, freeze row 1, enable filter on row 1.

- [ ] **Step 3: Create the `Meta` tab**

Click the `+` at the bottom, rename to `Meta`. Fill:

| Cell | Value |
|---|---|
| A1 | `Key` |
| B1 | `Value` |
| A2 | `last_local_run` |
| B2 | (empty) |
| A3 | `last_remote_run` |
| B3 | (empty) |
| A4 | `schema_version` |
| B4 | `1` |

- [ ] **Step 4: Copy the Sheet ID**

From the URL `https://docs.google.com/spreadsheets/d/<ID>/edit`, copy `<ID>` — save for Task 7.

---

### Task 7: Create Raphael's `config.md`

**Files:**
- Create: `~/hunt/config.md` (local, gitignored)

- [ ] **Step 1: Create the hunt directory**

```bash
mkdir -p ~/hunt/outreach
```

- [ ] **Step 2: Copy the template and fill in**

```bash
cp ~/dev/london-flat-hunt/london-property-hunt-public/config.example.md ~/hunt/config.md
```

- [ ] **Step 3: Edit `~/hunt/config.md`**

Set:

```
HUNT_MODE=flats

YOUR_NAME=Raphael
YOUR_AGE=36
YOUR_NATIONALITY=Swiss
YOUR_PROFESSION=computer engineer
YOUR_EMPLOYER_TYPE=an AI/gaming startup
YOUR_PROFILE_DESCRIPTION=single, non-smoker, clean, respectful, quiet, permanent contract
YOUR_PROFILE_SUMMARY=Swiss computer engineer, permanent contract, single and quiet

YOUR_WORK_POSTCODE=King's Cross area
YOUR_EMAIL=rvonaar@gmail.com
GMAIL_ACCOUNT_INDEX=0
MOVE_IN_DATE=early June 2026
YOUR_HUNT_DIR=~/hunt

# Flats mode
FLAT_BEDROOMS_MIN=1
FLAT_BEDROOMS_MAX=2
FLAT_SIZE_FLOOR_M2=60
FLAT_FURNISHED_PREFERENCE=unfurnished
FLAT_BUDGET_SWEET_MAX=3200
FLAT_BUDGET_HARD_CAP=3500
FLAT_STRETCH_PENALTY=1
FLAT_PRIMARY_AREAS=Islington, Angel, Camden Town, Kentish Town, De Beauvoir Town, Highbury, Canonbury, Clerkenwell
FLAT_SECONDARY_AREAS=Tufnell Park, Holloway, Bloomsbury, Russell Square, Barbican, Finsbury Park, London Bridge, Bermondsey
FLAT_WEIGHT_TOP_FLOOR=3
FLAT_WEIGHT_NEW_BUILD=3
FLAT_WEIGHT_CALM=2
FLAT_WEIGHT_BATHTUB=1
FLAT_WEIGHT_LIGHT=1
FLAT_WEIGHT_WOODEN_FLOOR=1
FLAT_TIER_HIGH_THRESHOLD=7
FLAT_TIER_MEDIUM_THRESHOLD=4
FLAT_TRACKER_SHEET_ID=<paste Sheet ID from Task 6 Step 4>
FLAT_TRACKER_FLATS_TAB=Flats
FLAT_TRACKER_META_TAB=Meta
FLAT_OUTREACH_STYLE=agent
FLAT_OUTREACH_AVAILABILITY=weekday evenings and weekends
```

- [ ] **Step 4: Verify `config.md` is gitignored**

```bash
cd ~/dev/london-flat-hunt/london-property-hunt-public
grep -E '^config\.md$|^\*?config\.md' .gitignore
```

Expected: a match. If no match, add `config.md` to `.gitignore` and commit.

---

### Task 8: Authenticate Gmail and Drive MCPs

- [ ] **Step 1: Authenticate Gmail MCP for rvonaar@gmail.com**

In Claude Code, invoke:

```
mcp__claude_ai_Gmail__authenticate
```

Follow the OAuth flow in the browser, sign in as `rvonaar@gmail.com`, grant access.

- [ ] **Step 2: Confirm Gmail auth**

Send a test draft:

```
Create a Gmail draft to rvonaar@gmail.com with subject "MCP test" and body "If you see this, Gmail MCP is wired up."
```

Check the drafts folder in the browser.

- [ ] **Step 3: Authenticate Google Drive MCP**

Invoke:

```
mcp__claude_ai_Google_Drive__authenticate
```

Sign in as `rvonaar@gmail.com`, grant access to Drive.

- [ ] **Step 4: Confirm Drive auth**

Ask Claude to read cell `Meta!B4` from the `London Flat Hunt` sheet. Expected: `1`.

---

### Task 9: Confirm Claude-in-Chrome is installed + signed in

- [ ] **Step 1: Install the extension (if missing)**

Visit the Claude-in-Chrome extension page in Chrome and install.

- [ ] **Step 2: Sign in**

Open the extension, sign in with the same Anthropic account used by Claude Code.

- [ ] **Step 3: Smoke test**

Ask Claude Code: "Use Claude-in-Chrome to open `https://www.spareroom.co.uk/flats-to-rent/london/hackney` and return the page title." Expected: a page title string.

If the tool is unavailable → the skill will fall through to WebFetch mode. Fix before relying on local cron.

---

## Phase 3 — Scheduling (user-executed)

### Task 10: Add local crontab entry

- [ ] **Step 1: Find the claude CLI path**

```bash
which claude
```

Note the path (e.g. `/usr/local/bin/claude` or `~/.local/bin/claude`).

- [ ] **Step 2: Edit crontab**

```bash
crontab -e
```

Append (replace `<CLAUDE_PATH>` with the path from Step 1 and `<USER>` with your home dir):

```
0 10 * * * cd /home/<USER>/hunt && <CLAUDE_PATH> "Run the London flat hunt skill (local mode)" >> /home/<USER>/hunt/cron.log 2>&1
```

- [ ] **Step 3: Verify crontab installed**

```bash
crontab -l | grep "London flat hunt"
```

Expected: the entry above.

---

### Task 11: Create remote fallback trigger

- [ ] **Step 1: Create the trigger via `/schedule`**

In Claude Code:

```
/schedule 0 15 * * * Run the London flat hunt skill (remote fallback mode) — check Meta!B2 first and exit silently if already stamped today
```

- [ ] **Step 2: List triggers to confirm**

```
/schedule list
```

Expected: the new trigger appears with next run at the upcoming 15:00 London time.

---

### Task 12: Create weekly health-check trigger (optional)

- [ ] **Step 1: Create the trigger**

```
/schedule 0 9 * * 1 Read Meta!B2 and Meta!B3 from the London Flat Hunt sheet. If either is more than 2 calendar days stale (vs today), email rvonaar@gmail.com a one-line status.
```

- [ ] **Step 2: Confirm**

```
/schedule list
```

---

## Phase 4 — Smoke test and calibration

### Task 13: Manual end-to-end invocation

- [ ] **Step 1: Invoke the skill locally**

```bash
cd ~/hunt && claude "Run the London flat hunt skill (local mode)"
```

- [ ] **Step 2: Verify outputs**

Check:
- [ ] `Flats` tab in the Google Sheet has new rows with correct tier colouring (HIGH=green, MEDIUM=yellow, LOW=red)
- [ ] `Meta!B2` is stamped with today's date
- [ ] Gmail drafts folder has a new email subject `🏠 London Flat Hunt — <today>`
- [ ] Email body renders correctly on phone (open on phone to check)
- [ ] `~/hunt/outreach/<today>_*.txt` files exist for every HIGH listing
- [ ] Zero listings appear with stated size < 60 m² or beds outside 1–2

- [ ] **Step 3: Inspect scoring on 3 HIGH listings**

For 3 random HIGH rows, manually verify the `Reason` column matches the listing (open URL, sanity-check top-floor/new-build/calm/light/bathtub/wooden-floor claims).

- [ ] **Step 4: Adjust thresholds if needed**

If >20 HIGH or <1 HIGH, reconsider weights or thresholds in `config.md` (no skill change required — it re-reads config on every run).

---

### Task 14: Verify remote fallback no-ops when local succeeded

- [ ] **Step 1: Wait for next 10:00 local cron**

Let the scheduled local cron run naturally. Check `~/hunt/cron.log` afterwards for a successful run line.

- [ ] **Step 2: Wait for next 15:00 remote trigger**

Check: `Meta!B2` still equals today's date; `Meta!B3` is **not** updated; no second email was sent.

Expected: remote trigger saw `B2 == today` and exited silently.

---

### Task 15: Break-glass test — remote fallback runs when local missed

- [ ] **Step 1: Simulate a missed local run**

In the Google Sheet, clear `Meta!B2` (delete the date).

- [ ] **Step 2: Manually fire the remote trigger**

In Claude Code:

```
/schedule run <trigger-id>
```

(Get `<trigger-id>` from `/schedule list`.)

- [ ] **Step 3: Verify remote execution**

Check:
- [ ] `Meta!B3` is stamped with today's date (remote ran)
- [ ] Gmail drafts has an email with header "Run source: Remote fallback"
- [ ] New rows appear on `Flats` (dedup skipped any that already existed from the real local run — should only add genuinely new ones, or none if nothing new since last real run)

- [ ] **Step 4: Restore `Meta!B2`**

Re-enter today's date in `Meta!B2` so the next real local cron run can proceed normally.

---

## Done

When all of the above pass:

- Repo shipped with the flats variant (Phase 1)
- Raphael's personal hunt is live and self-healing (Phases 2–4)
- First real HIGH listings should appear within 1–3 days
- Expected steady-state: 0–5 new listings/day during active season, ~1 HIGH per day, 3–8 HIGH/week

### Known limitations to revisit after 2 weeks of real runs

- If WebFetch frequently fails on Rightmove/Zoopla in remote mode → upgrade to Browserbase MCP (spec §14, deferred item)
- If size-unknown 1-beds flood the tracker → tighten `inferred-1bed-photos` rule or re-adopt strict skip
- If HIGH hit rate < 1/week → relax area list or bump hard cap
