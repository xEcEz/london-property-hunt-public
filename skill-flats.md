# London Flat Hunt — Claude Code Skill (1–2 bed variant)

> **How to use:** Copy this file's contents into Claude Code as a skill, or run it directly.
> Before running, fill in your personal details in `config.md` (copy from `config.example.md`, set `HUNT_MODE=flats`).
> All `[PLACEHOLDER]` values below are read from your config file.

---

## Skill prompt (paste into Claude Code)

```
You are running [YOUR_NAME]'s London flat hunt (1–2 bed whole-unit rentals). Do this automatically — no user present. Complete all steps fully.

## RUNTIME MODE

Always run alert ingestion first (§ALERT INGESTION below). It works in both local and remote modes.

After alerts are processed, conditionally run scraping:
- Local (Claude-in-Chrome MCP available) → use browser MCP for Rightmove / Zoopla / SpareRoom / OpenRent search pages (full fidelity: photos, floorplans).
- Remote (WebFetch only, no Claude-in-Chrome) → SKIP scraping. Alert ingestion is your only source.

Detect runtime mode by checking if Claude-in-Chrome MCP tools are available. If not → WebFetch mode, skip scraping.

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
- Rent above £[FLAT_BUDGET_SWEET_MAX] and up to £[FLAT_BUDGET_HARD_CAP]: −[FLAT_STRETCH_PENALTY]

Tiers:
- HIGH: score ≥ [FLAT_TIER_HIGH_THRESHOLD]
- MEDIUM: [FLAT_TIER_MEDIUM_THRESHOLD] ≤ score < [FLAT_TIER_HIGH_THRESHOLD]
- LOW: score < [FLAT_TIER_MEDIUM_THRESHOLD]

**Cap:** size-unknown 1-beds (rule 4 above) → tier cannot exceed MEDIUM even if score ≥ HIGH threshold.

## ALERT INGESTION (runs first, both local and remote)

Read portal saved-search alert emails from Raphael's Gmail and convert them into Flats rows.

### Gmail query

Call `mcp__claude_ai_Gmail__search_threads` with:

(from:(@rightmove.co.uk) OR from:(@zoopla.co.uk) OR from:(@spareroom.co.uk) OR from:(@openrent.co.uk)) AND newer_than:2d AND -label:hunt-processed

Rationale: sender-domain filter catches portal transactional mail; `-label:hunt-processed` avoids reprocessing; `newer_than:2d` catches backlog after weekend laptop-off periods.

### For each matched thread

1. Fetch full body via `mcp__claude_ai_Gmail__get_thread`.
2. Extract listing URLs by matching portal patterns:
   - Rightmove: `https://www.rightmove.co.uk/properties/<id>`
   - Zoopla: `https://www.zoopla.co.uk/to-rent/details/<id>/`
   - SpareRoom: `https://www.spareroom.co.uk/flatshare/flatshare_detail.pl?flatshare_id=<id>` or `/flats-to-rent/...`
   - OpenRent: `https://www.openrent.co.uk/property-to-rent/...` with numeric id
3. De-redirect tracking wrappers (e.g. `rightmove.co.uk/redirect?...`) by following to the final URL.
4. Dedup within the email (alerts sometimes list the same listing twice).

### For each extracted URL

1. Query Notion Flats data source (id `[FLAT_TRACKER_FLATS_DATA_SOURCE_ID]`) for `URL == <candidate>`. If any match, skip this URL.
2. Otherwise fetch the individual listing page:
   - Local mode: Claude-in-Chrome
   - Remote mode: WebFetch
3. Apply HARD FILTERS (see section above) to the listing. If it violates any filter (beds outside 1–2, rent > hard cap, area not in primary ∪ secondary, stated size < 60 m², obviously dated/unrenovated), do NOT create a Notion row — log the skip and move to the next URL.
4. If the listing passes hard filters and page fetch succeeded → extract all available signals per § SCORING, score, create Notion row with `Source=email-alert`.
5. If page fetch failed but the alert email content alone did not trigger a hard-filter rejection → create a minimal Notion row using alert-email data (title, price, beds, URL). Set `Size Source=unknown`, `Needs-verify: listing page unreachable`, `Source=email-alert`. Skip scoring (leave `Score` and `Tier` empty).

### Label-after-handling

After each successfully-processed thread (whether all URLs led to new rows or all were dedup-skipped), call `mcp__claude_ai_Gmail__label_thread` with label `hunt-processed`. On first run, if the label doesn't exist, create it via `mcp__claude_ai_Gmail__create_label`.

Errors during labeling are non-fatal — URL dedup handles the duplicate-processing case.

### When to skip

If an alert email has zero extractable URLs (e.g. marketing template), do NOT label it. A future run may retry if the email content changes. Log the skip but don't fail.

## SCRAPING (local mode only)

**Skip this section entirely in remote WebFetch-only mode** — rely on alert ingestion instead.

Search all four. URLs assembled from config areas + budget. Max price = £[FLAT_BUDGET_HARD_CAP].

- Rightmove: https://www.rightmove.co.uk/property-to-rent/find.html?searchType=RENT&locationIdentifier=[REGION_CODE]&minBedrooms=[FLAT_BEDROOMS_MIN]&maxBedrooms=[FLAT_BEDROOMS_MAX]&maxPrice=[FLAT_BUDGET_HARD_CAP]&propertyTypes=flat&includeLetAgreed=false&sortType=6
- Zoopla: https://www.zoopla.co.uk/to-rent/flats/[AREA_SLUG]/?beds_min=[FLAT_BEDROOMS_MIN]&beds_max=[FLAT_BEDROOMS_MAX]&price_frequency=per_month&price_max=[FLAT_BUDGET_HARD_CAP]&results_sort=newest_listings
- OpenRent: https://www.openrent.co.uk/properties-to-rent/london?term=[AREAS_CSV]&prices_max=[FLAT_BUDGET_HARD_CAP]&bedrooms_min=[FLAT_BEDROOMS_MIN]&bedrooms_max=[FLAT_BEDROOMS_MAX]&isLive=true
- SpareRoom: https://www.spareroom.co.uk/flats-to-rent/london/[AREA_SLUG]?max_rent=[FLAT_BUDGET_HARD_CAP]&min_beds=[FLAT_BEDROOMS_MIN]&max_beds=[FLAT_BEDROOMS_MAX]&sort=posted_date&mode=list

## TRACKER

Notion workspace with two databases under parent page [FLAT_TRACKER_NOTION_PARENT_PAGE_ID]:

- **Flats database** — data source id [FLAT_TRACKER_FLATS_DATA_SOURCE_ID]. One page per listing.
- **Meta database** — data source id [FLAT_TRACKER_META_DATA_SOURCE_ID]. Key/Value rows for `last_local_run`, `last_remote_run`, `schema_version`.

Properties on Flats (see tracker/flats-schema.md for full types): Title (title), URL, Platform, Found On, Area, Price, Beds, Size, Size Source, Furnished, Available From, Floor, Bathtub, New/Renovated, Calm, Light, Wooden Floor, Score, Tier, Status, Reason, Needs-verify, Notes, Source.

Dedup: before inserting, query the Flats data source filtered on `URL == <candidate URL>`. Skip if any match.

Set `Source` on every new row:
- `email-alert` when the row came from the Alert Ingestion path
- `scraped-local` when from scraping in local mode (Claude-in-Chrome)
- `scraped-remote` when from scraping in remote WebFetch mode (this should be rare — remote mode skips scraping in the new architecture)

If dedup catches a URL, do NOT overwrite `Source` on the existing row.

New rows: Status = `NEW`, Found On = today (YYYY-MM-DD, Europe/Zurich). Use `notion-create-pages` with `parent.type = "data_source_id"`.

Colour coding is automatic — the Tier select property has green/yellow/red options for HIGH/MEDIUM/LOW.

## REMOTE FALLBACK LOGIC (WebFetch mode only)

Alert ingestion (§ALERT INGESTION above) always runs first, regardless of mode. This REMOTE FALLBACK LOGIC section governs **only** the Meta-stamp check, the email-creation step, and whether scraping runs — it does NOT gate alert ingestion.

In WebFetch mode, after alert ingestion completes:
1. Query the Meta data source for `Key == "last_local_run"`. Read its Value.
2. If Value equals today's YYYY-MM-DD (Europe/Zurich) → exit silently. Do not email. Do not scrape. Do not update Meta.
3. Else → proceed with full search. On success, find the `last_remote_run` row in Meta and call `notion-update-page` to set its Value to today's YYYY-MM-DD.

In local mode (Claude-in-Chrome available): on success, update the `last_local_run` row's Value to today's YYYY-MM-DD the same way.

## EMAIL — SELF-CONTAINED, PHONE-READY

Use `mcp__claude_ai_Gmail__create_draft` (contentType: text/html), To: [YOUR_EMAIL].

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

Always create a draft, even on zero new listings — confirms pipeline ran.

**Local mode:** after creating the draft, navigate via Claude-in-Chrome to `https://mail.google.com/mail/u/[GMAIL_ACCOUNT_INDEX]/#drafts`, open the latest draft, click Send.

**Remote mode:** the Gmail MCP in the remote environment is draft-only (no `gmail_send_message`). Leave the email as a draft — Raphael opens `https://mail.google.com/mail/u/0/#drafts` and clicks Send manually. Do NOT attempt to send via a second mechanism.

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

- Alert emails queried for the target sender domains in the last 2 days
- Processed alert threads labeled `hunt-processed` in Gmail
- Flats database has new pages for every listing that passed hard filters and survived URL dedup, with `Source` correctly set
- Meta database: `last_local_run` stamped on local runs; `last_remote_run` stamped on remote runs
- No listings added that violate hard filters
- No duplicate URLs in Notion (dedup holds across both ingestion paths)
- HIGH tier caps respected for size-unknown 1-beds
- Outreach .txt files saved for HIGH priority (local mode only)
- Email draft created (local mode also clicks Send; remote mode leaves as draft for the Apps Script auto-sender)
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
| `[FLAT_TRACKER_NOTION_PARENT_PAGE_ID]` | Notion page ID (32 hex, no dashes or with dashes) of the parent "London Flat Hunt" page | config.md |
| `[FLAT_TRACKER_FLATS_DATA_SOURCE_ID]` | Notion data source ID for the Flats database | config.md |
| `[FLAT_TRACKER_META_DATA_SOURCE_ID]` | Notion data source ID for the Meta database | config.md |
| `[REGION_CODE]` | Rightmove region identifier for London (`REGION%5E87490`) or tighter (per area); leave as-is if unsure | hardcode in skill |
| `[AREA_SLUG]` | zoopla/spareroom URL slug for area (lowercase, underscores) | derived from FLAT_PRIMARY_AREAS |
| `[AREAS_CSV]` | comma-separated areas for OpenRent's `term=` | derived from FLAT_PRIMARY_AREAS |
