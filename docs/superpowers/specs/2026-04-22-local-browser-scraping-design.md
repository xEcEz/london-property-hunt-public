# Local Browser Scraping (Playwright MCP) — Design Spec

**Date:** 2026-04-22
**Author:** Raphael (rvonaar@gmail.com)
**Status:** Draft — pending user review
**Target repo:** `london-property-hunt-public` (flats variant)
**Supersedes:** scraping-only ingestion (2026-04-16 spec) and alerts-only local mode introduced in the 2026-04-21 layered-ingestion spec.

---

## 1. Context

Six days of real operation revealed reliability problems with every ingestion path we've tried:

- **WebFetch scraping** of portal search pages gets HTTP 403 from Anthropic's datacenter IPs on Rightmove / Zoopla / SpareRoom (~50% of runs). WebFetch from local cron also runs on datacenter IPs, so "local" scraping via WebFetch is no better than remote.
- **Gmail alert ingestion** works for Rightmove (plaintext contains tracker-wrapped URLs that redirect to canonical listings — we added a redirect-follow step on 2026-04-22 and harvested 31 rows that day). It **does not work** for Zoopla (per-listing URLs are HTML-only; Gmail MCP exposes only `plaintextBody`), nor for SpareRoom / OpenRent (alerts not arriving 24h+ after saved-search setup).
- **"Claude in Chrome"** extension turned out to be a browser-side product for claude.ai web sessions, not a Claude Code CLI MCP. It doesn't bridge to the skill at all.
- **Laptop availability** is ~70% (weekdays only), so "local-only" architectures miss ~30% of days.

The 2026-04-22 smoke test of **Playwright MCP** (`@playwright/mcp`, official Microsoft) confirmed it launches its own Chromium, uses the laptop's residential IP, and successfully loads a Rightmove search page (25 listings extracted, no CAPTCHA/block). This spec defines a layered architecture built around that capability.

## 2. Goals

- Restore **full-fidelity scraping** as the primary local ingestion path, driving all 4 portals with residential IP + real browser — no datacenter gating.
- Keep **alert ingestion** as a secondary local path (freshness catch) and the sole remote-trigger path (laptop-off fallback).
- Both paths dedup via the Notion URL column; no duplicate rows.
- Near-zero ongoing cost (Playwright MCP is free, self-hosted on the laptop).
- Remove obsolete layers (WebFetch scraping fallback, Claude-in-Chrome autostart hacks, Zoopla-plaintext workarounds).

## 3. Non-goals

- Browserbase or any hosted browser (~$40/mo). Not needed now that Playwright + residential IP works.
- Multi-laptop / multi-user architecture. Single-operator tool.
- Real-time / sub-minute latency. Daily runs remain the cadence.
- Replacing the alert ingestion path entirely — the Rightmove alert path is working today and gives freshness that daily scraping can't.
- Rewriting the scoring model, Notion schema, email digest, or outreach template.

## 4. Architecture

```
LOCAL CRON — daily 10:00 Europe/Zurich on Raphael's laptop (residential IP, Chromium via Playwright MCP)
  ├─ Step 1: Alert ingestion (~30s)
  │           queries Gmail; extracts URLs from Rightmove alert plaintext (tracker-wrapped →
  │           redirect-follow → canonical listing URL). Zoopla/SpareRoom/OpenRent: best-effort;
  │           current templates yield 0 URLs and that's OK.
  │
  ├─ Step 2: Browser-MCP scraping (~10–15 min)
  │           for each of {Rightmove, Zoopla, SpareRoom, OpenRent} × {8 primary areas}:
  │             - Playwright navigate to search URL (sorted by newest)
  │             - Extract listing URLs from results
  │             - Dedup against Notion; for each NEW url:
  │                 - Playwright navigate to listing page, extract fields
  │                 - Apply HARD FILTERS; score; insert to Notion with Source=scraped-local
  │             - Early-exit: if a page yields ≥1 dedup hit, stop paginating that portal-area
  │
  ├─ Step 3: Build digest email (draft), Source breakdown per portal
  └─ Step 4: Stamp Meta.last_local_run = today

REMOTE TRIGGER — daily 13:00 UTC on Anthropic infra (datacenter IP)
  ├─ Read Meta.last_local_run. If == today → no-op silently.
  └─ Else:
     ├─ Step 1: Alert ingestion (same as local Step 1, remote instance)
     ├─ Step 2: NO SCRAPING (datacenter IPs get gated; Playwright MCP is local-only)
     ├─ Step 3: Build digest email draft (To: fruiteam@gmail.com)
     └─ Step 4: Stamp Meta.last_remote_run = today

APPS SCRIPT — hourly in rvonaar@gmail.com
  └─ Sweep Drafts with subject starting "🏠 London Flat Hunt" → send to fruiteam@gmail.com
```

### Failure coverage matrix

| Day | Primary path | Fallback path | Expected coverage |
|---|---|---|---|
| Weekday, laptop on, Playwright healthy | Local: alerts + scrape | — | 4/4 portals, canonical URLs, full fidelity |
| Weekday, laptop on, Playwright broken | Local: alerts only | Remote fires next (alerts only) | Rightmove only |
| Weekend / travel, laptop off | Remote: alerts only | — | Rightmove only |
| Both broken | — | Weekly health check emails within 2 days | 0 |

## 5. Playwright MCP — install and registration

**Install** (one-time, global user-scope):

```
claude mcp add --scope user playwright -- npx '@playwright/mcp@latest'
```

Already executed and verified on 2026-04-22. `claude mcp list` shows `playwright: npx @playwright/mcp@latest - ✓ Connected`.

**First-run:** Playwright MCP downloads Chromium on first invocation (~150MB, ~1–2 min). Subsequent runs reuse the cached binary.

**Tool surface:** Playwright MCP exposes `playwright__navigate`, `playwright__snapshot`, `playwright__click`, `playwright__evaluate`, `playwright__screenshot`, and a few more. The skill uses primarily `navigate` + `snapshot` (DOM-as-accessibility-tree readout) for URL extraction, and `navigate` + `evaluate` for per-listing field extraction.

**Headless by default.** No Chrome window appears on the laptop. No auto-start, no `.desktop` file, no Chrome keep-alive setting needed — Playwright manages its own Chromium.

## 6. Scraping strategy

### 6.1 Portal search URLs

One saved search per primary area per portal. 8 primary areas × 4 portals = 32 search URLs. Templates in the skill:

- **Rightmove:** `https://www.rightmove.co.uk/property-to-rent/find.html?locationIdentifier=[REGION_CODE]&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6` (sortType=6 is "newest first")
- **Zoopla:** `https://www.zoopla.co.uk/to-rent/flats/[area-slug]/?beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings`
- **SpareRoom:** `https://www.spareroom.co.uk/flats-to-rent/london/[area_slug]?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list`
- **OpenRent:** `https://www.openrent.co.uk/properties-to-rent/london?term=[area]&prices_max=3500&bedrooms_min=1&bedrooms_max=2&isLive=true`

Rightmove region codes are hard-coded in the skill since they're invariant per area (REGION^93965=Islington, REGION^85262=Camden Town, REGION^87500=Clerkenwell, REGION^70393=De Beauvoir Town, REGION^70438=Highbury, REGION^85230=Kentish Town — plus Angel/Canonbury/Highbury roll into Islington/Highbury regions). Other portals' area slugs derive from the human area names.

### 6.2 Search-page scrape loop

For each portal × primary area:

1. `playwright__navigate` to the search URL.
2. `playwright__snapshot` to get the rendered DOM tree.
3. Extract listing URLs via pattern match on the DOM (portal-specific selectors documented in the skill).
4. For each URL, query Notion Flats for `URL == candidate`. If match, it's already in the tracker — mark it as a "dedup hit".
5. **Early-exit requires ≥3 consecutive known listings in newest-first order**, or hitting the 3-page cap (whichever comes first). A single dedup hit mid-page does NOT stop pagination, because mixed ingestion states (an alert may have inserted one listing while peers on the next page are still untracked; a prior scrape may have aborted after CAPTCHA; a laptop-off streak may have left alerts-only coverage with scrape-only listings still on page 2/3) can produce isolated known entries amid new ones.
6. **Per-listing processing is independent of pagination.** Every URL on every visited page is dedup-checked + enriched + scored regardless of the exit condition. Early-exit only determines whether we load the next page, not whether we process the current one.
7. **Cap at 3 pages per portal-area** (prevents runaway pagination on new-run first day or after a long outage).

### 6.3 Listing-page enrichment

For each new URL:

1. `playwright__navigate` to the listing URL.
2. `playwright__evaluate` a portal-specific JavaScript snippet to extract: `title`, `price`, `beds`, `size`, `floor` (where stated), `furnished`, `available from`, `bathtub/wooden floor/renovation hints` from description, photos list.
3. Apply HARD FILTERS (beds, rent, size ≥60m², area, dated/unrenovated). Skip if violated.
4. Score per SCORING section. Insert Notion row with `Source=scraped-local`.

### 6.4 Rate limiting

- 2-second delay between `navigate` calls on the same portal (avoid thundering-herd).
- No cross-portal parallelism. Scrape portals sequentially.
- Total budget: 32 search pages + ~50 listing pages × ~5s each = 6–10 min typical runtime.

### 6.5 CAPTCHA / block handling

If `playwright__snapshot` returns a page whose title or heading matches known CAPTCHA markers (`"Access denied"`, `"Something went wrong"`, `"Just a moment"`, etc.):

- Log the block to the digest email.
- Skip that portal-area for this run.
- Continue with the next portal-area.

No retry loop. A run either yields what it yielded or notes the gap.

### 6.6 Secondary-area scraping cadence

Primary areas (8) scraped every run. Secondary areas (8) scraped only on **Monday runs** — they rarely produce HIGH hits and halve the daily scrape budget.

## 7. Alert ingestion (kept as local Step 1 + remote primary)

Unchanged from the 2026-04-21 spec, with these concrete confirmations from observed behavior:

- **Rightmove alerts:** tracker-wrapped `clicks.rightmove.co.uk/f/a/<token>` URLs in plaintext → follow redirect via `curl -sIL` → canonical `rightmove.co.uk/properties/<id>`. **Working as of 2026-04-22.**
- **Zoopla alerts:** plaintext body contains only `zoopla.co.uk?utm_campaign=...` homepage roots (both instant and daily-digest variants). Yield 0 URLs. Not labeled `hunt-processed` (allows future parser fix). **Scraping path covers Zoopla instead.**
- **SpareRoom / OpenRent alerts:** not yet delivered as of 2026-04-22. Saved searches were set up 2026-04-21. If they never arrive reliably, scraping covers both portals.

The `hunt-processed` Gmail label scheme is preserved — label tools are available again on the Gmail MCP connector as of 2026-04-22.

## 8. Notion schema — no change

All 24 properties from the 2026-04-21 spec plus Human Tier / Human Rationale remain. `Source` enum already includes `scraped-local` as an option; this spec just makes that value the most common one going forward.

**Dedup invariant:** The URL column is the canonical dedup key, but Notion's `url`-type property **does NOT enforce uniqueness at the database level** — it's just a typed string. Dedup correctness therefore relies entirely on the pre-insert query succeeding. See §12 for the fail-closed retry-and-skip behavior on query error.

## 9. Skill-flats.md changes

### SCRAPING section — full rewrite

Replace the current "skip this section in remote mode" stub with detailed Playwright-based instructions (§6 above). Invocation pattern examples:

```
- Call mcp__playwright__navigate with url="https://www.rightmove.co.uk/...sortType=6"
- Call mcp__playwright__snapshot. Parse listing URLs matching `/properties/\d+`.
- For each new URL: mcp__playwright__navigate, then mcp__playwright__evaluate with portal-specific extraction JS.
- If navigate times out or returns a known block page → log and skip.
```

### RUNTIME MODE section

Update to describe the new detection:

- Local (Playwright MCP registered + healthy) → both alert ingestion AND scraping.
- Local (Playwright broken for this run) → alert ingestion only; email digest flags "scraping skipped".
- Remote (datacenter IP) → alert ingestion only; NO scraping attempt.

Detect Playwright availability by testing a single `playwright__navigate` at the start of the scrape step. If it throws, fall through to alerts-only.

### SUCCESS CRITERIA

Add bullets:

- Playwright MCP availability probed; scraping ran (or degradation flagged).
- For each portal, either (a) ≥1 new row added, or (b) an early-exit dedup note, or (c) a CAPTCHA/block note.
- `Source=scraped-local` rows outnumber `Source=email-alert` rows on local-full runs.

### Retire

Remove any mention of:

- "Claude-in-Chrome" as an MCP the skill relies on
- WebFetch-based scraping as the fallback (WebFetch remains for individual listing pages only when Playwright fails specifically on enrichment — rare)

## 10. Remote trigger — update prompt

Update `trig_01St8iTA4T9nkW5iRFbRMp7a` prompt to:

- Remove any residual scraping instructions (explicitly state: "scraping is local-only via Playwright MCP; do not attempt WebFetch scraping of portal search pages").
- Keep alert ingestion as today.
- Keep the Meta check + email draft flow.

## 11. Config / repo changes

- `README.md` Requirements section: replace "Claude in Chrome MCP extension" line with "Playwright MCP (`@playwright/mcp`) registered via `claude mcp add --scope user playwright -- npx '@playwright/mcp@latest'`".
- `tracker/alerts-setup.md`: add a line per portal noting which ones are expected to work via alerts (Rightmove) vs. via scraping (Zoopla, SpareRoom, OpenRent). Makes the spec↔reality map explicit.
- `config.example.md`: no change (nothing Playwright-specific lives in config; it's global MCP registration).

## 12. Error handling

| Scenario | Behavior |
|---|---|
| Playwright MCP not installed / not registered | Scraping step detects absence, logs "Playwright MCP not found — scraping skipped. Install with: claude mcp add --scope user playwright -- npx @playwright/mcp@latest", continues with alerts-only. |
| Playwright launched but Chromium download failed | Same as above; surfaces once the download retries. |
| Portal returns CAPTCHA / block page | Per §6.5: log, skip that portal-area, continue. |
| Navigate times out (>30s) | Same as block: log, skip, continue. |
| Listing page extraction fails for one URL | Create a minimal Notion row with the URL + title and `Needs-verify: extraction failed`. Don't block the rest of the run. |
| Notion dedup query errors | **Fail closed.** Notion does NOT enforce uniqueness on `url`-type properties (unlike a SQL UNIQUE index), so dedup relies entirely on the pre-insert query succeeding. On query error: retry 3× with exponential backoff (1s, 3s, 9s). If still failing, **skip insertion** for this candidate URL and log the skip to the digest email under a "Dedup-check failures" section. Never insert without a successful dedup check. |
| Alert ingestion fails in Step 1 | Log warning, proceed to scraping. |

## 13. Rollout plan

### Phase 1 — Playwright MCP installed and validated (DONE 2026-04-22)

- `claude mcp add --scope user playwright -- npx '@playwright/mcp@latest'` ✓
- Smoke test: fresh `claude -p` stdin run navigates Rightmove Islington search, returns 25 listing URLs ✓

### Phase 2 — Skill update

- Rewrite SCRAPING section in `skill-flats.md` with Playwright pattern.
- Update RUNTIME MODE detection + SUCCESS CRITERIA.
- Update README Requirements, alerts-setup notes.
- Commit.

### Phase 3 — Remote trigger prompt update

- Remove scraping references from `trig_01St8iTA4T9nkW5iRFbRMp7a`.
- Verify via `RemoteTrigger action=get`.

### Phase 4 — Smoke test end-to-end

- Manual `claude -p` run from `~/hunt` exercising the full updated skill.
- Verify: Notion rows with `Source=scraped-local` from multiple portals, `Meta.last_local_run` stamped, email draft to `fruiteam@gmail.com`, Apps Script delivery.

### Phase 5 — Observation (7 days)

- Daily: count per-portal rows, scraping durations, block events.
- Decide: any portals consistently blocked? Any alerts-only path abandoned? Any scoring tweaks needed?

## 14. Success criteria (observable over 7-day observation window)

- `Source=scraped-local` rows ≥ 70% of new rows (confirms scraping is the main source).
- All 4 portals produce ≥1 row on ≥60% of days (catches any portal systematically broken).
- Zero runtime > 20 min (scraping budget held).
- Zero CAPTCHA/block events on Rightmove (we know residential IP works); occasional blocks on Zoopla/SpareRoom tolerated.
- Gmail digest lands in `fruiteam@gmail.com` daily (via local Send or via Apps Script from remote drafts).
- Weekly health-check trigger never emails (both run stamps within 2 days).

## 15. Open questions / deferred

- **Secondary-area scraping** (Monday-only) — may also need weekly-Wednesday or other cadence tuning once we see what those areas yield.
- **CAPTCHA recovery** — if Rightmove or Zoopla starts gating residential IP too (unlikely but possible), we'd revisit Browserbase (Option C from 2026-04-22 brainstorm) or residential proxy services.
- **Alert parser upgrade for Zoopla** — if Gmail MCP later exposes `htmlBody` or we add IMAP bypass, alerts could cover Zoopla too. Low priority while scraping handles it.
- **Browserbase revisit** — only if laptop-off days start meaningfully costing us. Today's assumption is "Rightmove-alerts-only" is an acceptable floor for those days.

---
