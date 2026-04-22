# London Flat Hunt — Claude Code Skill (1–2 bed variant)

> **How to use:** Copy this file's contents into Claude Code as a skill, or run it directly.
> Before running, fill in your personal details in `config.md` (copy from `config.example.md`, set `HUNT_MODE=flats`).
> All `[PLACEHOLDER]` values below are read from your config file.

---

## Skill prompt (paste into Claude Code)

```
You are running [YOUR_NAME]'s London flat hunt (1–2 bed whole-unit rentals). Do this automatically — no user present. Complete all steps fully.

## RUNTIME MODE

Two ingestion paths — alert ingestion (§ALERT INGESTION) always runs first in every mode; scraping (§SCRAPING) runs only when the runtime can reach Playwright.

**Local mode (laptop cron, residential IP):**
- Run alert ingestion.
- Probe Playwright via `mcp__playwright__navigate` to `https://example.com`. If it responds within 30s with no error, run § SCRAPING for all 4 portals. If it fails, log "Playwright unavailable — scraping skipped" and continue.

**Remote mode (Anthropic-hosted trigger, datacenter IP):**
- Run alert ingestion.
- **SKIP § SCRAPING entirely.** Remote environments have no Playwright MCP; WebFetch from datacenter IPs is gated on Rightmove/Zoopla/SpareRoom search pages. Alert ingestion is the only remote input source.

Detect remote mode by the absence of Playwright MCP tools (probe as above and catch the "server not available" error pattern).

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

(from:(rightmove.co.uk) OR from:(zoopla.co.uk) OR from:(spareroom.co.uk) OR from:(openrent.co.uk) OR from:(openrent.com)) AND newer_than:2d AND -label:hunt-processed

Rationale: sender-domain substring filter catches portal transactional mail including subdomains (e.g. `news@email.zoopla.co.uk`, `property-alerts@rightmove.co.uk`); `-label:hunt-processed` skips threads already handled; `newer_than:2d` is a safety net in case labeling fails. The Notion URL column is still the canonical dedup key — labels are just an efficiency optimization.

### For each matched thread

1. Fetch full body via `mcp__claude_ai_Gmail__get_thread`.
2. **Search both plaintext AND HTML bodies** for listing URLs. Zoopla in particular only puts per-listing links in the HTML part — the plaintext usually contains just the homepage root and marketing copy. If the tool returns the thread with both `body` / `plaintextBody` and `htmlBody` / `rawBody`, check all of them. If only plaintext is exposed, query Gmail again with `format=full` or `format=raw` to get the MIME parts.
3. Extract listing URLs by matching portal patterns:
   - Rightmove: `https://www.rightmove.co.uk/properties/<id>`
   - Zoopla: `https://www.zoopla.co.uk/to-rent/details/<id>/`
   - SpareRoom: `https://www.spareroom.co.uk/flatshare/flatshare_detail.pl?flatshare_id=<id>` or `/flats-to-rent/...`
   - OpenRent: `https://www.openrent.co.uk/property-to-rent/...` with numeric id
4. **Unwrap tracker-wrapped links.** Portal alert emails wrap their listing URLs in click-tracking redirects that look like:
   - Zoopla: `https://comms-2.zoopla.co.uk/l/?link=<URL-encoded target>` or `https://links.zoopla.co.uk/...`
   - Rightmove: `https://www.rightmove.co.uk/redirect?...` or `https://email.rightmove.co.uk/q/...`
   - OpenRent: `https://openrent.us.list-manage.com/track/click?...`
   - SpareRoom: `https://click.spareroom.co.uk/...`

   Two unwrap strategies, try in order:
   (a) **Parse the query string.** Look for a `link=`, `url=`, `u=`, `target=`, or `destination=` parameter. URL-decode it. If it matches a portal listing pattern (step 3), use it directly — no HTTP fetch needed.
   (b) **Follow the redirect.** If the wrapper's query string doesn't contain the target URL, HEAD or GET the tracker URL and follow the `Location:` redirect chain until you land on a portal listing URL.

   Only use strategy (b) when (a) can't extract the target — it costs a network hop and may be rate-limited.
5. Dedup within the email (alerts sometimes list the same listing twice or link to the same listing via different tracker URLs).

### For each extracted URL

1. Query Notion Flats data source (id `[FLAT_TRACKER_FLATS_DATA_SOURCE_ID]`) for `URL == <candidate>`. If any match, skip this URL.
2. Otherwise fetch the individual listing page:
   - Local mode: Claude-in-Chrome
   - Remote mode: WebFetch
3. Apply HARD FILTERS (see section above) to the listing. If it violates any filter (beds outside 1–2, rent > hard cap, area not in primary ∪ secondary, stated size < 60 m², obviously dated/unrenovated), do NOT create a Notion row — log the skip and move to the next URL.
4. If the listing passes hard filters and page fetch succeeded → extract all available signals per § SCORING, score, create Notion row with `Source=email-alert`.
5. If page fetch failed but the alert email content alone did not trigger a hard-filter rejection → create a minimal Notion row using alert-email data (title, price, beds, URL). Set `Size Source=unknown`, `Needs-verify: listing page unreachable`, `Source=email-alert`. Skip scoring (leave `Score` and `Tier` empty).

### Label-after-handling

After a thread is fully processed (all URLs extracted, dedup-checked, rows inserted or rejected), apply the Gmail label `hunt-processed` via `mcp__claude_ai_Gmail__label_thread`. This keeps the primary Gmail query fast by excluding already-handled threads. On first run, create the label via `mcp__claude_ai_Gmail__create_label` if `mcp__claude_ai_Gmail__list_labels` doesn't return it.

Labeling is non-fatal — if it errors (e.g. transient scope issue), the Notion URL column still prevents duplicate rows on the next run. The worst case is re-processing the same thread once, which is cheap.

### When there are no URLs to extract

If an alert email has zero extractable URLs (e.g. a marketing template, or a template change where our regex misses the new URL shape), do NOT label it `hunt-processed`. A future run may retry once we've fixed the parser. Log the skip but don't fail.

### Deduplication across runs

- **Primary:** Gmail `hunt-processed` label — excludes handled threads from the main query.
- **Secondary:** Notion URL column — every extracted URL is O(1) checked before any listing page is fetched. Catches cases where a listing was seen on a prior run via a different path (scraping vs. alert, or an email we had to re-process after fixing the parser).

## SCRAPING (local mode only, Playwright MCP)

Drives a real Chromium from the laptop's residential IP via Playwright MCP. Bypasses the datacenter-IP gating that blocks WebFetch on Rightmove / Zoopla / SpareRoom.

**Skip this section entirely in remote WebFetch-only mode** — the remote trigger has no Playwright MCP available. Alert ingestion is the only remote input.

### Playwright availability probe

At the start of this section, probe Playwright by calling `mcp__playwright__navigate` with URL `https://example.com`. If it succeeds (any title returned), Playwright is healthy — proceed. If it throws or times out within 30s, mark the run as "scraping skipped (Playwright unavailable)" and go straight to the TRACKER + EMAIL steps with only the alert-ingestion results.

### Portal search URLs (primary areas)

Rightmove uses numeric `REGION_CODE` identifiers; the others take text area slugs.

- **Rightmove** (6 URLs covering the 8 primary areas via broader regions):
  - Islington (covers Islington + Angel + Canonbury): `https://www.rightmove.co.uk/property-to-rent/find.html?locationIdentifier=REGION%5E93965&minBedrooms=[FLAT_BEDROOMS_MIN]&maxBedrooms=[FLAT_BEDROOMS_MAX]&maxPrice=[FLAT_BUDGET_HARD_CAP]&propertyTypes=flat&includeLetAgreed=false&sortType=6`
  - Highbury: `locationIdentifier=REGION%5E70438`
  - Camden Town: `locationIdentifier=REGION%5E85262`
  - Kentish Town: `locationIdentifier=REGION%5E85230`
  - De Beauvoir Town: `locationIdentifier=REGION%5E70393`
  - Clerkenwell: `locationIdentifier=REGION%5E87500`

- **Zoopla** (one URL per primary area, slug = lowercased area with dashes):
  `https://www.zoopla.co.uk/to-rent/flats/[area-slug]/?beds_min=[FLAT_BEDROOMS_MIN]&beds_max=[FLAT_BEDROOMS_MAX]&price_frequency=per_month&price_max=[FLAT_BUDGET_HARD_CAP]&results_sort=newest_listings`
  Slugs: `islington`, `angel`, `camden-town`, `kentish-town`, `de-beauvoir-town`, `highbury`, `canonbury`, `clerkenwell`.

- **SpareRoom** (one URL per primary area, slug = lowercased area with underscores):
  `https://www.spareroom.co.uk/flats-to-rent/london/[area_slug]?min_beds=[FLAT_BEDROOMS_MIN]&max_beds=[FLAT_BEDROOMS_MAX]&max_rent=[FLAT_BUDGET_HARD_CAP]&sort=posted_date&mode=list`
  Slugs: `islington`, `angel`, `camden_town`, `kentish_town`, `de_beauvoir_town`, `highbury`, `canonbury`, `clerkenwell`.

- **OpenRent** (single URL with comma-separated terms):
  `https://www.openrent.co.uk/properties-to-rent/london?term=Islington,Angel,Camden%20Town,Kentish%20Town,De%20Beauvoir%20Town,Highbury,Canonbury,Clerkenwell&prices_max=[FLAT_BUDGET_HARD_CAP]&bedrooms_min=[FLAT_BEDROOMS_MIN]&bedrooms_max=[FLAT_BEDROOMS_MAX]&isLive=true`

### Scraping loop (per portal-area search URL)

1. Call `mcp__playwright__navigate` with the search URL.
2. Call `mcp__playwright__snapshot` to get the rendered DOM tree.
3. Detect block page: if the `mcp__playwright__snapshot` response's title or any `h1`/`h2`/`h3` heading text matches `/Access denied|Just a moment|Something went wrong|unusual activity|403 Forbidden|Please verify you are human|Too many requests|Rate limit exceeded|Access to .* was denied/i`, log "Block detected on {portal}/{area}" to the digest, mark this portal-area as blocked, and skip to the next portal-area. No retry. If you observe a new block page pattern in practice (e.g. a portal rolls out a new challenge), expand this regex in `skill-flats.md`.
4. Otherwise, extract listing URLs from the DOM by matching portal-specific patterns:
   - Rightmove: anchor `href` matching `/properties/\d+` → full URL `https://www.rightmove.co.uk/properties/<id>`.
   - Zoopla: anchor `href` matching `/to-rent/details/\d+/` → full URL `https://www.zoopla.co.uk/to-rent/details/<id>/`.
   - SpareRoom: anchor `href` matching `/flats-to-rent/[^"]+/\d+` or `flatshare_detail.pl?flatshare_id=\d+`.
   - OpenRent: anchor `href` matching `/property-to-rent/[^"]+/\d+`.
5. Walk the extracted URLs in the order they appeared on the page (newest-first). For each URL:
   a. Dedup check (see step 6 below).
   b. If new → enrich + score + insert (see step 7 below).
6. **Dedup check (fail-closed):**
   - Call `mcp__claude_ai_Notion__notion-search` with `query=<URL>`, `data_source_url=collection://[FLAT_TRACKER_FLATS_DATA_SOURCE_ID]`, `page_size=1`, and inspect the result for a page whose URL property equals the candidate.
   - If the MCP call errors (timeout, 5xx, transient network), retry up to 3× with exponential backoff at 1s, 3s, 9s.
   - If still failing after 3 retries, **skip this URL** (do NOT insert) and append a line to the digest's "Dedup-check failures" section.
   - Rationale: Notion `url`-type properties are NOT uniqueness-enforced, so inserting without a successful dedup check can create duplicate rows.
7. **Enrichment:**
   - Call `mcp__playwright__navigate` with the listing URL.
   - Call `mcp__playwright__evaluate` with a portal-specific snippet that pulls title / price / beds / size / floor / furnished / available-from / description text / amenity hints. Examples:
     - Rightmove: `() => window.PAGE_MODEL || document.querySelector('script#__NEXT_DATA__')?.textContent` then parse JSON.
     - Zoopla: `() => window.__PRELOADED_STATE__ || document.querySelector('script#__NEXT_DATA__')?.textContent`.
     - SpareRoom / OpenRent: no reliable global state object. Inspect the rendered page via `mcp__playwright__snapshot` and read visible text for price / beds / size / address / furnished / available-from, plus `<meta property="og:title">`, `<meta name="description">`, and any JSON-LD `<script type="application/ld+json">` block. If fields can't be confidently extracted, insert a minimal row (Title + URL + Platform + Source=`scraped-local` + Needs-verify=`extraction incomplete`) rather than dropping the listing.
   - Apply HARD FILTERS (beds in 1–2, rent ≤ [FLAT_BUDGET_HARD_CAP], area in primary ∪ secondary, stated size ≥ [FLAT_SIZE_FLOOR_M2] or size-inferred per § SIZE RESOLUTION, not obviously dated). If any filter fails, skip — do NOT create a Notion row.
   - Score per § SCORING. Call `mcp__claude_ai_Notion__notion-create-pages` with parent `{type: "data_source_id", data_source_id: "[FLAT_TRACKER_FLATS_DATA_SOURCE_ID]"}` and properties: Title, URL, Platform, Found On (today's date, Europe/Zurich), Area, Price, Beds, Size, Size Source, Furnished, Available From, Floor, Bathtub, New/Renovated, Calm, Light, Wooden Floor, Score, Tier, Status=`NEW`, Reason, Needs-verify, Source=`scraped-local`. Do NOT set Human Tier or Human Rationale.
   - If listing-page navigate times out or the portal returns an empty/error page for this specific listing, create a minimal Notion row with Title + URL + Platform + Source=`scraped-local` + Needs-verify=`listing page unreachable`, and leave Score + Tier blank.

### Pagination cutoff (per portal-area)

Paginate to page 2, then page 3, capped at 3 pages total, subject to this early-exit rule:

- Track **consecutive known listings** (dedup hits) as you walk the extracted URL list in newest-first order.
- Reset the counter to 0 whenever you encounter a new URL.
- **Stop paginating** as soon as 3 consecutive known listings appear — the tracker has caught up for this portal-area.
- Never early-exit on just 1 or 2 consecutive known listings, because mixed-state scenarios (alert-ingested singletons, prior CAPTCHA'd scrapes, laptop-off streaks) routinely create isolated known entries amid new listings.
- Always process (enrich + score + insert) every new URL on every visited page, regardless of the exit condition. Early-exit only controls whether you load the next page.

### Rate limiting

- 2-second wait between `mcp__playwright__navigate` calls on the same portal.
- Portals processed sequentially, not in parallel.
- Target total scrape runtime: 6–12 minutes typical. If a run exceeds 25 minutes, log and exit to avoid blocking downstream steps.

### Secondary-area cadence

Primary areas (8) scrape every run. **Secondary areas (8) scrape only on Monday runs** (day-of-week = 1 in Europe/Zurich). This halves the daily budget and reflects that secondary areas rarely produce HIGH hits.

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

Use `mcp__claude_ai_Gmail__create_draft` (contentType: text/html), To: [FLAT_EMAIL_TO] (destination — the hunt digest target inbox, usually different from `[YOUR_EMAIL]` which is the sender account).

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

- Alert emails queried for the target sender domains in the last 2 days (labeled threads excluded via `-label:hunt-processed`)
- Fully-processed alert threads labeled `hunt-processed` in Gmail
- Playwright MCP availability probed at the start of § SCRAPING. If unavailable, digest records a `⚠️ Scraping skipped (Playwright unavailable)` line.
- Scraping attempted for every primary portal-area; for each one, digest records one of: (a) new rows added, (b) early-exit after ≥3 consecutive known listings, (c) 3-page cap reached, (d) block/CAPTCHA skip.
- On Mondays only, secondary portal-areas also scraped.
- Flats database has new pages for every listing that passed hard filters and survived URL dedup, with `Source=scraped-local` for scraping-origin rows and `Source=email-alert` for alert-origin rows.
- Dedup-check failures (Notion query errored after 3 retries with exponential backoff) surfaced in the digest under a "Dedup-check failures" section; NO row inserted on fail-closed.
- Meta database: `last_local_run` stamped on local runs; `last_remote_run` stamped on remote runs.
- No listings added that violate hard filters.
- No duplicate URLs in Notion (dedup holds across both ingestion paths; URL column is canonical but NOT uniqueness-enforced by Notion — correctness depends on the pre-insert query succeeding).
- HIGH tier caps respected for size-unknown 1-beds.
- Outreach .txt files saved for HIGH priority (local mode only).
- Email draft created (local mode: Send directly if Playwright navigates to Gmail to click send; remote mode: leave as draft for the Apps Script auto-sender).
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
| `[FLAT_EMAIL_TO]` | Email address where the hunt digest is delivered (usually your main inbox) | config.md |
| `[REGION_CODE]` | Rightmove region identifier for London (`REGION%5E87490`) or tighter (per area); leave as-is if unsure | hardcode in skill |
| `[AREA_SLUG]` | zoopla/spareroom URL slug for area (lowercase, underscores) | derived from FLAT_PRIMARY_AREAS |
| `[AREAS_CSV]` | comma-separated areas for OpenRent's `term=` | derived from FLAT_PRIMARY_AREAS |
