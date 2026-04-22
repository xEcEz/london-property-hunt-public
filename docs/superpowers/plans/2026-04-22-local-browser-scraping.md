# Local Browser Scraping (Playwright MCP) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restore full-fidelity scraping as the primary local ingestion path for the flats-variant skill, driving all four portals via Playwright MCP with the laptop's residential IP, while keeping alert ingestion as a local supplement and the sole remote-trigger path.

**Architecture:** Local cron runs alert ingestion (~30s) then Playwright-driven scraping of all 4 portals (~10–15 min). Both paths dedup via the Notion URL column. Remote trigger stays alerts-only (datacenter IP can't scrape). Codex adversarial-review findings are incorporated: dedup query failures retry 3× with exponential backoff then fail-closed, and pagination early-exit requires ≥3 consecutive known listings in newest-first order (not just one) to avoid under-ingestion in mixed-state scenarios.

**Tech Stack:**
- Claude Code skill prompt (markdown)
- Playwright MCP (`@playwright/mcp@latest`, Microsoft official) — already registered user-scope on 2026-04-22
- Gmail MCP (read threads, create drafts, apply labels)
- Notion MCP (create pages, search, update data source, update pages)
- `curl` / `HEAD` for Rightmove tracker URL redirect-follow in alert ingestion
- Crontab + `/home/ara/.local/bin/claude` CLI

**Source spec:** [`../specs/2026-04-22-local-browser-scraping-design.md`](../specs/2026-04-22-local-browser-scraping-design.md)

**Branch:** currently on `main`. All Phase 1 changes are markdown edits to `skill-flats.md`, `README.md`, and `tracker/alerts-setup.md`. Safe to commit directly to `main`, or create a branch (`local-scraping-v2`) for PR-able history.

---

## Phase 1 — Skill and documentation updates (agent-executed)

### Task 1: Rewrite the `## SCRAPING` section in `skill-flats.md`

**Files:**
- Modify: `skill-flats.md`

- [ ] **Step 1: Find and remove the current `## SCRAPING` section**

In `skill-flats.md`, find the section starting with `## SCRAPING (local mode only)`. It currently contains the short "Skip this section entirely in remote WebFetch-only mode" note plus 4 platform URL templates. Delete the entire section from `## SCRAPING (local mode only)` up to (but not including) the next `## TRACKER` heading.

- [ ] **Step 2: Insert the new `## SCRAPING` section in the same location**

Paste this full section in place of what was deleted (immediately before `## TRACKER`):

```
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
3. Detect block page: if the title or any heading matches `/Access denied|Just a moment|Something went wrong|unusual activity|403 Forbidden/i`, log "Block detected on {portal}/{area}" to the digest, mark this portal-area as blocked, and skip to the next portal-area. No retry.
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
     - SpareRoom / OpenRent: fall back to querying specific `meta` tags and visible DOM text (each portal documents its layout inline).
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
```

- [ ] **Step 3: Verify the section**

Read `skill-flats.md` and confirm:
- `## SCRAPING (local mode only, Playwright MCP)` section exists between SCORING and TRACKER
- Section mentions `mcp__playwright__navigate`, `mcp__playwright__snapshot`, `mcp__playwright__evaluate`
- "3 consecutive known listings" appears verbatim
- "fail-closed" appears verbatim
- No `[PLACEHOLDER]` references that aren't in the placeholder docs table at the bottom

- [ ] **Step 4: Commit**

```bash
cd /home/ara/dev/london-flat-hunt/london-property-hunt-public
git add skill-flats.md
git commit -m "skill(flats): rewrite SCRAPING section for Playwright MCP

Replace the terse 'skip in remote' stub with the full Playwright-based
scraping playbook. Portal URL templates, per-page extraction rules,
3-consecutive-known-listings early-exit cutoff, fail-closed dedup
with 1s/3s/9s exponential backoff, secondary-area Monday cadence.

Per 2026-04-22 spec and Codex adversarial review fixes."
```

---

### Task 2: Update RUNTIME MODE and SUCCESS CRITERIA in `skill-flats.md`

**Files:**
- Modify: `skill-flats.md`

- [ ] **Step 1: Replace the `## RUNTIME MODE` section**

Find the current `## RUNTIME MODE` section (it starts with "Always run alert ingestion first" and describes alert + conditional scraping). Replace the whole section with:

```
## RUNTIME MODE

Two ingestion paths — alert ingestion (§ALERT INGESTION) always runs first in every mode; scraping (§SCRAPING) runs only when the runtime can reach Playwright.

**Local mode (laptop cron, residential IP):**
- Run alert ingestion.
- Probe Playwright via `mcp__playwright__navigate` to `https://example.com`. If it responds within 30s with no error, run § SCRAPING for all 4 portals. If it fails, log "Playwright unavailable — scraping skipped" and continue.

**Remote mode (Anthropic-hosted trigger, datacenter IP):**
- Run alert ingestion.
- **SKIP § SCRAPING entirely.** Remote environments have no Playwright MCP; WebFetch from datacenter IPs is gated on Rightmove/Zoopla/SpareRoom search pages. Alert ingestion is the only remote input source.

Detect remote mode by the absence of Playwright MCP tools (probe as above and catch the "server not available" error pattern).
```

- [ ] **Step 2: Replace the `## SUCCESS CRITERIA` section**

Find the current SUCCESS CRITERIA bullets. Replace with:

```
- Alert emails queried for the target sender domains in the last 2 days (labeled threads excluded via `-label:hunt-processed`)
- Fully-processed alert threads labeled `hunt-processed` in Gmail
- Playwright MCP availability probed at the start of § SCRAPING. If unavailable, digest email footer includes a `⚠️ Scraping skipped (Playwright unavailable)` line.
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

- [ ] **Step 3: Verify**

Confirm in `skill-flats.md`:
- RUNTIME MODE now probes Playwright, distinguishes local (scrape) from remote (no scrape).
- SUCCESS CRITERIA mentions "Dedup-check failures", "3 consecutive known listings", "Monday secondary areas".

- [ ] **Step 4: Commit**

```bash
git add skill-flats.md
git commit -m "skill(flats): update RUNTIME MODE + SUCCESS CRITERIA for Playwright

Runtime mode now probes Playwright at the start of scraping and
degrades gracefully when unavailable. Success criteria reflect
the new scrape behavior (early-exit threshold, dedup failure
surfacing, Monday secondary cadence)."
```

---

### Task 3: Update Requirements in `README.md`

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace the Requirements section**

Find the `## Requirements` section. It currently lists Claude Code, Claude in Chrome, Gmail MCP, Notion MCP (flats), Python + openpyxl (rooms). Replace the entire bullet list with:

```
- **Claude Code** (CLI or desktop app) — [claude.ai/code](https://claude.ai/code)
- **Playwright MCP** — *flats variant only* — for residential-IP browser scraping. Install with `claude mcp add --scope user playwright -- npx '@playwright/mcp@latest'`. Downloads its own Chromium on first use (~150MB). Runs headless — no user-facing browser window required.
- **Gmail MCP connector** — for inbox reading (alert ingestion), draft creation, and `hunt-processed` labeling.
- **Notion MCP connector** — *flats variant only* — for the Notion-based tracker (full CRUD).
- **Python 3 + openpyxl** — *rooms variant only* — for xlsx spreadsheet updates (`pip install openpyxl`).
- A Gmail / Google account you can grant MCP access to.
```

- [ ] **Step 2: Verify**

Read `README.md` and confirm:
- Requirements mentions `claude mcp add --scope user playwright`.
- "Claude in Chrome MCP extension" no longer appears in the Requirements list.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs(readme): replace Claude-in-Chrome requirement with Playwright MCP

Claude-in-Chrome was a claude.ai browser extension, never actually a
Claude Code CLI MCP. Replaced with Playwright MCP which bundles its
own Chromium and is the real local-scraping path."
```

---

### Task 4: Update `tracker/alerts-setup.md` with reality notes

**Files:**
- Modify: `tracker/alerts-setup.md`

- [ ] **Step 1: Add a "What to expect per portal" section**

In `tracker/alerts-setup.md`, immediately after the initial "All four saved searches should match the criteria from your `config.md`" paragraph (and its bullet list), insert this new section:

```

## What to expect per portal

Not every portal's alert email is ingestible by the skill. Based on 6 days of observation (2026-04-16 to 2026-04-22):

- **Rightmove** — ✅ Alerts work. Plaintext contains tracker-wrapped `clicks.rightmove.co.uk/f/a/<token>` URLs which resolve to canonical `rightmove.co.uk/properties/<id>` via HTTP redirect. Skill follows the redirect with `curl -sIL` and extracts the canonical URL.
- **Zoopla** — ❌ Alerts NOT ingestible. Both instant and daily-digest plaintext templates contain only `zoopla.co.uk?utm_campaign=...` homepage roots. Per-listing URLs live in the HTML part, which the Gmail MCP does not expose. **Zoopla is covered by scraping instead.** Keep saved searches on for human inbox triage if you like.
- **SpareRoom** — ⚠️ Saved-search alerts may be paid-plan only; instant alerts have not delivered in testing. **SpareRoom is covered by scraping.**
- **OpenRent** — ⚠️ Saved-search alerts have not delivered in testing (may be a delivery or spam-filter issue). **OpenRent is covered by scraping** — and as the only non-gated portal for WebFetch, the remote trigger can cover OpenRent listings too if alerts ever start flowing.

Bottom line: alerts are a *bonus* freshness channel. Scraping (Playwright MCP) is the authoritative source for inventory. You still benefit from having portal alerts on — they land in your inbox before the next scrape, so you have the option to act within minutes on a hot listing that the scheduled scrape would have picked up the next morning.
```

- [ ] **Step 2: Verify**

Read `tracker/alerts-setup.md` and confirm:
- `## What to expect per portal` section exists.
- Mentions `clicks.rightmove.co.uk`, `zoopla.co.uk?utm_campaign`, and "scraping (Playwright MCP) is the authoritative source".

- [ ] **Step 3: Commit**

```bash
git add tracker/alerts-setup.md
git commit -m "tracker: add per-portal alert reality notes

Document what 6 days of operation revealed: Rightmove alerts work,
Zoopla plaintext has no per-listing URLs, SpareRoom and OpenRent
alerts have not delivered. Scraping is the authoritative source;
alerts are a bonus freshness channel."
```

---

### Task 5: Final Phase 1 verification

- [ ] **Step 1: List recent commits**

```bash
cd /home/ara/dev/london-flat-hunt/london-property-hunt-public
git log --since='2 hours ago' --oneline
```

Expected: 4 Phase-1 commits (SCRAPING rewrite, RUNTIME MODE/SUCCESS CRITERIA, README, alerts-setup).

- [ ] **Step 2: Confirm working tree clean**

```bash
git status --short
```

Expected: no output (working tree clean).

- [ ] **Step 3: Cross-check skill-flats.md placeholders**

```bash
grep -oE '\[FLAT_[A-Z_]+\]' skill-flats.md | sort -u
```

Expected: every `[FLAT_*]` placeholder referenced in the new SCRAPING section is in this list. Cross-check against `config.example.md`:

```bash
grep -oE '^FLAT_[A-Z_]+' config.example.md | sort -u
```

Every `[FLAT_*]` in skill-flats.md must appear as a `FLAT_*=...` line in config.example.md. Any mismatch → fix in skill-flats.md or add to config.example.md.

- [ ] **Step 4: Cross-check Playwright MCP tool names**

```bash
grep -oE 'mcp__playwright__[a-z_]+' skill-flats.md | sort -u
```

Expected tools: `mcp__playwright__navigate`, `mcp__playwright__snapshot`, `mcp__playwright__evaluate`. Confirm these are available in the current Playwright MCP (the skill will error at runtime if a referenced tool doesn't exist).

---

## Phase 2 — Remote trigger prompt update (agent-executed)

### Task 6: Update remote trigger prompt to remove scraping references and point at the new skill

- [ ] **Step 1: Fetch current prompt**

Call `RemoteTrigger action:"get"` with `trigger_id: trig_01St8iTA4T9nkW5iRFbRMp7a`. Copy the CONFIG block verbatim (unchanged) for reuse in Step 2.

- [ ] **Step 2: Update with new prompt**

Call `RemoteTrigger action:"update"` with `trigger_id: trig_01St8iTA4T9nkW5iRFbRMp7a` and body containing this prompt (substituting the CONFIG block from Step 1 where marked):

```
You are the remote fallback runner for Raphael's London flat hunt (1–2 bed whole-unit rentals near King's Cross).

Read `skill-flats.md` from the repo root — that is your full skill prompt. All `[PLACEHOLDER]` values are filled from the config below.

## CONFIG

[paste CONFIG block from Step 1 verbatim — all YOUR_* and FLAT_* values]

## ALERT INGESTION (sole remote input source)

1. Query Gmail via mcp__claude_ai_Gmail__search_threads with: (from:(rightmove.co.uk) OR from:(zoopla.co.uk) OR from:(spareroom.co.uk) OR from:(openrent.co.uk) OR from:(openrent.com)) AND newer_than:2d AND -label:hunt-processed
2. For each thread returned, fetch full body via mcp__claude_ai_Gmail__get_thread. Gmail MCP only exposes plaintextBody.
3. Extract listing URLs by matching portal patterns:
   - Rightmove: tracker-wrapped URLs look like `clicks.rightmove.co.uk/f/a/<token>`; follow via `curl -sIL` and extract the canonical `rightmove.co.uk/properties/<id>` from the redirect chain.
   - Zoopla: per-listing URLs are HTML-only in plaintext alerts (plaintext contains only `zoopla.co.uk?utm_campaign=...` homepage roots). For Zoopla threads with zero extractable URLs, do NOT label `hunt-processed` — allows future retry once parser supports HTML or plaintext fallback.
   - SpareRoom: `/flatshare/flatshare_detail.pl?flatshare_id=<id>` or `/flats-to-rent/...`
   - OpenRent: `/property-to-rent/...` with numeric id
4. For each extracted URL:
   a. Query Notion Flats data source (id c417b1a4-2ebd-4686-85ec-ca1219a8b0d4) via mcp__claude_ai_Notion__notion-search for URL == candidate. If query errors, retry 3x with exponential backoff (1s, 3s, 9s); if still failing, SKIP this URL and log to digest "Dedup-check failures" section. Never insert without a successful dedup check — Notion does NOT enforce URL uniqueness at the database level.
   b. Otherwise WebFetch the individual listing page.
   c. Apply HARD FILTERS (beds in 1–2, rent ≤ £3500, area in primary∪secondary, stated size ≥ 60m², not obviously dated). If violated → skip.
   d. If page fetch succeeded and filters pass → score per skill-flats.md's SCORING, create Notion row with Source=email-alert, Status=NEW, Found On=today (Europe/Zurich YYYY-MM-DD).
   e. If page fetch failed but alert-email data alone did not violate hard filters → create a minimal row (title, price, beds, URL). Set Size Source=unknown, Needs-verify="listing page unreachable", Source=email-alert. Leave Score and Tier blank.
5. Label the thread `hunt-processed` via mcp__claude_ai_Gmail__label_thread after processing. Create the label via mcp__claude_ai_Gmail__create_label on first run if list_labels doesn't return it. Labeling is non-fatal.
6. Do NOT label threads with zero extractable URLs (e.g. current Zoopla plaintext-only template) — allows future retry.

## NO SCRAPING IN REMOTE MODE

The flats-variant skill's § SCRAPING section is local-only — it requires Playwright MCP which only runs on Raphael's laptop. Do NOT attempt WebFetch of portal search pages from this remote environment. The WebFetch-fetch of individual listing pages in step 4b above is the only enrichment path available here.

## META + EMAIL

1. Query the Notion Meta database (data source id e914a179-d983-4730-b863-97d188843f90) for Key == "last_local_run". Read Value.
2. Compute today's date in Europe/Zurich as YYYY-MM-DD.
3. If Meta.last_local_run.Value equals today's date → do NOT create an email draft. Still update Meta.last_remote_run. Exit without emailing.
4. Otherwise compose and create a Gmail draft via mcp__claude_ai_Gmail__create_draft per skill-flats.md's EMAIL section. Subject: 🏠 London Flat Hunt — {DATE} — {N} new. To: fruiteam@gmail.com. No Cc.
5. Update Meta last_remote_run = today's date (Europe/Zurich, YYYY-MM-DD).

## NOTES

- Flats database has Human Tier and Human Rationale columns for reviewer use — DO NOT set these on new rows.
- Outreach: inline in email cards (no .txt files in remote mode).
- Apps Script in rvonaar@gmail.com auto-sends drafts matching the subject — do NOT attempt alternative send paths.
- When creating Notion pages for new listings, pass parent {type: "data_source_id", data_source_id: "c417b1a4-2ebd-4686-85ec-ca1219a8b0d4"} and set all applicable properties per the Flats schema including Source.
```

- [ ] **Step 3: Verify**

Call `RemoteTrigger action:"get"` for `trig_01St8iTA4T9nkW5iRFbRMp7a` and confirm the prompt contains:
- `## ALERT INGESTION (sole remote input source)`
- `## NO SCRAPING IN REMOTE MODE`
- `retry 3x with exponential backoff`
- `Notion does NOT enforce URL uniqueness`

---

## Phase 3 — Local environment cleanup (user-executed, optional)

### Task 7: Remove obsolete Chrome keep-alive autostart

- [ ] **Step 1: Remove the autostart desktop entry**

```bash
rm /home/ara/.config/autostart/google-chrome-keepalive.desktop
```

Playwright MCP manages its own headless Chromium. The manually-autostarted Chrome was only needed for the never-wired-up "Claude in Chrome" extension scheme — no longer required.

- [ ] **Step 2: (Optional) Disable "Continue running background apps when Chrome is closed"**

Chrome's `chrome://settings/system` → turn OFF "Continue running background apps when Google Chrome is closed" if you no longer want Chrome lingering in the background.

- [ ] **Step 3: (Optional) Restore or leave the crontab env vars**

The crontab still exports `DISPLAY=:0`, `DBUS_SESSION_BUS_ADDRESS=...`, `XDG_RUNTIME_DIR=...`, `WAYLAND_DISPLAY=wayland-0`. Playwright MCP doesn't need these (it runs headless in its own Chromium subprocess). They're harmless. If you want a clean crontab:

```bash
crontab -e
# Remove the DISPLAY, DBUS_SESSION_BUS_ADDRESS, XDG_RUNTIME_DIR, and WAYLAND_DISPLAY lines
# Keep CRON_TZ, HOME, PATH, and the cron entry itself.
```

---

## Phase 4 — Smoke test and observation (agent + user)

### Task 8: Manual smoke test end-to-end

- [ ] **Step 1: Append a marker to the cron log**

```bash
echo "" >> /home/ara/hunt/cron.log
echo "=== MANUAL SMOKE TEST STARTED $(date -u +%Y-%m-%dT%H:%M:%SZ) (post-Playwright rewrite) ===" >> /home/ara/hunt/cron.log
```

- [ ] **Step 2: Invoke the skill manually**

```bash
cd /home/ara/hunt && /home/ara/.local/bin/claude -p < <(echo "Run the London flat hunt skill (local mode)") >> /home/ara/hunt/cron.log 2>&1
```

Expected runtime: 8–15 min (alert ingestion + full Playwright scrape of 6 Rightmove + 8 Zoopla + 8 SpareRoom + 1 OpenRent URLs, plus per-listing enrichment).

- [ ] **Step 3: Append the finish marker**

```bash
echo "=== MANUAL SMOKE TEST FINISHED $(date -u +%Y-%m-%dT%H:%M:%SZ) ===" >> /home/ara/hunt/cron.log
```

- [ ] **Step 4: Inspect run summary**

```bash
awk '/=== MANUAL SMOKE TEST STARTED/{flag=1} flag' /home/ara/hunt/cron.log | tail -60
```

Success signals:
- Playwright availability probe succeeded.
- At least one portal returned listings (most likely Rightmove; Zoopla / SpareRoom / OpenRent behavior is what we're validating).
- Notion Flats has new rows with `Source=scraped-local` and `Found On=<today>`.
- `Meta.last_local_run` stamped to today.
- Gmail draft created with subject `🏠 London Flat Hunt — <today> — N new (H:n/M:n/L:n)` and To: fruiteam@gmail.com.

Failure signals:
- "Playwright unavailable" footer → Playwright install or registration issue; revisit `claude mcp list`.
- "Block detected on Rightmove/Islington" or similar → residential IP unexpectedly gated; revisit rate limiting or try a different portal-area to confirm.
- "Dedup-check failures" section present → Notion MCP issues; check connector health with `claude mcp list`.

- [ ] **Step 5: Cross-verify via Notion + Gmail MCPs from this session**

Call `mcp__claude_ai_Notion__notion-search` with `query=<today's date>`, `data_source_url=collection://c417b1a4-2ebd-4686-85ec-ca1219a8b0d4`, `page_size=25`. Expect ≥1 `Source=scraped-local` page with today's Found On.

Call `mcp__claude_ai_Gmail__list_drafts` with `query=subject:"London Flat Hunt" newer_than:1d`. Expect a draft with today's subject and To: fruiteam@gmail.com.

### Task 9: 7-day observation

- [ ] **Step 1: Daily check, for 7 consecutive days, log:**
  - Per-Source row count (scraped-local vs email-alert).
  - Per-portal new-row count (filter Notion by Platform).
  - Any "Block detected" or "Dedup-check failures" notes in the digest.
  - Daily runtime (from cron.log MANUAL RUN STARTED/FINISHED timestamps).

- [ ] **Step 2: Success thresholds (after 7 days):**
  - `Source=scraped-local` ≥ 70% of new rows.
  - All 4 portals produce ≥1 row on ≥60% of days.
  - Zero runs exceeding 20 min.
  - Zero CAPTCHA/block events on Rightmove (residential IP should work).
  - Gmail digest lands in fruiteam@gmail.com daily.

- [ ] **Step 3: If thresholds miss:**
  - Scraping contributes <70% → investigate which portal is failing; tune selectors, pagination depth, rate limiting.
  - A portal blocks consistently → Browserbase MCP becomes worth the $40/mo.
  - Runtime consistently over 20 min → reduce pagination cap to 2 pages, or move secondary areas to weekly.
