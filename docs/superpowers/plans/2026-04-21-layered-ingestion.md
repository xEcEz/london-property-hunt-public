# Layered Ingestion (Alerts + Scraping) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an email-alerts ingestion path alongside the existing scraping in the flats variant of the london-property-hunt-public skill, giving redundancy against portal anti-bot gating and local-cron unavailability.

**Architecture:** Two ingestion paths feed the same Notion Flats DB. Path A (alerts) reads portal saved-search emails from `rvonaar@gmail.com` via Gmail MCP, extracts listing URLs, fetches listing pages, scores, dedupes on URL, writes to Notion. Path B (scraping, local-only) remains unchanged. A new `Source` property on Flats tracks which path found each row.

**Tech Stack:**
- Claude Code skill prompts (markdown)
- Gmail MCP (`mcp__claude_ai_Gmail__*`) for inbox reading + labeling
- Notion MCP (`mcp__claude_ai_Notion__*`) for database writes
- Claude-in-Chrome MCP (local only, unchanged)
- WebFetch for listing-page retrieval

**Source spec:** [`../specs/2026-04-21-layered-ingestion-design.md`](../specs/2026-04-21-layered-ingestion-design.md)

**Branch:** currently on `main`. All Phase 1 changes are additive markdown/schema edits — safe to commit directly to `main`, or create an isolation branch (`layered-ingestion`) if you prefer PR-able history.

---

## Phase 1 — Repo: skill and docs (agent-executed)

### Task 1: Update `skill-flats.md` — add ALERT INGESTION, rename PLATFORMS, update related sections

**Files:**
- Modify: `skill-flats.md`

- [ ] **Step 1: Replace the `## RUNTIME MODE` section**

Find the current `## RUNTIME MODE` section in `skill-flats.md` (currently 4 lines after the `## Skill prompt (paste into Claude Code)` fenced block starts). Replace it entirely with:

```
## RUNTIME MODE

Always run alert ingestion first (§ALERT INGESTION below). It works in both local and remote modes.

After alerts are processed, conditionally run scraping:
- Local (Claude-in-Chrome MCP available) → use browser MCP for Rightmove / Zoopla / SpareRoom / OpenRent search pages (full fidelity: photos, floorplans).
- Remote (WebFetch only, no Claude-in-Chrome) → SKIP scraping. Alert ingestion is your only source.

Detect runtime mode by checking if Claude-in-Chrome MCP tools are available. If not → WebFetch mode, skip scraping.
```

- [ ] **Step 2: Insert a new `## ALERT INGESTION` section between `## SCORING` and `## PLATFORMS`**

Insert immediately after the line `**Cap:** size-unknown 1-beds (rule 4 above) → tier cannot exceed MEDIUM even if score ≥ HIGH threshold.` (end of SCORING) and before `## PLATFORMS`:

```
## ALERT INGESTION (runs first, both local and remote)

Read portal saved-search alert emails from Raphael's Gmail and convert them into Flats rows.

### Gmail query

Call `mcp__claude_ai_Gmail__search_threads` with:

```
(from:(@rightmove.co.uk) OR from:(@zoopla.co.uk) OR from:(@spareroom.co.uk) OR from:(@openrent.co.uk)) AND newer_than:2d AND -label:hunt-processed
```

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
3. If page fetch succeeds → extract all available signals per § SCORING, score, create Notion row with `Source=email-alert`.
4. If page fetch fails (403, timeout, any error) → still create a Notion row using data from the alert email (title, price, beds, URL). Set `Size Source=unknown`, `Needs-verify: listing page unreachable`, `Source=email-alert`. Skip scoring (leave `Score` and `Tier` empty).

### Label-after-handling

After each successfully-processed thread (whether all URLs led to new rows or all were dedup-skipped), call `mcp__claude_ai_Gmail__label_thread` with label `hunt-processed`. On first run, if the label doesn't exist, create it via `mcp__claude_ai_Gmail__create_label`.

Errors during labeling are non-fatal — URL dedup handles the duplicate-processing case.

### When to skip

If an alert email has zero extractable URLs (e.g. marketing template), do NOT label it. A future run may retry if the email content changes. Log the skip but don't fail.

```

- [ ] **Step 3: Rename `## PLATFORMS` to `## SCRAPING` and add local-only note**

Find the line `## PLATFORMS` and replace with:

```
## SCRAPING (local mode only)

**Skip this section entirely in remote WebFetch-only mode** — rely on alert ingestion instead.
```

Leave the rest of that section (the 4 platform URLs + subsection intros) unchanged.

- [ ] **Step 4: Update `## TRACKER` section — add Source property**

In the TRACKER section, find the line that begins with `Columns (see tracker/flats-schema.md):` and replace the inline property list with:

```
Columns (see tracker/flats-schema.md): Title (title), URL, Platform, Found On, Area, Price, Beds, Size, Size Source, Furnished, Available From, Floor, Bathtub, New/Renovated, Calm, Light, Wooden Floor, Score, Tier, Status, Reason, Needs-verify, Notes, Source.
```

Then, after the "Dedup: before inserting…" paragraph (which ends `Skip if any match.`), add a new paragraph:

```
Set `Source` on every new row:
- `email-alert` when the row came from the Alert Ingestion path
- `scraped-local` when from scraping in local mode (Claude-in-Chrome)
- `scraped-remote` when from scraping in remote WebFetch mode (this should be rare — remote mode skips scraping in the new architecture)

If dedup catches a URL, do NOT overwrite `Source` on the existing row.
```

- [ ] **Step 5: Update `## SUCCESS CRITERIA` section**

Replace the current success criteria bullets with:

```
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

- [ ] **Step 6: Verify the file**

Read `skill-flats.md` and confirm:
- `## ALERT INGESTION` section exists between SCORING and SCRAPING
- `## PLATFORMS` no longer exists (renamed to `## SCRAPING`)
- TRACKER section mentions `Source` in the column list and has the new "Set `Source` on every new row" paragraph
- SUCCESS CRITERIA includes alert-labeling and source-setting bullets
- RUNTIME MODE mentions "Always run alert ingestion first"

- [ ] **Step 7: Commit**

```bash
cd /home/ara/dev/london-flat-hunt/london-property-hunt-public
git add skill-flats.md
git commit -m "skill(flats): add alert ingestion path, rename PLATFORMS→SCRAPING

Adds a new ALERT INGESTION section that reads portal saved-search emails
from Gmail, extracts listing URLs, fetches listing pages, and writes to
Notion. Runs first in every mode. Scraping is renamed for clarity and
gated to local-mode only. TRACKER and SUCCESS CRITERIA updated for the
new Source property."
```

---

### Task 2: Update `tracker/flats-schema.md` — document the new `Source` property

**Files:**
- Modify: `tracker/flats-schema.md`

- [ ] **Step 1: Add `Source` row to the Flats database property table**

In `tracker/flats-schema.md`, find the Flats database property table (starts with `| Property | Type | Notes |`). Add a new row at the bottom, right before the closing of the table (i.e. after the `Notes` row and before the next section header):

```
| Source | select | email-alert / scraped-local / scraped-remote — which ingestion path added this row |
```

- [ ] **Step 2: Update the "23 properties total" sentence**

Find the sentence below the table that currently reads `23 properties total. Tier select colours render HIGH/MEDIUM/LOW rows green/yellow/red automatically.` Replace with:

```
24 properties total (23 original + Source). Tier select colours render HIGH/MEDIUM/LOW rows green/yellow/red automatically. Source has no semantic colour meaning but is colour-coded for scanability.
```

- [ ] **Step 3: Add Source to the setup snippet's SQL DDL**

In the setup snippet's `CREATE TABLE` statement for the Flats database, add the `Source` column after `"Notes" RICH_TEXT`:

```
    "Notes" RICH_TEXT,
    "Source" SELECT('email-alert':green, 'scraped-local':blue, 'scraped-remote':purple)
```

- [ ] **Step 4: Verify**

Read `tracker/flats-schema.md` and confirm:
- New `| Source | select | ... |` row exists in the property table
- "24 properties total" sentence is updated
- `CREATE TABLE` DDL in the setup snippet includes `"Source" SELECT(...)`

- [ ] **Step 5: Commit**

```bash
git add tracker/flats-schema.md
git commit -m "tracker: document Source property for layered ingestion"
```

---

### Task 3: Create `tracker/alerts-setup.md` — per-portal saved-search instructions

**Files:**
- Create: `tracker/alerts-setup.md`

- [ ] **Step 1: Write the file**

Create `tracker/alerts-setup.md` with:

```markdown
# Saved-Search Email Alerts — Per-Portal Setup (Flats Variant)

The flats variant's alert ingestion path relies on each portal sending instant email alerts to your hunt Gmail account when a new listing matches your saved search. This is a one-time setup, ~20 min total across the four portals.

All four saved searches should match the criteria from your `config.md`:
- 1–2 bedrooms
- Max rent £3,500 pcm (hard cap)
- Primary + secondary area list (set as broadly as the portal's filter allows)
- Unfurnished or part-furnished preferred (where filter supports it)

## Rightmove

1. Go to [rightmove.co.uk](https://www.rightmove.co.uk/) and sign in (create account on rvonaar@gmail.com if needed).
2. Search for flats in your first primary area (e.g. Islington). Apply filters: 1+ bedrooms, max 2 bedrooms, max £3,500 pcm, flat type, let agreed = exclude.
3. Click **Create alert** (or bell icon).
4. Choose **Instant alerts** (not daily digest).
5. Repeat for every primary area that doesn't appear in your first search's radius. Rightmove doesn't let you multi-select areas in a single saved search — you'll likely end up with 4–8 saved searches to cover the whole area list.
6. Verify: trigger a test alert by clicking "Send me a test email" or wait for the next real listing. An email should arrive from `property-alerts@rightmove.co.uk` (or similar `@rightmove.co.uk`) within minutes.

## Zoopla

1. Go to [zoopla.co.uk](https://www.zoopla.co.uk/) and sign in.
2. Search: enter "Islington" (or first area), filter to 1–2 bed, max £3,500 pcm, to-rent, flats only.
3. Click **Save search** → **Alert me instantly** (or "Alert me daily" if instant isn't offered; instant is preferred).
4. Repeat for each primary area. Zoopla also limits single-search multi-area selection.
5. Verify emails arrive from `@zoopla.co.uk` sender.

## SpareRoom

1. Go to [spareroom.co.uk](https://www.spareroom.co.uk/) and sign in.
2. In **Flats-to-rent** mode (not rooms), search with your criteria.
3. Click **Save this search**. SpareRoom saved searches come with email alerts on by default (daily or instant — pick instant).
4. Verify emails arrive from `@spareroom.co.uk`.

## OpenRent

1. Go to [openrent.co.uk](https://www.openrent.co.uk/) and sign in.
2. Use the search to set your criteria (1–2 bed, max £3,500, areas, furnished preference).
3. Click **Save search** or the bell icon.
4. Verify emails arrive from `@openrent.com` or `@openrent.co.uk` sender.

## Post-setup check

After setting up all four:
- Wait 24 hours.
- Check the Gmail inbox at rvonaar@gmail.com — you should see at least one email per portal over that window (assuming the market has new listings, which is typical in London).
- Confirm each email's sender matches one of the four domains used in the skill's Gmail query: `@rightmove.co.uk`, `@zoopla.co.uk`, `@spareroom.co.uk`, `@openrent.co.uk` (or `@openrent.com`).
- If a portal uses a different sender domain (e.g. `@rmemail.co.uk` or `@news.zoopla.co.uk`), update the Gmail query in `skill-flats.md`'s ALERT INGESTION section to include it.

## Troubleshooting

- **No alerts after 24h**: check the portal's email settings on your account page; some default to daily digest even when you picked instant. Turn off daily digest entirely to avoid duplicate emails.
- **Alerts in spam**: mark as not-spam and add a filter in Gmail to auto-mark as important and skip spam.
- **Alert volume too high**: narrow saved searches (tighter bed range, area list, price). The skill's hard filters will drop out-of-scope listings anyway, but reducing volume lowers Gmail MCP noise.
- **A portal doesn't offer saved-search alerts on your plan**: skip that portal for alerts; it still gets picked up via local scraping.
```

- [ ] **Step 2: Verify**

Confirm `tracker/alerts-setup.md` exists and contains headings for Rightmove, Zoopla, SpareRoom, OpenRent, Post-setup check, and Troubleshooting.

- [ ] **Step 3: Commit**

```bash
git add tracker/alerts-setup.md
git commit -m "tracker: add per-portal saved-search alert setup guide"
```

---

### Task 4: Update `README.md` — mention alert ingestion

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Update the Flats variant row in the Variants table**

Find the row in the Variants table that says:

```
| **Flats** | [`skill-flats.md`](skill-flats.md) | [Notion](tracker/flats-schema.md) (Notion MCP) | Renting a whole 1–2 bed flat; scoring on top-floor / new-build / calm / bathtub / light / wooden floor |
```

Replace the "Who should use it" cell to also mention ingestion:

```
| **Flats** | [`skill-flats.md`](skill-flats.md) | [Notion](tracker/flats-schema.md) (Notion MCP) | Renting a whole 1–2 bed flat; alert-emails + local scraping; scoring on top-floor / new-build / calm / bathtub / light / wooden floor |
```

- [ ] **Step 2: Update the Flats-variant setup step 2 (Create your tracker)**

Find the `**Flats variant** (Notion):` block in `### 2. Create your tracker`. After the sentence "Paste the three resulting IDs into `FLAT_TRACKER_*` in your `config.md`.", add a new paragraph:

```

Set up portal email alerts (primary ingestion path): see [tracker/alerts-setup.md](tracker/alerts-setup.md).
```

- [ ] **Step 3: Update the repo tree in the Repository structure section**

Find the repo tree and add `alerts-setup.md` to the `tracker/` subtree:

```
├── tracker/
│   ├── README.md          ← rooms tracker (xlsx) column schema
│   ├── flats-schema.md    ← flats tracker (Google Sheet) column schema
│   └── alerts-setup.md    ← flats per-portal saved-search setup
```

Wait — the current tree line for flats-schema says "flats tracker (Google Sheet)" but that changed to Notion. Correct both:

Replace the `tracker/` block in the tree with:

```
├── tracker/
│   ├── README.md          ← rooms tracker (xlsx) column schema
│   ├── flats-schema.md    ← flats tracker (Notion) column schema
│   └── alerts-setup.md    ← flats per-portal saved-search email alerts
```

- [ ] **Step 4: Verify**

Read `README.md` and confirm:
- Variants table row for Flats mentions "alert-emails + local scraping"
- Setup step 2's flats block links to `tracker/alerts-setup.md`
- Repo tree shows `alerts-setup.md` and the flats-schema description now says "(Notion)" not "(Google Sheet)"

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs(readme): mention alert ingestion and alerts-setup.md"
```

---

### Task 5: Final Phase 1 verification

- [ ] **Step 1: List commits from this phase**

Run: `git log --since="1 hour ago" --oneline` and confirm four commits: skill-flats.md, flats-schema.md, alerts-setup.md (new), README.md.

- [ ] **Step 2: Confirm no untracked files or unstaged diffs**

Run: `git status` — should be clean.

- [ ] **Step 3: Cross-check skill-flats.md against config.example.md**

Run: `grep -E '\[FLAT_[A-Z_]+\]' skill-flats.md | sort -u` and verify every listed placeholder exists in `config.example.md`. Expected: the new ALERT INGESTION section reuses `[FLAT_TRACKER_FLATS_DATA_SOURCE_ID]`, so no new placeholders should appear.

- [ ] **Step 4: Cross-check skill-flats.md for Gmail MCP tool names**

Run: `grep -oE 'mcp__claude_ai_Gmail__[a-z_-]+' skill-flats.md | sort -u`. Expected tools referenced: `mcp__claude_ai_Gmail__search_threads`, `mcp__claude_ai_Gmail__get_thread`, `mcp__claude_ai_Gmail__label_thread`, `mcp__claude_ai_Gmail__create_label`. All four are available in the claude.ai Gmail connector.

---

## Phase 2 — Live system: Notion schema and remote trigger

### Task 6: Add `Source` property to the Notion Flats database

This task uses the Notion MCP from within the current Claude Code session. Run it here, not on the remote trigger.

- [ ] **Step 1: Call `mcp__claude_ai_Notion__notion-update-data-source`**

Use `ADD COLUMN` (the Source property doesn't exist yet):

```
data_source_id: c417b1a4-2ebd-4686-85ec-ca1219a8b0d4
statements: ADD COLUMN "Source" SELECT('email-alert':green, 'scraped-local':blue, 'scraped-remote':purple) COMMENT 'Which ingestion path added this row — email-alert is the primary path, scraped-local is laptop Chrome, scraped-remote is Anthropic-hosted fallback (should be rare)'
```

- [ ] **Step 2: Verify via fetch**

Call `mcp__claude_ai_Notion__notion-fetch` with:

```
id: collection://c417b1a4-2ebd-4686-85ec-ca1219a8b0d4
```

Confirm the returned schema includes a `Source` property of type `select` with the three options `email-alert`, `scraped-local`, `scraped-remote`.

- [ ] **Step 3: Update the default view to show Source**

Call `mcp__claude_ai_Notion__notion-update-view` with:

```
view_id: 2cf5f712-83f0-42bd-90b6-cab3f1e20f25
configure: SHOW "Title", "Tier", "Human Tier", "Human Rationale", "Score", "Status", "Source", "Area", "Price", "Beds", "Size", "Floor", "Platform", "Found On", "URL", "Reason", "Needs-verify", "Furnished", "Bathtub", "Wooden Floor", "Light", "New/Renovated", "Calm", "Size Source", "Available From", "Notes"
```

Source is placed between Status and Area for scanability — early enough to be visible, not so early it crowds the tier/score decision columns.

---

### Task 7: Update the remote fallback trigger prompt

- [ ] **Step 1: Fetch the current trigger to get the existing CONFIG block**

First call `RemoteTrigger action:"get"` with `trigger_id: trig_01St8iTA4T9nkW5iRFbRMp7a` and copy the CONFIG section (the `- YOUR_NAME=...` through `- FLAT_OUTREACH_AVAILABILITY=...` block). This block is unchanged in this update.

- [ ] **Step 2: Call `RemoteTrigger action:"update"` with the new prompt**

Use this body (CONFIG block copied verbatim from Step 1; all other content below is new):

```json
{
  "job_config": {
    "ccr": {
      "environment_id": "env_0133B9zXFFuyAwFrHouxZ75Z",
      "session_context": {
        "model": "claude-sonnet-4-6",
        "sources": [{"git_repository": {"url": "https://github.com/xEcEz/london-property-hunt-public"}}],
        "allowed_tools": ["Read", "Glob", "Grep", "Bash", "WebFetch"]
      },
      "events": [{
        "data": {
          "uuid": "129bff36-fa80-45ec-be52-1928a92812b5",
          "session_id": "",
          "type": "user",
          "parent_tool_use_id": null,
          "message": {
            "role": "user",
            "content": "<FULL PROMPT BELOW>"
          }
        }
      }]
    }
  }
}
```

The full prompt content to paste into the `content` field (build in your language; paste raw):

```
You are the remote fallback runner for Raphael's London flat hunt (1–2 bed whole-unit rentals near King's Cross).

The git repository is checked out at the working directory. Read `skill-flats.md` from the repo root — that is your full skill prompt. All `[PLACEHOLDER]` values are filled from the config below.

## CONFIG

[same config block as current prompt — YOUR_NAME, YOUR_AGE, ..., FLAT_OUTREACH_AVAILABILITY]

## ALERT INGESTION (PRIMARY PATH — always run first)

1. Query Gmail via mcp__claude_ai_Gmail__search_threads with: (from:(@rightmove.co.uk) OR from:(@zoopla.co.uk) OR from:(@spareroom.co.uk) OR from:(@openrent.co.uk)) AND newer_than:2d AND -label:hunt-processed
2. For each thread returned, fetch full body via mcp__claude_ai_Gmail__get_thread, extract listing URLs matching the portal patterns defined in skill-flats.md's ALERT INGESTION section.
3. For each extracted URL: query the Notion Flats data source (id c417b1a4-2ebd-4686-85ec-ca1219a8b0d4) for URL == candidate. If match, skip.
4. Otherwise WebFetch the individual listing page and extract signals per skill-flats.md's SCORING section. If WebFetch fails (403, timeout, any error), create a minimal row using alert-email data: Title, URL, Price, Beds from the alert; Size Source=unknown, Needs-verify=\"listing page unreachable\", Source=email-alert, Score/Tier left blank.
5. Apply score and tier per skill-flats.md when page fetch succeeds. Every new row: Source=email-alert, Status=NEW, Found On=today (Europe/Zurich YYYY-MM-DD).
6. After processing each thread, apply Gmail label `hunt-processed` via mcp__claude_ai_Gmail__label_thread. If the label doesn't exist on first run, create it via mcp__claude_ai_Gmail__create_label first.

## NO SCRAPING IN REMOTE MODE

Do NOT scrape Rightmove/Zoopla/SpareRoom/OpenRent search pages via WebFetch. Those paths are aggressively gated from Anthropic's datacenter IPs. The ALERT INGESTION above is your sole input source.

## META + EMAIL (after alert ingestion completes)

1. Query the Notion Meta database (data source id e914a179-d983-4730-b863-97d188843f90) for the row where Key == \"last_local_run\". Read its Value field.
2. Compute today's date in Europe/Zurich as YYYY-MM-DD.
3. If Meta.last_local_run.Value equals today's date → do not create an email draft. Still update Meta.last_remote_run below (so the health check knows we ran). Exit without emailing.
4. Otherwise compose and create a Gmail draft via gmail_create_draft per skill-flats.md's EMAIL section. Subject: 🏠 London Flat Hunt — {DATE} — {N} new. To: fruiteam@gmail.com. No Cc.
5. Find the Meta row where Key == \"last_remote_run\" and call notion-update-page to set its Value to today's date.

## NOTES

- The Flats database has \"Human Tier\" (select HIGH/MEDIUM/LOW) and \"Human Rationale\" (rich text) columns — reviewer only. Do NOT set on new listings.
- Outreach in remote mode: inline the outreach copy in the email cards (no .txt files).
- Apps Script in rvonaar@gmail.com auto-sends matching drafts — do NOT try to send via a second mechanism.
- When calling notion-create-pages for a new listing, pass parent as {type: \"data_source_id\", data_source_id: \"c417b1a4-2ebd-4686-85ec-ca1219a8b0d4\"} and set applicable properties per the Flats schema including the new Source property.
```

(Substitute the CONFIG block with the exact same list from the current prompt — all `FLAT_*` and `YOUR_*` values unchanged.)

- [ ] **Step 3: Verify via `RemoteTrigger action:"get"`**

Fetch trigger `trig_01St8iTA4T9nkW5iRFbRMp7a` and confirm the prompt content now contains:
- `## ALERT INGESTION (PRIMARY PATH`
- `## NO SCRAPING IN REMOTE MODE`
- `Source=email-alert`

---

## Phase 3 — User-executed (portal saved-search setup)

### Task 8: Rightmove saved-search alerts

- [ ] **Step 1:** Sign in at rightmove.co.uk as rvonaar@gmail.com (or equivalent) per `tracker/alerts-setup.md`.
- [ ] **Step 2:** Create a saved search per each primary area (Islington, Angel, Camden Town, Kentish Town, De Beauvoir Town, Highbury, Canonbury, Clerkenwell). Filters: 1–2 bed, max £3,500, flats, exclude let agreed.
- [ ] **Step 3:** Turn on instant email alerts for each saved search.
- [ ] **Step 4:** Verify: a test email arrives from `@rightmove.co.uk` within 24 hours.

### Task 9: Zoopla saved-search alerts

Same steps as Task 8, applied to zoopla.co.uk. Verify emails from `@zoopla.co.uk`.

### Task 10: SpareRoom saved-search alerts

Same, applied to spareroom.co.uk (Flats-to-rent mode). Verify emails from `@spareroom.co.uk`.

### Task 11: OpenRent saved-search alerts

Same, applied to openrent.co.uk. Verify emails from `@openrent.com` or `@openrent.co.uk`.

---

## Phase 4 — Smoke test and observation

### Task 12: Fire remote trigger to test alert ingestion end-to-end

- [ ] **Step 1: Wait until at least one alert email has landed in rvonaar@gmail.com**

Check Gmail via `mcp__claude_ai_Gmail__search_threads` with query `from:(@rightmove.co.uk) OR from:(@zoopla.co.uk) OR from:(@spareroom.co.uk) OR from:(@openrent.co.uk) newer_than:1d`. Expect ≥1 result.

- [ ] **Step 2: Fire the remote fallback trigger**

Call `RemoteTrigger action:"run"` with `trigger_id: trig_01St8iTA4T9nkW5iRFbRMp7a`.

- [ ] **Step 3: Wait ~5 minutes for the run to complete**

Then check:
1. Notion Flats db: new rows with `Source=email-alert` and `Found On=<today>`.
2. Notion Meta db: `last_remote_run` is `<today>`.
3. Gmail: a new draft subject `🏠 London Flat Hunt — <today> — N new` with To: fruiteam@gmail.com.
4. Gmail: the processed alert threads now have the `hunt-processed` label.

- [ ] **Step 4: Within 1 hour, Apps Script auto-sends the draft**

Verify the email arrives in fruiteam@gmail.com inbox.

- [ ] **Step 5: Break-glass test — label reset**

Manually remove the `hunt-processed` label from one thread in Gmail. Fire the remote trigger again. Verify:
- That thread is picked up again in the next run's Gmail query.
- The URLs extracted from it are dedup-skipped (they're already in Notion).
- The thread is re-labeled `hunt-processed`.
- No duplicate Notion rows.

---

### Task 13: 7-day observation window

- [ ] **Step 1: Each day for 7 days, check:**

  - Count of new rows per Source (email-alert / scraped-local / scraped-remote). Expect `email-alert` to grow fastest.
  - Any days with zero new rows (shouldn't happen in a normal market unless filters are too tight).
  - Any listings that appear via scraping but not via alerts (investigate: saved-search filter missing a match?).
  - Any `Needs-verify: listing page unreachable` rows (count them; if >20% of email-alert rows, consider escalating to Browserbase).

- [ ] **Step 2: After 7 days, decide Phase 2 rollout:**

  - If email-alert contributes ≥60% of new rows → primary path is healthy.
  - If scraping contributes <20% unique inventory → consider reducing local cron to weekly (YAGNI).
  - If email-alert contributes <60% and scraping is filling the gap → investigate alert setup (filter too narrow? sender domain missed?).

---

## Done

When Phase 4 observation window is green:

- Flats hunt has two redundant ingestion paths
- Daily email arrives regardless of laptop uptime or datacenter-IP gating
- Notion tracker stays full and deduped
- 15%-missing-days problem is resolved

### Limitations to revisit

- **Apps Script 15-min-to-1-hour delivery latency** is fine for a daily hunt but noticeable. Deferred: Gmail Push + webhook if sub-minute latency becomes important.
- **Portal alert template drift** could break URL extraction over time. Mitigation: monitor `Size Source=unknown` and `Needs-verify` rates monthly; patch skill-flats.md regex/patterns if drift emerges.
- **SpareRoom / OpenRent alert coverage** may be weaker than Rightmove/Zoopla for ≥60 m² 1–2 bed flats. If a portal becomes consistently low-yield for alerts, keep scraping it locally.
