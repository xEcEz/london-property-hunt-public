# Layered Ingestion (Alerts + Scraping) — Design Spec

**Date:** 2026-04-21
**Author:** Raphael (rvonaar@gmail.com)
**Status:** Draft — pending user review
**Target repo:** `london-property-hunt-public` (flats variant)
**Supersedes:** scraping-only ingestion defined in the original 2026-04-16 spec

---

## 1. Context

The flats variant currently ingests listings by scraping the four portals' search pages (Rightmove, Zoopla, SpareRoom, OpenRent). Two weeks of real operation exposed reliability problems:

1. **Remote-trigger gating** — Rightmove, Zoopla and SpareRoom return HTTP 403 to WebFetch from Anthropic's datacenter IPs on ~50% of remote runs. On those days the skill pulls only from OpenRent, producing thin results (all LOW tier).
2. **Local-cron fragility** — the laptop isn't always on at 10:00 Zurich (weekends, travel). Recent runs showed "no Chrome MCP this run" footer messages when the Chrome extension wasn't reachable, degrading local runs to WebFetch as well.
3. **Missed days** — 2026-04-18 produced no email at all (laptop off + remote run also failed silently).

Combined effect: ~15% of days yield zero or near-zero inventory, which is costly over a 6-week hunt window.

This spec defines a second ingestion path — **portal email alerts** — and layers it on top of the existing scraping to achieve redundancy + zero-cost reliability.

## 2. Goals

- Add an **alerts ingestion path** that reads portal-generated email alerts from `rvonaar@gmail.com` and writes listings to Notion, bypassing search-page scraping entirely.
- Keep the existing **scraping ingestion path** for local runs (Claude-in-Chrome full fidelity) but make it non-critical.
- Both paths deduplicate against the same Notion URL column — no risk of double insertion.
- Maintain near-zero ongoing cost (no hosted browser, no third-party scraper API).
- Cover ≥85% of inventory via alerts alone, so coverage no longer depends on laptop uptime.

## 3. Non-goals

- Browserbase or other hosted-browser services (deferred; revisit if alerts + local scraping still leave gaps).
- Rewriting the scoring model, Notion schema, or email digest format (unchanged).
- Real-time push (sub-minute) delivery — alert ingestion runs on the existing cron cadence.
- Switching to a different tracker (Notion stays).

## 4. User profile

Raphael. Flat hunt criteria unchanged from the original 2026-04-16 spec (1–2 bed, ≤ £3,500, ≥ 60 m², near King's Cross, scored on top-floor / new-build / calm / bathtub / light / wooden-floor). Laptop reliability: weekdays-only (~70% uptime at 10:00 Zurich).

## 5. Architecture

```
Rightmove alerts ┐
Zoopla alerts    ├─→ rvonaar@gmail.com inbox ─→ alert parser ─┐
SpareRoom alerts │                                             │
OpenRent alerts  ┘                                             ├─→ dedup (Notion URL) ─→ fetch listing page ─→ score ─→ write to Flats
                                                               │
laptop Chrome-in-ext (local only) ─→ search-page scrape ───────┘
```

Two independent ingestion paths feed the same downstream pipeline:

- Alert parser reads the Gmail inbox, extracts listing URLs from portal alert emails.
- Search-page scrape (local only) uses the existing 4-portal URL list.

Both converge on the same dedup check (Notion URL column) before writing to the `Flats` database. This is the canonical source of truth for "has this listing been seen before."

## 6. Alert ingestion

### 6.1 Gmail query

Skill queries via `mcp__claude_ai_Gmail__search_threads`:

```
(from:(@rightmove.co.uk) OR from:(@zoopla.co.uk) OR from:(@spareroom.co.uk) OR from:(@openrent.co.uk)) AND newer_than:2d AND -label:hunt-processed
```

Rationale:

- **Sender-domain filter** catches 95%+ of alert emails (portals use consistent `@<portal-domain>` sender addresses for transactional mail).
- **`-label:hunt-processed`** excludes already-handled alerts, letting the skill re-label after processing and avoid reprocessing on the next run.
- **`newer_than:2d`** catches weekend backlog when Monday's run is the first to fire after a local laptop-off period.

### 6.2 URL extraction

For each matched email:

1. Fetch the full HTML body via `mcp__claude_ai_Gmail__get_thread`.
2. Extract candidate listing URLs by matching the portal's canonical listing URL patterns:
   - Rightmove: `https://www.rightmove.co.uk/properties/<id>` (or `#/...`)
   - Zoopla: `https://www.zoopla.co.uk/to-rent/details/<id>/`
   - SpareRoom: `https://www.spareroom.co.uk/flatshare/flatshare_detail.pl?flatshare_id=<id>` or `/flats-to-rent/...`
   - OpenRent: `https://www.openrent.co.uk/property-to-rent/...` with numeric ID
3. De-redirect portal tracking links (e.g. Rightmove wraps outbound with `rightmove.co.uk/redirect?...`) by following to the final URL.
4. Dedup URLs within the same email (alerts sometimes list the same listing twice).

### 6.3 Label-after-handling

After a successful alert is processed (URL extracted, dedup-checked, listing fetched or skipped-with-flag):

- Apply Gmail label `hunt-processed` to the thread via `mcp__claude_ai_Gmail__label_thread`.
- If the label doesn't exist, the skill creates it via `mcp__claude_ai_Gmail__create_label` on first run.

Errors during labeling are non-fatal — the URL is already in Notion (dedup handles reprocessing).

### 6.4 Individual listing page fetch

For each non-dedup URL:

- **Local mode:** use Claude-in-Chrome MCP for full-fidelity page (photos, floorplan).
- **Remote mode:** use WebFetch on the individual listing URL. Individual pages are gated less aggressively than search pages (empirically), but some gating still occurs.
- **On fetch failure:** still create the Notion row using whatever data the alert email provided (title, price, beds, URL) plus `Size Source=unknown` and `Needs-verify: listing page unreachable`. Better a minimal row than losing the URL (which breaks future dedup).

### 6.5 Scoring

Same model as before (§8 of the original spec). Signals are inferred from whatever data is available — alert emails include title, price, beds, area; photos and floorplan come from the listing page fetch. If a signal cannot be determined, default to `Unknown`.

## 7. Scraping ingestion (unchanged, now local-only)

- Runs only when the local cron fires and Claude-in-Chrome MCP is available.
- Skips entirely in remote mode (trigger prompt already instructs this).
- Uses the existing 4-portal search-URL list from config.
- Writes to Notion via the same dedup + scoring pipeline.

## 8. Notion schema changes

Add one property to the `Flats` database:

| Property | Type | Values |
|---|---|---|
| Source | select | `email-alert`, `scraped-local`, `scraped-remote` |

Rationale: lets the operator see which ingestion path found each listing, diagnose whether one path is systematically missing inventory, and make informed decisions about whether to retire scraping in Phase 2.

No other schema changes. Existing 23 properties + Human Tier + Human Rationale all stay.

## 9. Skill-flats.md changes

### New section: `## ALERT INGESTION`

Inserted between `## PLATFORMS` (renamed to `## SCRAPING`) and `## TRACKER`. Contains the Gmail query from §6.1, URL extraction rules from §6.2, label-after-handling from §6.3, fetch behavior from §6.4.

### Renamed section: `## PLATFORMS` → `## SCRAPING`

Adds a one-line prefix: "Local mode only. In remote WebFetch-only mode, skip this section entirely — rely on alert ingestion instead."

### Modified: TRACKER section

Add `Source` to the property list. Note: "Alert-ingested rows set Source=email-alert. Scraping-ingested rows set Source=scraped-local or scraped-remote depending on runtime mode."

### Modified: SUCCESS CRITERIA

Add:
- "Alert emails queried and URLs extracted where available"
- "Processed alerts labeled `hunt-processed` in Gmail"
- "No duplicate Notion rows (URL dedup holds across both paths)"

### Modified: RUNTIME MODE

Update detection to prefer alerts regardless of scraping availability:
- Always run alert ingestion first.
- Run scraping ingestion second, only if Claude-in-Chrome is available and runtime is local.

## 10. Remote trigger prompt changes

Update `trig_01St8iTA4T9nkW5iRFbRMp7a` (remote fallback) to:

- Query Gmail alerts via the query in §6.1.
- Process and label per §6.3–6.4.
- **Skip scraping entirely** (already the case, reinforce).
- `Source=email-alert` on all new rows.
- Fetch listing pages via WebFetch; tolerate 403s by recording minimal rows.

## 11. Config changes

No changes to `config.md`. Alert ingestion uses fixed Gmail query logic in the skill; portal-specific URL patterns are also embedded in the skill prompt (not config-driven, since they're invariant per portal).

Optional future addition (not in MVP): `FLAT_ALERT_MAX_AGE_DAYS=2` to make the `newer_than:` window configurable.

## 12. Portal saved-search setup (user, one-time)

For each of the 4 portals:

1. Sign in at `rvonaar@gmail.com`.
2. Create a saved search matching the flat hunt criteria (1–2 bed, max £3,500, areas, unfurnished preferred where supported).
3. Turn on **instant email alerts** (not daily digest — instant gives faster detection).
4. Verify that a test alert email lands in `rvonaar@gmail.com` Gmail with a sender matching `@<portal-domain>`.

Expected total time: ~20 min across all 4 portals.

Setup details per portal are added to `tracker/alerts-setup.md` (new file) as part of the implementation.

## 13. Error handling

| Scenario | Behavior |
|---|---|
| Gmail MCP unavailable | Log warning, skip alert ingestion, continue with scraping-only if available. Still send daily email. |
| Individual listing page 403 | Record minimal row (title/price/beds/URL from alert), set `Size Source=unknown`, `Needs-verify: listing page unreachable`, `Source=email-alert`. |
| No matching alerts in 2-day window | Alert ingestion produces 0 rows. Scraping (if available) still runs. Email digest sent as normal. |
| Alert email with no extractable URL | Skip the email, do NOT label it as processed — so a next run could retry if the email content changes. |
| Label creation fails (first run) | Log error, continue without labeling. Next run will retry. Duplicate-processing is prevented by URL dedup. |
| URL already in Notion (dedup hit) | Skip insert. If `Source` differs from the stored value, prefer the earliest-seen value (do not overwrite). |

## 14. Rollout

### Phase 1 (MVP — this spec)

- Implement alert ingestion in `skill-flats.md`.
- Add `Source` property to Notion Flats DB.
- Update remote trigger prompt.
- User sets up saved-search alerts on all 4 portals.
- Run for 7 days, observe:
  - Daily counts per Source
  - Days with gating events
  - Days with laptop off
  - Missing-day rate

### Phase 2 (optional, post-week-1)

- If alerts cover ≥95% of inventory over the observation window, consider retiring local scraping or reducing to weekly sweeps.
- If scraping adds ≥20% unique inventory, keep both paths permanently.
- Tune `newer_than:` window if weekend backlog causes issues.
- Re-examine Browserbase if remote alert-to-listing-page fetches get gated meaningfully.

## 15. Repository changes

| File | Action |
|---|---|
| `skill-flats.md` | Modify — add ALERT INGESTION section, rename PLATFORMS→SCRAPING, update TRACKER, SUCCESS CRITERIA, RUNTIME MODE |
| `tracker/flats-schema.md` | Modify — document the new `Source` property |
| `tracker/alerts-setup.md` | Create — per-portal saved-search instructions |
| `README.md` | Modify — brief mention of alert ingestion in the flats-variant scheduling section |
| `config.example.md` | No change (MVP) |

Personal updates (not in repo):
- Raphael authenticates his 4 portal accounts and sets up saved searches
- Remote trigger prompt at `trig_01St8iTA4T9nkW5iRFbRMp7a` updated via `RemoteTrigger action=update`
- Notion Flats DB schema updated via `notion-update-data-source ADD COLUMN "Source" SELECT(...)`

## 16. Success criteria (observable)

- Over a rolling 7-day window after rollout:
  - Zero days with zero new rows in the Flats DB (unless legitimately no listings matched criteria).
  - ≥60% of new rows have `Source=email-alert` (confirms the primary path is active).
  - Average ≥10 unique new rows per day (maintains or exceeds current baseline).
- Gating independence: at least one day in the observation window where remote scraping would have been gated and alerts still delivered new rows.
- No duplicate URLs in Notion (dedup integrity holds across both paths).

## 17. Open questions / deferred

- Real-time triggering: Rightmove "instant alerts" are actually still batched at ~15-min intervals. If Raphael wants sub-minute latency, we'd need Gmail Push notifications + a webhook + always-on infra. Deferred — current cron cadence is adequate for a flat hunt.
- Alert email parsing robustness: portal alert templates change occasionally. If parsing breaks, rows will show `Size Source=unknown` and the skill will flag it via `Needs-verify`. Plan: monitor for a month, add regex patterns to the skill prompt if drift emerges.
- OpenRent alerts vs. scraping: OpenRent is the one portal that doesn't gate WebFetch. Arguably we could keep scraping it and skip its alerts. MVP keeps both for simplicity; revisit in Phase 2.
