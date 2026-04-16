# Flat Hunt Variant — Design Spec

**Date:** 2026-04-16
**Author:** Raphael (rvonaar@gmail.com)
**Status:** Draft — pending user review
**Target repo:** `london-property-hunt-public`

---

## 1. Context

The existing `skill.md` in this repo automates a London **shared-room** hunt: rooms in 2–3 bed flats, SpareRoom-heavy, flatmate-age filters, outreach to existing tenants. It's a public template — adaptable via `config.md`.

This spec defines a second variant for **whole-unit 1–2 bed flat rentals** with a different user profile, different criteria, and a richer scoring model. The two variants coexist in the repo; users pick one via `HUNT_MODE=rooms|flats` in their config.

## 2. Goals

- Daily automated search for 1–2 bed unfurnished flats within ~20 min Tube of King's Cross.
- Rank listings on quality-of-life signals that don't exist as portal filters (top floor, new build, calm, bathtub, light).
- Deliver a scored daily email with ready-to-send estate-agent outreach for HIGH-priority listings.
- Run reliably via a **local cron with a remote-trigger fallback**, so missed days are self-healing without requiring the laptop to be awake.
- Extend the public repo so other hunters can reuse the flats variant (not just fork it privately).

## 3. Non-goals

- Buying, not just renting (out of scope — this variant is rental-only).
- Rooms in shared flats (covered by existing `skill.md`).
- Hosted browser automation (Browserbase/Hyperbrowser) — deferred; revisit only if WebFetch fidelity drops.
- Auto-sending outreach messages or booking viewings — drafts only, human in the loop.

## 4. User profile (Raphael)

- 36yo Swiss male, computer engineer at an AI/gaming startup, permanent contract.
- Single, non-smoker, clean, respectful, quiet.
- Office near King's Cross (exact postcode TBC — not needed for the skill).
- Move-in: early June 2026.

## 5. Hard filters (auto-skip)

A listing is rejected outright if **any** of the following is true:

- Not 1 or 2 bedrooms
- Rent > £3,500 pcm
- Outside the primary + secondary area list (§7)
- Stated size < 60 m² (size-unknown handling per §6)
- Obviously dated or unrenovated (tired photos, old fittings, yellowing kitchens/bathrooms) — judgment call by the LLM on photos

## 6. Size (m²) resolution

UK listings often omit floor area. Resolution priority:

1. **Stated in listing text** → use directly.
2. **Stated on floorplan image** → extract visually from the floorplan.
3. **2-bed, size unknown** → assume ≥60 m², accept (pass the hard filter).
4. **1-bed, size unknown (no text, no floorplan)** → include, but **cap tier at MEDIUM** and populate `Needs-verify: size`. Rationale: period conversions and mansion-block flats in Clerkenwell / Islington / Bloomsbury / Barbican frequently omit m² but are generously sized. Small 1-beds won't reach HIGH anyway (they lack top-floor / new-build / dual-aspect signals), so they self-filter via scoring.

Record the chosen source in the tracker column `Size Source` (one of: `stated-text`, `floorplan`, `inferred-2bed`, `inferred-1bed-photos`, `unknown`).

For rule 4, set `Size Source` to `inferred-1bed-photos` if Claude's photo judgment suggests ≥60 m² living space (large living room + separate kitchen + proper double bedroom); otherwise `unknown`. Either way, the listing enters the tracker with the `Needs-verify: size` flag and a MEDIUM tier cap.

## 7. Areas

All areas chosen for ≤20 min Tube to King's Cross St Pancras + acceptable quality/price profile.

**Primary (+1 tier bump):**
- Islington / Angel
- Camden Town
- Kentish Town
- De Beauvoir Town
- Highbury / Canonbury
- Clerkenwell

**Secondary (−1 tier bump):**
- Tufnell Park
- Holloway
- Bloomsbury / Russell Square
- Barbican
- Finsbury Park
- London Bridge
- Bermondsey

Listings outside these areas are rejected (hard filter).

## 8. Scoring model

Score is on a 0–11 scale, summed from signal weights and modifiers.

### Signal weights

| Signal | Weight | How it's detected |
|---|---|---|
| Top floor / no neighbor above (incl. penthouse, loft, top-floor attic) | **3** | Listing text keywords, floorplan annotation, photos (roof/skylight visible) |
| New build or recent refurb (≤~5y) | **3** | Listing text ("newly refurbished", "new development", "2022 refurb"), photos (modern fittings, new kitchen/bathroom) |
| Calm street (residential, not arterial/high-street/pub-adjacent) | **2** | Street name vs. known arterials, map check, listing copy ("quiet street") |
| Bathtub (not shower-only) | **1** | Photos of bathroom, floorplan, listing copy |
| Light / windows (dual aspect, floor-to-ceiling, south-facing, high floor) | **1** | Photos, listing copy, floor level |
| Wooden floor (real wood or engineered, not laminate/carpet/tile in living areas) | **1** | Photos of living room / bedrooms; listing copy ("hardwood floors", "engineered oak") |

### Modifiers

| Modifier | Effect |
|---|---|
| Primary area (§7) | +1 |
| Secondary area (§7) | −1 |
| Unfurnished or part-furnished | +0.5 |
| Rent in stretch band £3,001–£3,500 | **−1** |

### Tiers

| Tier | Score |
|---|---|
| HIGH | ≥ 7 |
| MEDIUM | 4 – 6.9 |
| LOW | < 4 |

### Per-listing output

Each scored listing records:
- `Score` (0–11, one decimal)
- `Tier` (HIGH / MEDIUM / LOW)
- `Reason` — one-line summary: `"top floor ✅, new build ✅, on Camden Rd ❌ (loud)"`
- `Needs-verify` — list of unresolved signals: `"calm (street unclear)"`, `"bathtub (no bathroom photo)"`, `"size (no floorplan)"`

## 9. Budget

- **Hard cap:** £3,500 pcm (any higher → hard filter, skip)
- **Sweet spot:** ≤ £3,200 (no penalty)
- **Stretch band:** £3,201 – £3,500 (−1 score penalty; surfaces gems but unlikely to reach HIGH unless quality is very strong)

All rent figures are PCM (per calendar month), bills excluded unless noted.

## 10. Platforms & search

All four platforms, filtered at URL level to max £3,500, 1–2 bed, furnished + unfurnished + part-furnished (no furnished filter at URL level — filter in scoring).

- **Rightmove** — primary; has floorplans and photos, critical for m² and scoring
- **Zoopla** — secondary; overlaps Rightmove heavily but catches ~10% unique
- **OpenRent** — landlord-direct; smaller inventory, friendlier to WebFetch
- **SpareRoom (flats-to-rent section)** — low volume for whole units but occasionally has gems; keep as catch-all

Search URLs are generated from the area list in `config.md` and injected into the skill prompt at runtime. See `config.example.md` for the templates.

## 11. Tracker

**File:** Google Sheet in Google Drive (not a local xlsx — needed so both local and remote runs can R/W the same state).

**Why Google Sheet:** remote trigger cannot reach local files; Drive MCP is available remotely (`mcp__claude_ai_Google_Drive__*`). Local cron uses Drive-for-Desktop-synced local path *or* Drive API directly — both valid.

**Sheets in the workbook:**
- `Flats` — all 1–2 bed flat listings (this variant)
- (existing) `Listings` — rooms (for shared-room variant; untouched)
- (existing) `Studios & 1-Beds` — previous skill's 1-bed sheet; **deprecated for this user**, new flat hunt uses the `Flats` sheet instead

**`Flats` sheet columns:**

| # | Column | Type | Notes |
|---|---|---|---|
| 1 | URL | text | Dedup key |
| 2 | Platform | enum | Rightmove / Zoopla / OpenRent / SpareRoom |
| 3 | Found On | date | First-seen date |
| 4 | Title | text | Listing title |
| 5 | Area | text | From §7 list |
| 6 | Price | int | £ pcm |
| 7 | Beds | int | 1 or 2 |
| 8 | Size | number | m² |
| 9 | Size Source | enum | stated-text / floorplan / inferred-2bed |
| 10 | Furnished | enum | Furnished / Part / Unfurnished / Unknown |
| 11 | Available From | date | Stated availability |
| 12 | Floor | enum | Top / Mid / Ground / Unknown |
| 13 | Bathtub | enum | Yes / No / Unknown |
| 14 | New/Renovated | enum | Yes / No / Unknown |
| 15 | Calm | enum | Yes / No / Unknown |
| 16 | Light | enum | Good / Avg / Poor / Unknown |
| 17 | Wooden Floor | enum | Yes / No / Unknown |
| 18 | Score | number | 0–11 |
| 19 | Tier | enum | HIGH / MEDIUM / LOW |
| 20 | Status | enum | NEW 🔴 / CONTACTED / VIEWING / REJECTED / GONE |
| 21 | Reason | text | One-line scoring summary |
| 22 | Needs-verify | text | Comma-list of unresolved signals |
| 23 | Notes | text | Free-form |

**Row colouring:** HIGH = `E2EFDA` (green), MEDIUM = `FFFFC7` (yellow), LOW = `FCE4D6` (red) — same palette as existing skill.

**Dedup:** on URL; skip insert if URL already present. O(1) lookup.

## 12. Email format

Same structure as existing skill, adapted for flats. HTML via `gmail_create_draft`, sent to `rvonaar@gmail.com`.

**Subject:** `🏠 London Flat Hunt — {DATE} — {N} new listings (H:{n}/M:{n}/L:{n})`

**Sections:**
- **A) Header** — date, run source (local/remote), platform counts, tier totals
- **B) 🟢 HIGH New Today** — one card per listing, sorted by score desc. Each card: title-as-link, area, price, size, floor, furnished, score, one-line reason, needs-verify flags, green ready-to-send outreach block
- **C) 🟡 MEDIUM New Today** — condensed card, short outreach
- **D) ⚪ LOW / SKIP** — bullet list only
- **E) 🔁 Backlog** — up to 8 uncontacted HIGH from prior days (Status = NEW 🔴, Tier = HIGH, Found On ≠ today), with outreach cards
- **F) Stats footer** — totals, area breakdown, days-to-move-in countdown, next run time

**Always send**, even on zero new listings — confirms the pipeline ran.

## 13. Outreach

**Who is contacted:** estate agents (primary) or direct landlords (OpenRent). Not existing tenants.

**Template** (HIGH priority only, saved as `.txt` per listing + embedded in email card):

> Hi,
>
> I'd like to arrange a viewing of [LISTING TITLE / ADDRESS].
>
> I'm Raphael, 36, Swiss, a computer engineer at an AI/gaming startup on a permanent contract. Single, non-smoker, clean and respectful.
>
> References and proof of funds available on request. Looking to move in early June. Available for viewings [weekday evenings and weekends].
>
> Best,
> Raphael

Per-listing personalisation (one sentence referring to a specific feature: top floor, natural light, location) is inserted by the LLM when confidence is high.

Files saved to `$HUNT_DIR/outreach/YYYY-MM-DD_<slug>.txt`.

## 14. Scheduling

### Local cron (primary)

- **When:** daily at 10:00 London time
- **Where:** Raphael's laptop via `crontab` or `systemd` timer
- **How:** invokes `claude` CLI with the flats skill; has access to Claude-in-Chrome MCP for full-fidelity browser scraping (Rightmove/Zoopla photos, floorplans)
- **Success stamp:** writes today's date into `Meta!last_local_run` (a reserved cell on a `Meta` sheet in the same workbook)

### Remote trigger (fallback)

- **When:** daily at 15:00 London time (~4–5h after local, accounting for BST/GMT)
- **Where:** Anthropic-hosted remote trigger, created via `CronCreate`
- **How:** opens the Google Sheet, reads `Meta!last_local_run`:
  - If equal to today → no-op, exit silently
  - If not equal → runs the flats skill in **WebFetch mode** (no Claude-in-Chrome — still gets listing text and many photos; floorplan extraction may be degraded)
- **Success stamp:** writes today's date into `Meta!last_remote_run`

### Failure handling

- If both runs fail (laptop off + remote trigger error), no email is sent → Raphael sees gap in inbox → manual invocation.
- Monthly sanity: a separate lightweight remote trigger (weekly) reads both `Meta!last_local_run` and `Meta!last_remote_run`, emails a one-line status if either is more than 2 days stale.

### Why not hosted browser (Browserbase etc.)

Deferred. The local+remote-fallback gets ~95% of days with full fidelity at zero extra cost. Revisit if Rightmove or Zoopla start gating WebFetch such that remote-mode misses meaningful listings.

## 15. Repository changes

Option B confirmed — extend the public repo so the flats variant ships alongside the rooms variant.

### New files

- `skill-flats.md` — flats variant skill prompt, parallel to `skill.md`
- `tracker/flats-schema.md` — `Flats` sheet column spec (mirrors §11)

### Modified files

- `config.example.md`:
  - Add `HUNT_MODE=rooms|flats` at the top
  - Add flats-specific fields under a `## Flats mode` section: `UNIT_BEDROOMS=1-2`, `SIZE_FLOOR_M2=60`, `BUDGET_HARD_CAP`, `BUDGET_SWEET_MAX`, `BUDGET_STRETCH_MAX`, primary/secondary area lists, soft-criteria weights (optional override), `OUTREACH_TEMPLATE_STYLE=agent|landlord`
  - Preserve all existing fields for rooms mode (backwards-compatible)
- `README.md`:
  - Add a "Variants" section up top: rooms vs flats, one-line differences, pick one
  - Document local + remote scheduling setup
  - Link to `tracker/flats-schema.md`
- `.gitignore`: confirm `config.md` is ignored (already is)

### Unchanged files

- `skill.md`, `case-study.md`, `tracker/README.md` (rooms schema), `outreach/` structure

### User-local (gitignored)

- `config.md` — Raphael's values, per `config.example.md` template
- `~/hunt/` or equivalent (exact path in `config.md`)

## 16. Setup steps (for Raphael)

1. Create a Google Sheet titled `London Flat Hunt` in the target Google Drive; note the Sheet ID
2. Create `Flats` and `Meta` sheets inside the workbook; apply column headers from §11
3. Ensure Drive-for-Desktop is installed and the folder is synced locally (optional — needed only if local cron uses local path vs Drive API)
4. Copy `config.example.md` → `config.md`, fill in values (email, Sheet ID, areas, budget, move-in date)
5. Auth Gmail MCP against `rvonaar@gmail.com` (`mcp__claude_ai_Gmail__authenticate`)
6. Auth Google Drive MCP against the same account
7. Install Claude-in-Chrome extension and sign in (for local run)
8. Add `crontab` entry:
   ```
   0 10 * * * cd ~/hunt && claude "Run the London flat hunt skill"
   ```
9. Create remote fallback trigger via `CronCreate` (15:00 London time daily, reads `Meta!last_local_run` before running)
10. Manual test: invoke the skill once locally, verify email arrives and sheet is populated

## 17. Open questions / deferred

- Exact King's Cross office postcode — not needed for the skill, but relevant if commute-time scoring is added later.
- Whether to add a commute-distance signal (e.g. score boost for <10 min to KX vs 15–20 min). Deferred — current area list already encodes this coarsely.
- Migration path from xlsx to Google Sheet for the existing rooms variant — out of scope; rooms variant stays xlsx-local.
- Hosted browser (Browserbase) upgrade — deferred pending real-run evidence.

## 18. Success criteria

- Daily email arrives in `rvonaar@gmail.com` with correctly tiered listings.
- Zero listings with stated size < 60 m² appear (hard filter works).
- Zero 3+ bed or studio listings appear (hard filter works).
- Zero outside-area listings appear.
- HIGH tier listings consistently show score ≥ 7 with plausible one-line reasons.
- `Needs-verify` column is populated for size-inferred and photo-ambiguous cases.
- Over a 14-day window, missed-day rate < 10% (combined local + remote coverage).
- At least one HIGH listing per week on average (sanity check — if zero for 2 weeks, criteria are too strict).
