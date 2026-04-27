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
- Probe Playwright via `mcp__playwright__browser_navigate` to `https://example.com`. If it responds within 30s with no error, run § SCRAPING for all 4 portals. If it fails, log "Playwright unavailable — scraping skipped" and continue.

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
- **1-bed with verified size < 55 m²** (stated-text or floorplan source). 1-beds where size is not verified are NOT auto-rejected here — see § SIZE RESOLUTION rule 5 (kept, capped at MEDIUM). Rationale: human review consistently downgrades sub-55m² 1-beds with rationales like "very small" / "35sm".
- **Title or description contains the word "studio"** (case-insensitive, word-boundary `\bstudio\b`). Studios are routinely miscategorized by agents as 1-bed and consistently fail the size floor — drop pre-enrichment.
- Obviously dated/unrenovated (tired photos, old fittings, yellowing bathroom)

## SIZE RESOLUTION (priority order)

1. Stated in listing text (structured JSON, when sensible — positive number, 10–500 m², not in nonsensical units) → use, set `Size Source=stated-text`.
2. Stated on floorplan image → extracted via § VISUAL EXTRACTION — FLOORPLAN screenshot during scraping enrichment step 7f. Set `Size Source=floorplan`. This is the most reliable source — floorplan-stated size overrides structured JSON when both are available.
3. **Stated in description free-text** → run a regex over the listing description (Rightmove `description`, Zoopla `description` or `summary`, OpenRent body text) for any of these patterns (case-insensitive):
   - `(\d+(?:\.\d+)?)\s*m\s*²` or `(\d+(?:\.\d+)?)\s*(?:sq\.?\s*m|sqm|square\s*met(?:re|er)s?)` → metres directly.
   - `(\d+(?:\.\d+)?)\s*(?:sq\.?\s*ft|sqft|square\s*f(?:ee|oo)t)` → multiply by 0.0929 to get m².
   Take the FIRST match, sanity-check (10 ≤ size ≤ 500 m²), set `Size Source=stated-text`. This catches listings where the agent typed size into the body but didn't fill the structured field — a recurring failure mode from human-labeled feedback ("info on plans picture" / "info is available on plans picture" appeared in multiple LOW rationales).
4. 2-bed + no size → assume ≥60 m², accept, set `Size Source=inferred-2bed`.
5. 1-bed + no size (no text, no floorplan, no description regex hit):
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
- **Carpet detected in living/bedroom photo: −1.** Confirmed carpet (not wood/tile/vinyl) in living areas drove multiple human LOW labels with rationales like "Weird space, carpet" / "Carpet in rooms".
- **Open-plan kitchen detected: +1.** Kitchen visible from / pass-through to living area is a strong positive signal — the only HIGH-rationale verbatim was "real open kitchen and big living room". Closed/separated kitchen is neutral (no penalty), but record in Reason as `closed-kitchen` for transparency.
- **Noise hotspot match: −2 AND force `Calm=No`.** If the listing title, address, or description contains any substring from `[FLAT_NOISE_HOTSPOTS]` (case-insensitive), apply this penalty. Default hotspots (overridable in config): `Arsenal`, `Emirates Stadium`, `Highbury Stadium Square`, `Drayton Park`, `Wembley`, `White Hart Lane`, `Tottenham Hotspur`. These mark properties in immediate proximity to major venues / arenas where match-day disruption is significant. The user verbatim downgraded a Queensland Rd Arsenal listing from HIGH to MEDIUM citing "right next to the arsenal stadium, will be hell of disruption and noise" — this rule operationalises that.
- **Availability-window alignment** (anchor: [MOVE_IN_DATE], parsed as `MOVE_IN_TARGET` = the first of [MOVE_IN_MONTH] in the year given, e.g. "early June 2026" → 2026-06-01; "mid June 2026" → 2026-06-15):
  - Available From within `[MOVE_IN_TARGET − 14d, MOVE_IN_TARGET + 14d]`: **+1** (sweet spot)
  - Available From within `[MOVE_IN_TARGET − 30d, MOVE_IN_TARGET − 14d)` (slightly early — manageable, costs ~2 weeks of overlapping rent): **0**
  - Available From `< MOVE_IN_TARGET − 30d` (more than a month early — wastes a month+ of rent): **−0.5**
  - Available From `> MOVE_IN_TARGET + 14d` (later than the move-in target — would force temp housing): **−2**
  - Available From unknown / not stated: **0** (don't penalise; many listings don't surface this in structured JSON, leave for human review)
  - Append the resolved bucket to Reason as `avail-aligned` / `avail-early` / `avail-very-early` / `avail-late` / `avail-unknown` for transparency.

Tiers:
- HIGH: score ≥ [FLAT_TIER_HIGH_THRESHOLD] **AND `Size Source ∈ {stated-text, floorplan}`**. The size-verified gate is required for auto-HIGH because human review consistently downgrades unverified-size listings ("No plan", "info on plans picture" appeared as LOW rationales on auto-HIGH-scored rows). If score ≥ HIGH threshold but size is inferred/unknown → tier MEDIUM, append `unverified-size` to Needs-verify.
- MEDIUM: [FLAT_TIER_MEDIUM_THRESHOLD] ≤ score < [FLAT_TIER_HIGH_THRESHOLD], OR (score ≥ HIGH threshold but size is unverified).
- LOW: score < [FLAT_TIER_MEDIUM_THRESHOLD]

**Cap:** size-unknown 1-beds (SIZE RESOLUTION rule 5 above) → tier cannot exceed MEDIUM even if score ≥ HIGH threshold (this is a stricter form of the auto-HIGH gate, kept for clarity).

## INIT — LOAD KNOWN URL SET (runs once, before any ingestion)

**Why this exists:** Notion's `notion-search` MCP tool is semantic, not filter-by-property. Searching by URL value (e.g., `query="https://www.rightmove.co.uk/properties/12345"`) does NOT reliably return the page with that URL property. Per-candidate dedup via `notion-search` is unsafe — every URL would appear "new" and the run would create duplicate rows. This section builds a reliable O(1) in-memory URL set at run start; all downstream dedup checks use it.

**Runs once per run, after MCP availability probes and before any ingestion (alerts or scraping).**

1. Enumerate all pages in the Flats data source by paginating `mcp__claude_ai_Notion__notion-search`. Because empty queries aren't allowed (minLength: 1), use broad queries that cover typical page titles: `"bed"`, `"flat"`. For each query:
   - Call `mcp__claude_ai_Notion__notion-search` with `query=<term>`, `data_source_url=collection://[FLAT_TRACKER_FLATS_DATA_SOURCE_ID]`, `page_size=25`, `max_highlight_length=0`.
   - If the response contains `nextPageToken`, repeat with that token until no more pages. Cap at 20 pagination iterations per query as a safety net.
   - Collect each result's `id` (page UUID).
2. Union the page UUIDs from all queries into a single unique set. Typical result: 30–200 pages.
3. For each unique page UUID, call `mcp__claude_ai_Notion__notion-fetch` with the UUID. Read the `userDefined:URL` property from the returned `<properties>` block. Build an in-memory map: `url_string → page_uuid`.
4. Store the map as the **known URL set** for this run. All dedup checks (alert ingestion step 7.a and scraping step 6) consult this set — no further `notion-search` dedup calls are made.

**Error handling:**
- If `notion-search` errors on a pagination call: retry up to 3× with exponential backoff (1s, 3s, 9s). If still failing, log "URL set enumeration failed — aborting run" and exit without inserting any rows. Do NOT proceed with unreliable dedup.
- If `notion-fetch` errors on an individual page: retry once, then skip that page. Log the skip count in the digest. A partial URL set is acceptable as long as <5% of pages fail to fetch; otherwise abort.
- If the Notion MCP is entirely unavailable: exit immediately with a digest line "⚠️ Notion unavailable — full run aborted".

**Rationale for fail-closed behavior:** duplicate rows pollute the tracker permanently and are costly to clean up. A missed run is recoverable on the next day. Favor correctness.

**In-memory use only:** the URL set is not persisted. Every run rebuilds it from scratch. The data source is the source of truth.

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

1. Check the candidate URL against the **known URL set** built in § INIT — LOAD KNOWN URL SET. If the URL is in the set, skip this URL (dedup hit). O(1) local lookup — do NOT call `notion-search` for per-URL dedup, it is semantic and unreliable for URL-property matches.
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
- **Secondary:** In-memory known URL set built once at run start by § INIT — LOAD KNOWN URL SET. Every extracted URL is checked in O(1) local lookup before any listing page is fetched. Catches cases where a listing was seen on a prior run via a different path (scraping vs. alert, or an email we had to re-process after fixing the parser).

## SCRAPING (local mode only, Playwright MCP)

Drives a real Chromium from the laptop's residential IP via Playwright MCP. Bypasses the datacenter-IP gating that blocks WebFetch on Rightmove / Zoopla / SpareRoom.

**Skip this section entirely in remote WebFetch-only mode** — the remote trigger has no Playwright MCP available. Alert ingestion is the only remote input.

### Mandatory when Playwright is healthy

If Playwright is available, this section **MUST** run to completion for all 4 portals × (8 primary + 8 secondary) areas. **Scrape primary areas first, then secondary**, so the 60 min wall-clock kill (§ Rate limiting) preserves primary coverage if the budget runs out mid-scrape. Scraping is not optional based on "alert-ingestion already produced rows" or "projected runtime is long" or "alert/scrape overlap looks high". The whole point of scraping is to cover Zoopla / SpareRoom / OpenRent that alerts don't see. Overlap with alert-sourced Rightmove rows is EXPECTED and handled by the O(1) local URL dedup (§ INIT — LOAD KNOWN URL SET) — it is not a reason to skip.

**Critical**: Rightmove area scraping (the search-pages-by-area in this section) is DISTINCT from Rightmove alert-URL ingestion (§ ALERT INGESTION). Both must run. Alerts cover individual listings the user has saved-searches for; area scraping covers everything else listed in the area. They overlap by design and dedup handles it.

Only legitimate reasons to skip or abort scraping:
- Playwright unavailable (probe below fails).
- A single `mcp__playwright__browser_navigate` exceeds 30s timeout → skip that portal-area only, continue with the next.
- Total scraping wall-clock exceeds 60 min → mid-run kill (log "Scraping budget exceeded after N portal-areas" and proceed to TRACKER + EMAIL). This is a hard stop, **not** a pre-flight deferral.

**Self-budgeting (anticipating runtime to skip work) is FORBIDDEN.** "I'll skip Rightmove and SpareRoom because the run is taking long" is not a valid skip — the only time-based skip is the 60 min hard kill, observed after the fact, not predicted. Recurring failure mode in past runs: agent skipped portal-areas based on its own runtime forecast, finished at ~38 min wall (well under budget), and logged "skipped (budget)". Don't.

**Prior-coverage skips are also FORBIDDEN.** "An earlier cron today already scraped this area, so I'll skip it" is not a valid reason. Each run scrapes every portal-area independently; dedup against the in-memory URL set (§ INIT) handles overlap with no listing-creation cost. Listings can change status, price, or be re-listed by different agents within hours — re-scraping is cheap and correct. Observed 2026-04-27: agent skipped 5 of 6 RM primary regions citing "12:14 cron covered them ~80 min earlier"; that's a regression.

### Per-area accounting (REQUIRED in the digest)

The end-of-run digest MUST include a per-portal-area outcome line for **all** 4 portals × 16 areas (= effectively 14 Rightmove URLs covering 8 primary + 8 secondary, 16 Zoopla URLs, 16 SpareRoom URLs, and 2 combined OpenRent URLs). Format each as one line:

`<portal>/<area>: <outcome> [<count>]`

Where `<outcome>` is one of:
- `OK` — page loaded, listings extracted, dedup + enrichment ran. `<count>` = listings inserted.
- `EARLY-EXIT` — 3 consecutive known listings observed, paginated stop. `<count>` = listings inserted before exit.
- `BLOCKED` — block/CAPTCHA detected, no retry. `<count>` = 0.
- `TIMEOUT` — single navigate exceeded 30s. `<count>` = 0.
- `BUDGET-KILL` — 60 min wall-clock exceeded; this and all later areas not attempted.
- `ZERO-RESULTS` — page loaded but listing-URL regex matched nothing (often a portal URL-format change — flag explicitly).

If any portal-area is missing from this list, the run is incomplete. Self-budgeted skips are NOT a permitted outcome — they would surface here as untyped gaps and that is a regression.

### Playwright availability probe

At the start of this section, probe Playwright by calling `mcp__playwright__browser_navigate` with URL `https://example.com`. If it succeeds (any title returned), Playwright is healthy — proceed with the full scrape as specified below. If it throws or times out within 30s, mark the run as "scraping skipped (Playwright unavailable)" and go straight to the TRACKER + EMAIL steps with only the alert-ingestion results.

### Portal search URLs (primary areas)

Rightmove uses numeric `REGION_CODE` identifiers; the others take text area slugs.

- **Rightmove** (6 URLs covering the 8 primary areas via broader regions):
  - Islington (covers Islington + Angel + Canonbury): `https://www.rightmove.co.uk/property-to-rent/find.html?locationIdentifier=REGION%5E93965&minBedrooms=[FLAT_BEDROOMS_MIN]&maxBedrooms=[FLAT_BEDROOMS_MAX]&maxPrice=[FLAT_BUDGET_HARD_CAP]&propertyTypes=flat&includeLetAgreed=false&sortType=6`
  - Highbury: `locationIdentifier=REGION%5E70438`
  - Camden Town: `locationIdentifier=REGION%5E85262`
  - Kentish Town: `locationIdentifier=REGION%5E85230`
  - De Beauvoir Town: `locationIdentifier=REGION%5E70393`
  - Clerkenwell: `locationIdentifier=REGION%5E87500`

- **Zoopla** (one URL per primary area; the path is `/to-rent/flats/london/<slug>/` AND the area name is also passed via `q=<Area+Name>` because `london` is a broad city-level slug — the `q` param narrows it to the actual neighbourhood):
  `https://www.zoopla.co.uk/to-rent/flats/london/[area-slug]/?q=[area+name]&beds_min=[FLAT_BEDROOMS_MIN]&beds_max=[FLAT_BEDROOMS_MAX]&price_frequency=per_month&price_max=[FLAT_BUDGET_HARD_CAP]&results_sort=newest_listings`
  Slugs (path): `islington`, `angel`, `camden-town`, `kentish-town`, `de-beauvoir-town`, `highbury`, `canonbury`, `clerkenwell`.
  `q` values (URL-encoded, capitalised): `Islington`, `Angel`, `Camden+Town`, `Kentish+Town`, `De+Beauvoir+Town`, `Highbury`, `Canonbury`, `Clerkenwell`.
  **Note**: Zoopla's URL scheme has churned three times in two weeks (`/to-rent/flats/<slug>/` → `/to-rent/property/<slug>/?property_type=flats` → current). If you encounter zero-result pages on a fresh Zoopla URL, fall back to discovery: navigate `https://www.zoopla.co.uk/to-rent/`, type the area into the search box, observe the redirected URL, and use that pattern. Update this skill section with the new format if you confirm one.

- **SpareRoom** (one URL per primary area, slug = lowercased area with underscores):
  `https://www.spareroom.co.uk/flats-to-rent/london/[area_slug]?min_beds=[FLAT_BEDROOMS_MIN]&max_beds=[FLAT_BEDROOMS_MAX]&max_rent=[FLAT_BUDGET_HARD_CAP]&sort=posted_date&mode=list`
  Slugs: `islington`, `angel`, `camden_town`, `kentish_town`, `de_beauvoir_town`, `highbury`, `canonbury`, `clerkenwell`.

- **OpenRent** (single URL with comma-separated terms):
  `https://www.openrent.co.uk/properties-to-rent/london?term=Islington,Angel,Camden%20Town,Kentish%20Town,De%20Beauvoir%20Town,Highbury,Canonbury,Clerkenwell&prices_max=[FLAT_BUDGET_HARD_CAP]&bedrooms_min=[FLAT_BEDROOMS_MIN]&bedrooms_max=[FLAT_BEDROOMS_MAX]&isLive=true`

### Portal search URLs (secondary areas)

Run after primary areas complete. Same URL patterns as above; only the area slug / location identifier changes. For Rightmove, REGION codes for secondary areas aren't pre-cached — build the URL with `searchLocation=<area+name>` (URL-encoded) instead of `locationIdentifier=REGION%5E<id>`. Rightmove will resolve the search and the listing extraction logic in step 7b is unchanged.

- **Rightmove** (8 URLs, one per secondary area, using `searchLocation` text search):
  `https://www.rightmove.co.uk/property-to-rent/find.html?searchLocation=[area+name]&minBedrooms=[FLAT_BEDROOMS_MIN]&maxBedrooms=[FLAT_BEDROOMS_MAX]&maxPrice=[FLAT_BUDGET_HARD_CAP]&propertyTypes=flat&includeLetAgreed=false&sortType=6`
  Areas (URL-encoded): `Tufnell+Park`, `Holloway`, `Bloomsbury`, `Russell+Square`, `Barbican`, `Finsbury+Park`, `London+Bridge`, `Bermondsey`.

- **Zoopla** (one URL per secondary area; same `/to-rent/flats/london/<slug>/?q=<Area>` pattern as primary):
  Slugs (path): `tufnell-park`, `holloway`, `bloomsbury`, `russell-square`, `barbican`, `finsbury-park`, `london-bridge`, `bermondsey`.
  `q` values (URL-encoded, capitalised): `Tufnell+Park`, `Holloway`, `Bloomsbury`, `Russell+Square`, `Barbican`, `Finsbury+Park`, `London+Bridge`, `Bermondsey`.

- **SpareRoom** (one URL per secondary area, slug = lowercased underscored):
  Slugs: `tufnell_park`, `holloway`, `bloomsbury`, `russell_square`, `barbican`, `finsbury_park`, `london_bridge`, `bermondsey`.

- **OpenRent** (single combined URL):
  `https://www.openrent.co.uk/properties-to-rent/london?term=Tufnell%20Park,Holloway,Bloomsbury,Russell%20Square,Barbican,Finsbury%20Park,London%20Bridge,Bermondsey&prices_max=[FLAT_BUDGET_HARD_CAP]&bedrooms_min=[FLAT_BEDROOMS_MIN]&bedrooms_max=[FLAT_BEDROOMS_MAX]&isLive=true`

### Scraping loop (per portal-area search URL)

1. Call `mcp__playwright__browser_navigate` with the search URL.
2. Call `mcp__playwright__browser_snapshot` to get the rendered DOM tree.
3. Detect block page: if the `mcp__playwright__browser_snapshot` response's title or any `h1`/`h2`/`h3` heading text matches `/Access denied|Just a moment|Something went wrong|unusual activity|403 Forbidden|Please verify you are human|Too many requests|Rate limit exceeded|Access to .* was denied/i`, log "Block detected on {portal}/{area}" to the digest, mark this portal-area as blocked, and skip to the next portal-area. No retry. If you observe a new block page pattern in practice (e.g. a portal rolls out a new challenge), expand this regex in `skill-flats.md`.
4. Otherwise, extract listing URLs from the DOM by matching portal-specific patterns:
   - Rightmove: anchor `href` matching `/properties/\d+` → full URL `https://www.rightmove.co.uk/properties/<id>`.
   - Zoopla: anchor `href` matching `/to-rent/details/\d+/` → full URL `https://www.zoopla.co.uk/to-rent/details/<id>/`.
   - SpareRoom: anchor `href` matching `/flats-to-rent/[^"]+/\d+` or `flatshare_detail.pl?flatshare_id=\d+`.
   - OpenRent: anchor `href` matching `/property-to-rent/[^"]+/\d+`.
5. Walk the extracted URLs in the order they appeared on the page (newest-first). For each URL:
   a. Dedup check (see step 6 below).
   b. If new → enrich + score + insert (see step 7 below).
6. **Dedup check (O(1) local lookup):**
   - Check the candidate URL against the **known URL set** built in § INIT — LOAD KNOWN URL SET. If present, skip this URL (dedup hit).
   - Do NOT call `notion-search` for per-URL dedup — semantic search does not reliably match URL-property values, so per-candidate search would treat every URL as new and create duplicate rows.
   - The known URL set is populated once at run start via Flats data source enumeration and is the canonical dedup source for this run.
7. **Enrichment (Rightmove / Zoopla):**
   a. Call `mcp__playwright__browser_navigate` with the listing URL.
   b. Call `mcp__playwright__browser_evaluate` with a portal-specific snippet that pulls title / price / beds / size / floor / furnished / available-from / description text / amenity hints, **plus floorplan image URL(s) and listing photo URLs with captions**. Examples:
      - Rightmove: `() => { const m = window.PAGE_MODEL; if (!m) return null; const p = m.propertyData || {}; return { ...p, _floorplanUrl: ((p.floorplans || p.floorplanImages || [])[0]?.url) || null, _photos: (p.images || []).map(i => ({ url: i.url, caption: i.caption })) }; }` then parse the result.
      - Zoopla: `() => { const d = window.__PRELOADED_STATE__ || JSON.parse(document.querySelector('script#__NEXT_DATA__')?.textContent || '{}'); return d; }` then locate the listing object, floorplan URL (path varies — look for a `floorPlanImages`, `floorPlan`, `floorplan`, or `floorPlanImage` key in the listing data), and photo array.
   c. **Cheap hard filters (structured data + title/description text):** reject immediately if any of:
      - beds outside 1–2
      - rent > [FLAT_BUDGET_HARD_CAP]
      - area not in primary ∪ secondary
      - **title or description matches `\bstudio\b`** (case-insensitive, word-boundary) — see § HARD FILTERS
      If any fails → skip this listing entirely, no visual extraction. (Note: "obviously dated/unrenovated" is NOT a cheap filter for RM/Zoopla — photos now provide the New/Renovated signal via step 7g, and § SCORING already penalizes dated listings.)
   d. **Description-text size pass (cheap, runs before floorplan navigation):** if the structured JSON has no sensible size, scan the description body with the regex patterns in § SIZE RESOLUTION rule 3. If a hit is found, record it as `_descriptionSize` with `Size Source=stated-text` for the merge step (7g). Don't reject yet — let the floorplan pass run too, and the merge will pick the best source.
   e. **Size-gate check:** if a sensible size is now known (from structured JSON or step 7d description regex) AND it is below [FLAT_SIZE_FLOOR_M2] → reject immediately, no visual extraction. **Additional 1-bed gate:** if beds=1 AND the verified size (`stated-text` or `floorplan` from any prior step) is < 55 m² → reject (per § HARD FILTERS).
   f. **Floorplan pass (Rightmove / Zoopla only, if floorplan URL exists):** see § VISUAL EXTRACTION — FLOORPLAN below.
   g. **Photo pass (Rightmove / Zoopla only, if listing photos exist):** see § VISUAL EXTRACTION — PHOTOS below.
   h. **Merge visual data** into the listing record. Priority rules:
      - **Size:** floorplan-stated size wins (set `Size Source=floorplan`). If no floorplan size, use the description-regex size from step 7d if available (set `Size Source=stated-text`). If neither, use sensible structured JSON size from the evaluate result (set `Size Source=stated-text`). Otherwise fall back to § SIZE RESOLUTION inference rules.
      - **Boolean/enum fields** (bathtub, wood floor, light, new/renovated, floor): visual evidence overrides "Unknown" from structured data. Visual evidence never downgrades a "Yes" to "No" (the feature might not be photographed). If both floorplan and photos provide a signal, "Yes" wins.
      - **Floor level:** floorplan text overrides vague JSON. Only set "Top" when the source explicitly says top/penthouse/uppermost, or when both current floor and total floors are known and match. Ground/Lower Ground = "Ground". All other numbered floors = "Mid". Add `Needs-verify: floor` when the floor number is ≥3 but top-floor status can't be confirmed.
      - **Carpet / open-kitchen booleans** (from § VISUAL EXTRACTION — PHOTOS): feed § SCORING modifiers and append to Reason text as `carpet✓` / `open-kitchen✓` / `closed-kitchen` for transparency. Not stored as Notion columns.
   i. **Post-visual size filter:** if the merged size (from floorplan, description regex, structured data, or inference) is below [FLAT_SIZE_FLOOR_M2] → reject. **1-bed extra gate:** if beds=1 AND merged Size Source ∈ {stated-text, floorplan} AND merged size < 55 m² → reject. Do NOT create a Notion row.
   j. Score per § SCORING. Call `mcp__claude_ai_Notion__notion-create-pages` with parent `{type: "data_source_id", data_source_id: "[FLAT_TRACKER_FLATS_DATA_SOURCE_ID]"}` and properties: Title, URL (canonicalised per § TRACKER URL normalisation), Platform, Found On (today's date, Europe/Zurich), Area, Price, Beds, Size, Size Source, Furnished, Available From, Floor, Bathtub, New/Renovated, Calm, Light, Wooden Floor, Score, Tier, Status=`NEW`, Reason, Needs-verify, Source=`scraped-local`. Do NOT set Human Tier or Human Rationale.
   - If listing-page navigate times out or the portal returns an empty/error page for this specific listing, create a minimal Notion row with Title + URL + Platform + Source=`scraped-local` + Needs-verify=`listing page unreachable`, and leave Score + Tier blank.

   **Enrichment (SpareRoom / OpenRent):**
   - No floorplan or photo visual extraction. Use the existing SpareRoom/OpenRent enrichment approach: inspect the rendered page via `mcp__playwright__browser_snapshot` and read visible text for price / beds / size / address / furnished / available-from, plus `<meta property="og:title">`, `<meta name="description">`, and any JSON-LD `<script type="application/ld+json">` block. If fields can't be confidently extracted, insert a minimal row (Title + URL + Platform + Source=`scraped-local` + Needs-verify=`extraction incomplete`) rather than dropping the listing.
   - Apply HARD FILTERS (beds in 1–2, rent ≤ [FLAT_BUDGET_HARD_CAP], area in primary ∪ secondary, stated size ≥ [FLAT_SIZE_FLOOR_M2] or size-inferred per § SIZE RESOLUTION). If any filter fails, skip — do NOT create a Notion row.
   - Score and insert as per step 7j above.

### VISUAL EXTRACTION — FLOORPLAN

This section is called from enrichment step 7f. It runs for Rightmove and Zoopla listings only, when the structured JSON contains a floorplan image URL.

1. Extract the floorplan image URL:
   - **Rightmove:** `_floorplanUrl` from the evaluate result (populated from `PAGE_MODEL.propertyData.floorplans[0].url`).
   - **Zoopla:** locate the floorplan URL in the evaluate result. Look for keys named `floorPlanImages`, `floorPlan`, `floorplan`, or `floorPlanImage` in the listing data object. Take the first URL found.
   - If no floorplan URL is found, skip this section — continue to step 7g (photo pass).

2. Call `mcp__playwright__browser_navigate` with the floorplan image URL. Timeout: 15 seconds. If navigation fails or times out, log "Floorplan navigation failed for {listing URL}" and skip to step 7g.

3. Call `mcp__playwright__browser_take_screenshot`. If the screenshot fails, log and skip to step 7g.

4. Examine the screenshot and extract:
   - **Total floor area:** scan the ENTIRE image for ANY area text — corners, margins, top, bottom, embedded in title block, small print at edges. Common patterns:
     - "Approx. Gross Internal Floor Area NNN sq. ft / NN.NN sq. m"
     - "Total Area: NNN sq ft" / "Total: NN m²"
     - "GIA: NNN sqft" / "Approximate Floor Area: NN m²"
     - Just a bare "NNN sq ft" or "NN m²" near the title or in a corner
     - Some plans split per-room areas — sum them only as a last resort, and clearly mark the resulting size as `Size Source=floorplan` with `Needs-verify: size-summed`.
     If the area is in sq ft only, convert: `m² = sq_ft × 0.0929`. Round to 1 decimal place.
     Be aggressive — historical labeled data shows multiple LOW rationales of "info on plans picture" / "info is available on plans picture" (i.e., the human could see it but our extraction missed it). If the screenshot is too low-res to read small text, retry with `mcp__playwright__browser_take_screenshot` after `mcp__playwright__browser_resize` to a larger viewport (e.g. 1600×1200) once before giving up.
   - **Floor level:** look for text like "Fourth Floor", "Ground Floor", "First Floor", "Basement", "Lower Ground Floor" printed on or near the plan. Map to the Floor enum:
     - "Top" — only if the text explicitly says "top floor", "penthouse", or "uppermost", OR if both the current floor number and total building floors are known and they match.
     - "Ground" — if it says "Ground Floor" or "Lower Ground Floor".
     - "Mid" — all other numbered floors.
     - If the floor number is ≥3 but you cannot confirm it's the top floor, set Floor="Mid" and append `floor` to the Needs-verify field.
   - **Bathtub hint:** look for a rectangular bath shape in the bathroom area of the plan (distinct from the smaller square/quadrant shower cubicle shape). Set Bathtub="Yes" if clearly visible, "No" if the bathroom only shows a shower, "Unknown" if unclear.

5. Return the extracted values (size_m2, floor, bathtub) to the merge step (7g).

### VISUAL EXTRACTION — PHOTOS

This section is called from enrichment step 7g. It runs for Rightmove and Zoopla listings only, when the structured JSON contains listing photo URLs.

1. Select up to 3 photos from the listing's photo array:
   - **Caption-based selection (preferred):** scan the `_photos` array (Rightmove) or equivalent Zoopla photo array for entries whose `caption` (case-insensitive) contains any of: `bathroom`, `bath`, `shower room`, `kitchen`, `living`, `bedroom`, `reception`, `lounge`. Select photos in this priority order:
     a. First photo with a bathroom/bath/shower caption (highest value — bathtub detection).
     b. First photo with a living/reception/lounge/bedroom caption (wood floor, light).
     c. First photo with a kitchen caption (renovation state).
   - **Index-based fallback:** if fewer than 3 photos matched by caption, fill remaining slots from indices 0, 3, 6 of the photo array (spread across the gallery). Skip any index that was already selected by caption or that exceeds the array length.
   - Hard cap: 3 photos total. If the listing has fewer than 3 photos, take all available.

2. For each selected photo:
   a. Call `mcp__playwright__browser_navigate` with the photo URL. Timeout: 15 seconds. If navigation fails or times out, skip this photo — continue to the next.
   b. Call `mcp__playwright__browser_take_screenshot`. If the screenshot fails, skip this photo.
   c. Examine the screenshot and extract the relevant fields based on what the photo shows:
      - **Bathroom photo:** Bathtub — "Yes" if a rectangular tub is visible (freestanding, built-in, or shower-over-bath), "No" if only a shower stall/cubicle is visible, "Unknown" if unclear. Also note Light and New/Renovated if assessable.
      - **Room photo (living/bedroom/reception):** Wood floor — "Yes" if hardwood, parquet, or engineered-wood flooring is visible (laminate that looks like wood counts as "Yes"), "No" if carpet, tile, or vinyl is visible, "Unknown" if unclear. **Carpet flag** — separately, if carpet is the visible flooring (any colour, any pile), mark `carpet=true` for this photo. This is treated as a scoring penalty (see § SCORING) distinct from "wood=No" so we don't lose the signal when wood is absent for non-carpet reasons (tile, vinyl). Light — "Good" if large windows with visible daylight, "Poor" if small/few windows or dark, "Avg" if mixed, "Unknown" if photo doesn't show windows. New/Renovated — "Yes" if modern fittings, contemporary style, fresh paint, "No" if dated fixtures/wallpaper/worn surfaces, "Unknown" if unclear.
      - **Kitchen photo:** New/Renovated — "Yes" if modern kitchen with contemporary units/appliances, "No" if dated, "Unknown" if unclear. **Open-plan flag** — separately, if the photo shows a kitchen that is visually continuous with a living/dining space (no full-height wall separating the cooking area from a sofa/dining/living zone in the same frame, or a clear pass-through opening), mark `open_kitchen=true`. If the kitchen is fully enclosed (own room, doorway separator), mark `open_kitchen=false`. If unclear from the photo, leave `open_kitchen=null`. If a living/reception photo shows the kitchen visible in the same frame, that ALSO counts as `open_kitchen=true`.

3. Consolidate across all photos: for each field (bathtub, wood floor, light, new/renovated, carpet, open_kitchen), if any photo yielded a definitive "Yes"/"No"/`true`/`false`, use that. If multiple photos disagree, prefer "Yes"/`true` over "No"/`false` for positive signals (bathtub, wood, open_kitchen — the negative photo might show a different room). For carpet, conservative rule: only mark `carpet=true` if it appears in 2+ room photos OR if a single living-area photo clearly shows carpet (avoid false positives from one bedroom photo). If all photos yielded "Unknown"/`null`, keep "Unknown"/`null`.

4. Return the consolidated values (bathtub, wood_floor, light, new_renovated, carpet, open_kitchen) to the merge step (7g). The carpet and open_kitchen booleans feed the scoring modifiers in § SCORING and should be appended to the Reason text as `carpet✓` / `open-kitchen✓` / `closed-kitchen` for transparency. They do NOT need separate Notion columns — the score and Reason capture the signal.

### Pagination cutoff (per portal-area)

Paginate to page 2, then page 3, capped at 3 pages total, subject to this early-exit rule:

- Track **consecutive known listings** (dedup hits) as you walk the extracted URL list in newest-first order.
- Reset the counter to 0 whenever you encounter a new URL.
- **Stop paginating** as soon as 3 consecutive known listings appear — the tracker has caught up for this portal-area.
- Never early-exit on just 1 or 2 consecutive known listings, because mixed-state scenarios (alert-ingested singletons, prior CAPTCHA'd scrapes, laptop-off streaks) routinely create isolated known entries amid new listings.
- Always process (enrich + score + insert) every new URL on every visited page, regardless of the exit condition. Early-exit only controls whether you load the next page.

### Rate limiting

- 2-second wait between `mcp__playwright__browser_navigate` calls to portal search pages and listing pages on the same portal. This rate limit does NOT apply to CDN image URLs (floorplans, listing photos) — those can be navigated without delay.
- Portals processed sequentially, not in parallel.
- Typical scrape runtime: 15–30 minutes (with visual extraction). **Hard kill at 60 minutes** of scraping wall-clock — if reached, log "Scraping budget exceeded after N portal-areas" and proceed to TRACKER + EMAIL with whatever was captured. This is an in-progress interrupt only; it never justifies pre-flight skip. Start scraping immediately after the availability probe succeeds.

### Secondary-area cadence

Both primary (8) and secondary (8) areas scrape every run. The 60 min wall-clock kill is a sufficient budget guard — when we hit it, we already-have-good-coverage on primary by ordering: scrape primary areas first, then secondary. If the budget runs out mid-secondary, that's acceptable (secondary historically produces fewer HIGH hits anyway, so partial coverage is fine). The earlier Monday-only restriction starved 6/7 daily runs of secondary-area visibility for a marginal budget win — not worth it.

## TRACKER

Notion workspace with two databases under parent page [FLAT_TRACKER_NOTION_PARENT_PAGE_ID]:

- **Flats database** — data source id [FLAT_TRACKER_FLATS_DATA_SOURCE_ID]. One page per listing.
- **Meta database** — data source id [FLAT_TRACKER_META_DATA_SOURCE_ID]. Key/Value rows for `last_local_run`, `last_remote_run`, `schema_version`.

Properties on Flats (see tracker/flats-schema.md for full types): Title (title), URL, Platform, Found On, Area, Price, Beds, Size, Size Source, Furnished, Available From, Floor, Bathtub, New/Renovated, Calm, Light, Wooden Floor, Score, Tier, Status, Reason, Needs-verify, Notes, Source.

Dedup: use the in-memory known URL set built once at run start by § INIT — LOAD KNOWN URL SET. Skip the candidate if its URL is in the set. Do NOT use `notion-search` for per-URL dedup — it is semantic and does not reliably match URL-property values.

**URL normalisation before set lookup and before insertion** — listings can re-arrive with cosmetically-different URLs that resolve to the same canonical page. Normalise every candidate URL (and every URL stored in the known set) by:
1. Lowercasing the scheme + host.
2. Stripping trailing slashes from the path.
3. Dropping `?utm_*`, `?source=`, `?ref=`, `?campaign=`, `&channel=`, and any tracking query parameters.
4. For Rightmove: keep only `/properties/<id>` — strip everything after.
5. For Zoopla: keep only `/to-rent/details/<id>/` — strip everything after.
6. For OpenRent: keep only `/property-to-rent/london/<slug>/<id>` — strip everything after.
This catches the trailing-slash / trailing-`?` / utm-tagged variants that historically slipped past dedup and produced duplicate rows for the same listing.

**Residual duplicate source (acknowledged limitation):** the same physical property is sometimes re-listed by a different agent with a NEW listing ID (e.g., Bartholomew Rd appeared with two distinct Rightmove IDs days apart). URL-based dedup cannot catch these. A future enhancement could add an address+price+beds fingerprint as a secondary check; for now, accept the rare dupe and rely on human review to spot it.

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
- Scraping attempted for every primary AND secondary portal-area on every run (primary first, then secondary so budget kills preserve primary coverage). The digest's per-area accounting block (see § Per-area accounting) MUST list outcomes for all expected portal-areas (4 portals × 16 areas). A run that omits any expected area is incomplete and should be flagged. Self-budgeted "skipped (low yield expected)" outcomes are NOT permitted — only the post-hoc 60 min wall-clock kill (`BUDGET-KILL`) is a valid time-based skip.
- Flats database has new pages for every listing that passed hard filters and survived URL dedup, with `Source=scraped-local` for scraping-origin rows and `Source=email-alert` for alert-origin rows.
- Dedup-check failures (Notion query errored after 3 retries with exponential backoff) surfaced in the digest under a "Dedup-check failures" section; NO row inserted on fail-closed.
- Meta database: `last_local_run` stamped on local runs; `last_remote_run` stamped on remote runs.
- No listings added that violate hard filters.
- No duplicate URLs in Notion (dedup holds across both ingestion paths; URL column is canonical but NOT uniqueness-enforced by Notion — correctness depends on the pre-insert query succeeding).
- HIGH tier caps respected for size-unknown 1-beds, AND for any listing where `Size Source ∉ {stated-text, floorplan}` (auto-HIGH requires verified size — see § SCORING Tiers).
- No "studio" listings inserted (rejected at § HARD FILTERS via title/description word-boundary match).
- No 1-bed listings inserted with verified size < 55 m² (the new floor for 1-beds with stated-text or floorplan sizes).
- Description-text size regex (§ SIZE RESOLUTION rule 3) attempted for every Rightmove/Zoopla listing with no structured-JSON size; digest records hit-rate vs. attempts.
- Carpet / open-kitchen booleans surfaced in Reason text on every Rightmove/Zoopla row that ran photo extraction.
- For Rightmove/Zoopla scraped listings with floorplan images, `Size Source = floorplan` should appear on the majority of new rows. If consistently low, investigate whether floorplan URL extraction or screenshot reading is failing silently.
- Visual extraction (floorplan + photos) attempted for every new Rightmove/Zoopla listing that passes cheap hard filters. Digest records count of successful floorplan reads vs. total attempts.
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
