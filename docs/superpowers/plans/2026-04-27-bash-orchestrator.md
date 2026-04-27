# Bash-Orchestrated Subagent Dispatch — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the single inline `claude` invocation in `~/hunt/run-hunt.sh` with a Bash orchestrator that fires 11 narrow `claude -p` invocations (INIT, ALERTS, 8 SCRAPE-portal-tier, DIGEST), so per-portal-tier scope is structurally enforced by Bash rather than relying on the model to honour skill-text instructions.

**Architecture:** Bash drives the pipeline; each phase reads from / writes to a per-run state directory (`~/hunt/run-state/<run-id>/`); each phase has a prompt file template that Bash fills in and pipes to `claude -p`; Bash performs JSON-validation and per-area completeness checks on every phase output before continuing.

**Tech Stack:** Bash 5+, GNU coreutils (`timeout`, `date`, `mktemp`, `find`), `xvfb-run`, `jq` (already installed for the spec self-tests), `envsubst` (gettext), `claude` CLI 2.1+, claude.ai MCP connectors (Gmail, Notion), Playwright MCP.

**Reference spec:** `docs/superpowers/specs/2026-04-27-bash-orchestrator-design.md` (commit `cc224b4`, post-codex revisions).

---

## File Structure

The orchestrator lives in the user's home directory (`/home/ara/hunt/`, gitignored — local config), not in the public repo. Only `skill-flats.md` lives in the repo and gets edited as part of this plan.

```
/home/ara/hunt/                              # local, NOT in public repo
  run-hunt.sh                                # MODIFY (rewritten orchestrator)
  areas.json                                 # CREATE (declarative portal-area URLs)
  prompts/
    init.txt                                 # CREATE (INIT prompt template)
    alerts.txt                               # CREATE (ALERTS prompt template)
    scrape.txt                               # CREATE (SCRAPE prompt template, used 8×)
    digest.txt                               # CREATE (DIGEST prompt template)
  run-state/                                 # CREATE (per-run state dirs go here)

/home/ara/dev/london-flat-hunt/london-property-hunt-public/
  skill-flats.md                             # MODIFY (strip the orchestration block)
```

**Why these splits:**
- `run-hunt.sh` is the orchestration entrypoint — Bash logic only, no prompt content.
- `prompts/*.txt` keeps prompt content separate from script logic so prompts can be tweaked without editing the script.
- `areas.json` is the single source of truth for portal-area URLs; today they're embedded in `skill-flats.md` and have to be kept in sync. Lifting them out lets the orchestrator generate scrape prompts deterministically.
- `run-state/<run-id>/` is the per-run scratch space. Each phase writes its `<phase>.json`; DIGEST reads them all.

**Testing approach:** Per the spec § 7, no automated test suite — manual verification on a real run. Each task below has explicit verify steps that run a fresh shell command and check observable state (file presence, JSON validity, exit codes, log contents). No `pytest` / `bats` setup.

---

## Task 1: Create areas.json with all portal-area URLs

**Files:**
- Create: `/home/ara/hunt/areas.json`

**Why:** Single source of truth for the 4 portals × 16 areas. Bash reads this to assemble each scrape prompt's URL list. Today these URLs are in `skill-flats.md` § Portal search URLs (primary areas) and § Portal search URLs (secondary areas).

- [ ] **Step 1: Write areas.json**

```bash
cat > /home/ara/hunt/areas.json <<'JSON'
{
  "rm-primary": [
    {"area": "Islington",         "url": "https://www.rightmove.co.uk/property-to-rent/find.html?locationIdentifier=REGION%5E93965&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"},
    {"area": "Highbury",          "url": "https://www.rightmove.co.uk/property-to-rent/find.html?locationIdentifier=REGION%5E70438&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"},
    {"area": "Camden Town",       "url": "https://www.rightmove.co.uk/property-to-rent/find.html?locationIdentifier=REGION%5E85262&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"},
    {"area": "Kentish Town",      "url": "https://www.rightmove.co.uk/property-to-rent/find.html?locationIdentifier=REGION%5E85230&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"},
    {"area": "De Beauvoir Town",  "url": "https://www.rightmove.co.uk/property-to-rent/find.html?locationIdentifier=REGION%5E70393&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"},
    {"area": "Clerkenwell",       "url": "https://www.rightmove.co.uk/property-to-rent/find.html?locationIdentifier=REGION%5E87500&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"}
  ],
  "rm-secondary": [
    {"area": "Tufnell Park",   "url": "https://www.rightmove.co.uk/property-to-rent/find.html?searchLocation=Tufnell+Park&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"},
    {"area": "Holloway",       "url": "https://www.rightmove.co.uk/property-to-rent/find.html?searchLocation=Holloway&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"},
    {"area": "Bloomsbury",     "url": "https://www.rightmove.co.uk/property-to-rent/find.html?searchLocation=Bloomsbury&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"},
    {"area": "Russell Square", "url": "https://www.rightmove.co.uk/property-to-rent/find.html?searchLocation=Russell+Square&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"},
    {"area": "Barbican",       "url": "https://www.rightmove.co.uk/property-to-rent/find.html?searchLocation=Barbican&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"},
    {"area": "Finsbury Park",  "url": "https://www.rightmove.co.uk/property-to-rent/find.html?searchLocation=Finsbury+Park&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"},
    {"area": "London Bridge",  "url": "https://www.rightmove.co.uk/property-to-rent/find.html?searchLocation=London+Bridge&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"},
    {"area": "Bermondsey",     "url": "https://www.rightmove.co.uk/property-to-rent/find.html?searchLocation=Bermondsey&minBedrooms=1&maxBedrooms=2&maxPrice=3500&propertyTypes=flat&includeLetAgreed=false&sortType=6"}
  ],
  "zoopla-primary": [
    {"area": "Islington",        "url": "https://www.zoopla.co.uk/to-rent/flats/london/islington/?q=Islington&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "Angel",            "url": "https://www.zoopla.co.uk/to-rent/flats/london/angel/?q=Angel&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "Camden Town",      "url": "https://www.zoopla.co.uk/to-rent/flats/london/camden-town/?q=Camden+Town&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "Kentish Town",     "url": "https://www.zoopla.co.uk/to-rent/flats/london/kentish-town/?q=Kentish+Town&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "De Beauvoir Town", "url": "https://www.zoopla.co.uk/to-rent/flats/london/de-beauvoir-town/?q=De+Beauvoir+Town&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "Highbury",         "url": "https://www.zoopla.co.uk/to-rent/flats/london/highbury/?q=Highbury&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "Canonbury",        "url": "https://www.zoopla.co.uk/to-rent/flats/london/canonbury/?q=Canonbury&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "Clerkenwell",      "url": "https://www.zoopla.co.uk/to-rent/flats/london/clerkenwell/?q=Clerkenwell&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"}
  ],
  "zoopla-secondary": [
    {"area": "Tufnell Park",   "url": "https://www.zoopla.co.uk/to-rent/flats/london/tufnell-park/?q=Tufnell+Park&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "Holloway",       "url": "https://www.zoopla.co.uk/to-rent/flats/london/holloway/?q=Holloway&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "Bloomsbury",     "url": "https://www.zoopla.co.uk/to-rent/flats/london/bloomsbury/?q=Bloomsbury&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "Russell Square", "url": "https://www.zoopla.co.uk/to-rent/flats/london/russell-square/?q=Russell+Square&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "Barbican",       "url": "https://www.zoopla.co.uk/to-rent/flats/london/barbican/?q=Barbican&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "Finsbury Park",  "url": "https://www.zoopla.co.uk/to-rent/flats/london/finsbury-park/?q=Finsbury+Park&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "London Bridge",  "url": "https://www.zoopla.co.uk/to-rent/flats/london/london-bridge/?q=London+Bridge&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"},
    {"area": "Bermondsey",     "url": "https://www.zoopla.co.uk/to-rent/flats/london/bermondsey/?q=Bermondsey&beds_min=1&beds_max=2&price_frequency=per_month&price_max=3500&results_sort=newest_listings"}
  ],
  "sr-primary": [
    {"area": "Islington",        "url": "https://www.spareroom.co.uk/flats-to-rent/london/islington?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "Angel",            "url": "https://www.spareroom.co.uk/flats-to-rent/london/angel?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "Camden Town",      "url": "https://www.spareroom.co.uk/flats-to-rent/london/camden_town?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "Kentish Town",     "url": "https://www.spareroom.co.uk/flats-to-rent/london/kentish_town?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "De Beauvoir Town", "url": "https://www.spareroom.co.uk/flats-to-rent/london/de_beauvoir_town?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "Highbury",         "url": "https://www.spareroom.co.uk/flats-to-rent/london/highbury?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "Canonbury",        "url": "https://www.spareroom.co.uk/flats-to-rent/london/canonbury?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "Clerkenwell",      "url": "https://www.spareroom.co.uk/flats-to-rent/london/clerkenwell?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"}
  ],
  "sr-secondary": [
    {"area": "Tufnell Park",   "url": "https://www.spareroom.co.uk/flats-to-rent/london/tufnell_park?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "Holloway",       "url": "https://www.spareroom.co.uk/flats-to-rent/london/holloway?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "Bloomsbury",     "url": "https://www.spareroom.co.uk/flats-to-rent/london/bloomsbury?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "Russell Square", "url": "https://www.spareroom.co.uk/flats-to-rent/london/russell_square?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "Barbican",       "url": "https://www.spareroom.co.uk/flats-to-rent/london/barbican?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "Finsbury Park",  "url": "https://www.spareroom.co.uk/flats-to-rent/london/finsbury_park?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "London Bridge",  "url": "https://www.spareroom.co.uk/flats-to-rent/london/london_bridge?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"},
    {"area": "Bermondsey",     "url": "https://www.spareroom.co.uk/flats-to-rent/london/bermondsey?min_beds=1&max_beds=2&max_rent=3500&sort=posted_date&mode=list"}
  ],
  "or-primary": [
    {"area": "Combined-primary",   "url": "https://www.openrent.co.uk/properties-to-rent/london?term=Islington,Angel,Camden%20Town,Kentish%20Town,De%20Beauvoir%20Town,Highbury,Canonbury,Clerkenwell&prices_max=3500&bedrooms_min=1&bedrooms_max=2&isLive=true"}
  ],
  "or-secondary": [
    {"area": "Combined-secondary", "url": "https://www.openrent.co.uk/properties-to-rent/london?term=Tufnell%20Park,Holloway,Bloomsbury,Russell%20Square,Barbican,Finsbury%20Park,London%20Bridge,Bermondsey&prices_max=3500&bedrooms_min=1&bedrooms_max=2&isLive=true"}
  ]
}
JSON
```

- [ ] **Step 2: Verify it parses as valid JSON and contains the expected key counts**

Run:
```bash
jq -r 'to_entries | map("\(.key): \(.value | length)") | .[]' /home/ara/hunt/areas.json
```

Expected:
```
rm-primary: 6
rm-secondary: 8
zoopla-primary: 8
zoopla-secondary: 8
sr-primary: 8
sr-secondary: 8
or-primary: 1
or-secondary: 1
```

If `jq` errors or counts don't match, fix the JSON. Total = 48 rows, which corresponds to "4 portals × 16 areas" minus the 2 RM-primary URLs that cover 3 areas each (Islington covers Islington+Angel+Canonbury, that's why rm-primary has 6 not 8) plus the OpenRent combined URLs (1 each tier).

- [ ] **Step 3: No commit (file lives in `~/hunt/`, gitignored)**

The hunt directory is local config, not version-controlled. Skip commit for this task.

---

## Task 2: Write the four prompt templates

**Files:**
- Create: `/home/ara/hunt/prompts/init.txt`
- Create: `/home/ara/hunt/prompts/alerts.txt`
- Create: `/home/ara/hunt/prompts/scrape.txt`
- Create: `/home/ara/hunt/prompts/digest.txt`

**Why:** Each `claude -p` invocation is fed a prompt assembled from one of these templates plus runtime values. Using `envsubst` for `$VAR`-style substitution.

- [ ] **Step 1: Create the prompts directory**

```bash
mkdir -p /home/ara/hunt/prompts
```

- [ ] **Step 2: Write init.txt**

```bash
cat > /home/ara/hunt/prompts/init.txt <<'PROMPT'
You are the INIT subagent for the London flat hunt. Your scope:
enumerate the existing Notion Flats data source and write the
canonical known-URL set to disk.

Skill reference:
/home/ara/dev/london-flat-hunt/london-property-hunt-public/skill-flats.md
— specifically § INIT — LOAD KNOWN URL SET and § TRACKER for the
URL canonicalisation rules.

Configuration:
- Flats data source ID: $FLATS_DATA_SOURCE_ID
- Run-state directory: $RUN_STATE_DIR
- Output path for known URLs: $RUN_STATE_DIR/known_urls.txt
- Output path for phase result: $RUN_STATE_DIR/init.json

Required actions:
1. Use mcp__claude_ai_Notion__notion-search with broad queries
   (e.g. "bed", "flat", area names, tier values) on the data source
   $FLATS_DATA_SOURCE_ID, paginated via nextPageToken, to enumerate
   ALL page UUIDs. Cap at 30 pagination iterations per query.
2. For each unique page UUID, call mcp__claude_ai_Notion__notion-fetch
   in parallel batches of ~25, read the userDefined:URL property,
   apply the canonicalisation rules from § TRACKER (lowercase host,
   strip trailing slash, drop tracking query params, canonicalise
   /properties/<id> for Rightmove etc).
3. Write each canonical URL on its own line to
   $RUN_STATE_DIR/known_urls.txt (use the Write tool).
4. Write a JSON result file to $RUN_STATE_DIR/init.json with this
   exact shape (use the Write tool):

   {
     "phase": "init",
     "started_at": "<ISO-8601 UTC start time>",
     "finished_at": "<ISO-8601 UTC end time>",
     "exit_code": 0,
     "url_count": <number of URLs in known_urls.txt>,
     "pages_enumerated": <total unique page UUIDs found>,
     "pages_fetched_ok": <count of successful notion-fetch calls>,
     "pages_fetched_failed": <count of fetches that errored>,
     "errors": ["<one-line error desc>", ...]
   }

Failure handling:
- If notion-search errors persist after 3 retries with exponential
  backoff (1s, 3s, 9s), DO NOT write known_urls.txt. Write init.json
  with exit_code=1 and a clear error message. The orchestrator will
  detect this and set the init.failed flag, halting downstream
  writes for this run.
- If individual notion-fetch calls fail, retry once then skip that
  page; tally in pages_fetched_failed. Acceptable up to 5% failure
  rate; above that, set exit_code=1.

DO NOT:
- Write to Notion (no notion-create-pages, no notion-update-page).
- Send emails or create drafts.
- Run scraping.
- Read Gmail.
PROMPT
```

- [ ] **Step 3: Write alerts.txt**

```bash
cat > /home/ara/hunt/prompts/alerts.txt <<'PROMPT'
You are the ALERTS subagent for the London flat hunt. Your scope:
ingest portal-alert emails from Gmail and insert qualified listings
to Notion.

Skill reference:
/home/ara/dev/london-flat-hunt/london-property-hunt-public/skill-flats.md
— specifically § ALERT INGESTION (Gmail query, per-thread handling,
URL extraction by portal, label-after-handling), § HARD FILTERS,
§ SIZE RESOLUTION, § SCORING, § TRACKER (URL canonicalisation).

Configuration:
- Flats data source ID: $FLATS_DATA_SOURCE_ID
- Run-state directory: $RUN_STATE_DIR
- Known URL set file: $RUN_STATE_DIR/known_urls.txt

INIT-failure check (DO THIS FIRST):
If $RUN_STATE_DIR/init.failed exists, write
$RUN_STATE_DIR/alerts.json with this content and exit:

   {
     "phase": "alerts",
     "started_at": "<ISO-8601 UTC>",
     "finished_at": "<ISO-8601 UTC>",
     "exit_code": 0,
     "outcome": "SKIPPED-INIT-FAILED",
     "threads_processed": 0,
     "inserted_pages": [],
     "errors": []
   }

Do NOT call notion-create-pages under any circumstances when this
flag is present.

Required actions (when init has not failed):
1. Read $RUN_STATE_DIR/known_urls.txt into an in-memory set; treat
   each line as a URL.
2. Run § ALERT INGESTION end-to-end: Gmail search, per-thread URL
   extraction, dedup against the in-memory set, hard-filter,
   enrich-if-needed, score, insert via mcp__claude_ai_Notion__notion-create-pages
   with parent.data_source_id = $FLATS_DATA_SOURCE_ID and Source = "email-alert".
3. Immediately after each successful insert, append the canonical
   URL to $RUN_STATE_DIR/known_urls.txt as a newline-terminated line
   (use Write tool with single line) so subsequent phases of this
   run see it.
4. Apply Gmail label "hunt-processed" to fully-handled threads if
   the labels scope is available; if not, log "labels scope denied"
   in errors and continue (Notion URL dedup is the safety net).
5. Write $RUN_STATE_DIR/alerts.json with this shape:

   {
     "phase": "alerts",
     "started_at": "<ISO-8601 UTC>",
     "finished_at": "<ISO-8601 UTC>",
     "exit_code": 0,
     "threads_processed": <count of Gmail threads inspected>,
     "urls_extracted": <count of unique listing URLs after dedup>,
     "inserted_pages": [
       {"page_id": "...", "tier": "...", "score": ..., "title": "...", "url": "..."},
       ... one entry per Notion row created ...
     ],
     "errors": ["<one-line error desc>", ...]
   }

DO NOT:
- Run scraping (no Playwright).
- Stamp Meta last_local_run.
- Compose the email digest.
- Write outreach .txt files.
PROMPT
```

- [ ] **Step 4: Write scrape.txt**

```bash
cat > /home/ara/hunt/prompts/scrape.txt <<'PROMPT'
You are the SCRAPE subagent for $PORTAL $TIER. Your scope: process
EXACTLY $N portal-area URLs.

Skill reference:
/home/ara/dev/london-flat-hunt/london-property-hunt-public/skill-flats.md
— sections § HARD FILTERS, § SIZE RESOLUTION, § SCORING,
§ Scraping loop (per portal-area search URL),
§ VISUAL EXTRACTION — FLOORPLAN, § VISUAL EXTRACTION — PHOTOS,
§ TRACKER, § Pagination cutoff, § Rate limiting.

Configuration:
- Phase name: $PHASE_NAME
- Portal: $PORTAL
- Tier: $TIER
- Number of areas to process: $N
- Flats data source ID: $FLATS_DATA_SOURCE_ID
- Run-state directory: $RUN_STATE_DIR
- Known URL set file: $RUN_STATE_DIR/known_urls.txt

Areas (process in order):
$AREAS_BLOCK

INIT-failure check (DO THIS FIRST):
If $RUN_STATE_DIR/init.failed exists, write
$RUN_STATE_DIR/$PHASE_NAME.json with all areas marked
SKIPPED-INIT-FAILED and exit. Use this template:

   {
     "phase": "$PHASE_NAME",
     "started_at": "<ISO-8601 UTC>",
     "finished_at": "<ISO-8601 UTC>",
     "exit_code": 0,
     "areas": [
       {"area": "<every area listed above>", "outcome": "SKIPPED-INIT-FAILED", "inserted": 0}
     ],
     "inserted_pages": [],
     "errors": []
   }

Do NOT call notion-create-pages under any circumstances when this
flag is present.

Required actions (when init has not failed):
1. Probe Playwright with mcp__playwright__browser_navigate to
   https://example.com (15s timeout). If it fails, mark all areas
   BLOCKED and exit cleanly with the JSON output below.
2. For each area in the list above, in order:
   a. Run § Scraping loop on the area's URL.
   b. Extract listing URLs via the portal-specific regex.
   c. For each listing URL:
      - Read $RUN_STATE_DIR/known_urls.txt and check membership; if
        present, treat as dedup hit.
      - If new, follow § Scraping loop step 7 (enrich, hard-filter,
        score, insert via notion-create-pages with
        parent.data_source_id = $FLATS_DATA_SOURCE_ID and
        Source = "scraped-local").
      - Immediately after each successful insert, append the
        canonical URL to $RUN_STATE_DIR/known_urls.txt.
   d. Determine the area's outcome:
      OK | EARLY-EXIT | BLOCKED | TIMEOUT | ZERO-RESULTS
3. Write $RUN_STATE_DIR/$PHASE_NAME.json with this exact shape:

   {
     "phase": "$PHASE_NAME",
     "started_at": "<ISO-8601 UTC>",
     "finished_at": "<ISO-8601 UTC>",
     "exit_code": 0,
     "areas": [
       {"area": "<area name>", "outcome": "<enum value>", "inserted": <n>},
       ... EXACTLY $N entries, one per assigned area, in the same order ...
     ],
     "inserted_pages": [
       {"page_id": "...", "tier": "...", "score": ..., "title": "...", "url": "..."},
       ... one entry per row inserted to Notion ...
     ],
     "errors": ["<one-line error desc>", ...]
   }

DO NOT:
- Process areas not in your assigned list.
- Skip areas in your list (every input area must produce one output
  entry; if you couldn't reach the URL, mark it BLOCKED or TIMEOUT).
- Process other portals or tiers.
- Run alert ingestion.
- Stamp Meta or send emails.
PROMPT
```

- [ ] **Step 5: Write digest.txt**

```bash
cat > /home/ara/hunt/prompts/digest.txt <<'PROMPT'
You are the DIGEST subagent for the London flat hunt. Your scope:
read all phase output files, compose the email digest, save outreach
.txt files for HIGH listings, and stamp Meta.

Skill reference:
/home/ara/dev/london-flat-hunt/london-property-hunt-public/skill-flats.md
— sections § TRACKER (Meta stamping), § EMAIL — SELF-CONTAINED,
PHONE-READY, § OUTREACH.

Configuration:
- Flats data source ID: $FLATS_DATA_SOURCE_ID
- Meta data source ID: $META_DATA_SOURCE_ID
- Run-state directory: $RUN_STATE_DIR
- Outreach directory: $OUTREACH_DIR
- Run mode: $RUN_MODE (local or remote)
- Email recipient: $FLAT_EMAIL_TO

Required actions:
1. Read all phase JSON files in $RUN_STATE_DIR:
   - init.json
   - alerts.json
   - scrape-rm-primary.json, scrape-zoopla-primary.json,
     scrape-sr-primary.json, scrape-or-primary.json
   - scrape-rm-secondary.json, scrape-zoopla-secondary.json,
     scrape-sr-secondary.json, scrape-or-secondary.json
2. Concatenate all phases' areas[] entries into the per-area
   accounting block. If any phase JSON is missing or has
   exit_code != 0, surface it in a "## Failed phases" block in the
   email body.
3. Aggregate inserted_pages from every phase. Group by tier (HIGH,
   MEDIUM, LOW). Render the tier breakdown in the email subject and
   body per § EMAIL.
4. For every HIGH-tier inserted page, save an outreach .txt file to
   $OUTREACH_DIR/$(date +%Y-%m-%d)_<slug>.txt using the template in
   § OUTREACH.
5. Stamp Meta last_local_run (if RUN_MODE=local) or last_remote_run
   (if RUN_MODE=remote) to today's date in Europe/Zurich. Do this
   ONLY if at least one phase succeeded; if all phases failed,
   leave Meta untouched so the next run retries.
6. Compose the email digest per § EMAIL and create a Gmail draft
   via mcp__claude_ai_Gmail__create_draft with toRecipients =
   $FLAT_EMAIL_TO. Do NOT auto-send.
7. Write $RUN_STATE_DIR/digest.json with this shape:

   {
     "phase": "digest",
     "started_at": "<ISO-8601 UTC>",
     "finished_at": "<ISO-8601 UTC>",
     "exit_code": 0,
     "draft_id": "<Gmail draft id, or null if INIT failed>",
     "subject": "<email subject>",
     "tier_counts": {"HIGH": N, "MEDIUM": N, "LOW": N},
     "outreach_files": ["path1", "path2", ...],
     "meta_stamped": true | false,
     "failed_phases": ["alerts", "scrape-zoopla-primary", ...],
     "errors": ["<one-line error desc>", ...]
   }

If $RUN_STATE_DIR/init.failed exists:
- DO NOT stamp Meta (leave for next run).
- Compose a brief email digest with subject "London Flat Hunt — <date> — INIT FAILED" and a body explaining that dedup state was unavailable so all writes were skipped this run. Still create the Gmail draft.
- Write digest.json with meta_stamped=false and a clear error.

DO NOT:
- Run scraping.
- Read Gmail alerts.
- Insert listings to Notion (those should already exist from earlier phases).
- Auto-send the email (always leave as draft).
PROMPT
```

- [ ] **Step 6: Verify all four files exist and aren't empty**

Run:
```bash
ls -la /home/ara/hunt/prompts/
wc -l /home/ara/hunt/prompts/*.txt
```

Expected: 4 files (`init.txt`, `alerts.txt`, `scrape.txt`, `digest.txt`), each at least 30 lines.

- [ ] **Step 7: No commit (files in `~/hunt/`, gitignored)**

---

## Task 3: Add run-id generation and run-state directory management to run-hunt.sh

**Files:**
- Modify: `/home/ara/hunt/run-hunt.sh`

**Why:** Every run needs a unique state directory; old dirs need pruning so the disk doesn't fill up. This is preparatory infrastructure before the per-step orchestration.

- [ ] **Step 1: Read the current run-hunt.sh to confirm baseline**

Run:
```bash
cat /home/ara/hunt/run-hunt.sh
```

Expected: 40-line script with health-check loop and a single `xvfb-run … timeout 90m claude "Run the London flat hunt skill (local mode)"` invocation. Save the existing content mentally; we're rewriting it.

- [ ] **Step 2: Create a backup of the current script**

```bash
cp /home/ara/hunt/run-hunt.sh /home/ara/hunt/run-hunt.sh.pre-orchestrator
```

This lets us roll back if anything goes wrong during testing.

- [ ] **Step 3: Write the new run-hunt.sh skeleton with run-id generation and pruning**

```bash
cat > /home/ara/hunt/run-hunt.sh <<'SCRIPT'
#!/bin/bash
# London Flat Hunt cron orchestrator.
# 11-step pipeline: INIT → ALERTS → 8 SCRAPE-portal-tier → DIGEST.
# Each step is its own claude -p invocation with narrow scope.
# State shared via files in ~/hunt/run-state/<run-id>/.
# Spec: docs/superpowers/specs/2026-04-27-bash-orchestrator-design.md

set -u

CLAUDE=/home/ara/.local/bin/claude
HUNT_DIR=/home/ara/hunt
LOG="$HUNT_DIR/cron.log"
PROMPTS="$HUNT_DIR/prompts"
AREAS_FILE="$HUNT_DIR/areas.json"
RUN_STATE_BASE="$HUNT_DIR/run-state"
PER_STEP_TIMEOUT=25m
KILL_AFTER=30s

# ---------- run-state setup ----------
RUN_ID="$(date -u +%Y-%m-%dT%H-%M-%S)"
RUN_DIR="$RUN_STATE_BASE/$RUN_ID"
mkdir -p "$RUN_DIR/log"
ln -sfn "$RUN_DIR" "$RUN_STATE_BASE/latest"

# Initialise empty known_urls.txt and timings.json so phase prompts can append.
: > "$RUN_DIR/known_urls.txt"
echo '{"steps":[]}' > "$RUN_DIR/timings.json"

# Prune run-state dirs older than 14 days.
find "$RUN_STATE_BASE" -maxdepth 1 -mindepth 1 -type d -mtime +14 -exec rm -rf {} +

exec >> "$LOG" 2>&1
echo
echo "=== CRON ORCHESTRATOR START $(date -Iseconds) RUN_ID=$RUN_ID ==="

# ---------- health check (keep existing) ----------
for i in 1 2 3 4 5; do
  count=$("$CLAUDE" -p "Use ToolSearch to count tools matching 'notion-search'. Output only the integer." --max-turns 2 2>/dev/null | grep -oE '^[0-9]+' | head -1)
  echo "[health-check #$i] notion-search tool count: ${count:-<empty>}"
  if [[ "${count:-0}" =~ ^[1-9] ]]; then
    echo "[health-check] claude.ai MCPs ready after $i attempt(s)"
    break
  fi
  if [[ $i -lt 5 ]]; then
    sleep_s=$((i * 30))
    echo "[health-check] sleeping ${sleep_s}s before retry"
    sleep $sleep_s
  fi
done

if ! [[ "${count:-0}" =~ ^[1-9] ]]; then
  echo "[health-check] FAILED after 5 attempts — running steps anyway (each step bails on missing MCPs)"
fi

# Tasks 4-10 will append step-dispatch logic below this line.

echo "=== CRON ORCHESTRATOR END $(date -Iseconds) ==="
SCRIPT
chmod +x /home/ara/hunt/run-hunt.sh
```

- [ ] **Step 4: Verify the skeleton runs and creates a run-state dir**

Run:
```bash
/home/ara/hunt/run-hunt.sh
ls -la /home/ara/hunt/run-state/
readlink /home/ara/hunt/run-state/latest
ls -la "$(readlink -f /home/ara/hunt/run-state/latest)"
```

Expected:
- A new directory under `run-state/` with name like `2026-04-27T22-15-03`
- `latest` symlink points to it
- Inside: `log/` subdir, empty `known_urls.txt`, `timings.json` containing `{"steps":[]}`
- The script ran the health check (5 attempts) and exited cleanly

- [ ] **Step 5: Verify pruning logic by faking an old dir**

Run:
```bash
mkdir -p /home/ara/hunt/run-state/2026-03-01T00-00-00
touch -d '20 days ago' /home/ara/hunt/run-state/2026-03-01T00-00-00
ls /home/ara/hunt/run-state/ | grep 2026-03-01
/home/ara/hunt/run-hunt.sh
ls /home/ara/hunt/run-state/ | grep 2026-03-01 || echo "pruned correctly"
```

Expected: the old dir is listed before the run; after the run, the grep returns nothing and "pruned correctly" prints.

- [ ] **Step 6: No commit (file in `~/hunt/`, gitignored)**

---

## Task 4: Add the `run_step` helper function

**Files:**
- Modify: `/home/ara/hunt/run-hunt.sh`

**Why:** Every phase has the same shape — render a prompt, run `claude -p` with timeout, capture output, validate JSON, log timings, mark completion or failure. Putting it in one function avoids duplicating ~30 lines of boilerplate eight times.

- [ ] **Step 1: Add the `run_step` function and helpers above the health-check section**

Edit `/home/ara/hunt/run-hunt.sh` and insert these helper functions immediately after the line `RUN_STATE_BASE="$HUNT_DIR/run-state"` (i.e. after the variable block, before `RUN_ID=` is set). Replace the existing variable block + setup so it ends up looking like this:

```bash
CLAUDE=/home/ara/.local/bin/claude
HUNT_DIR=/home/ara/hunt
LOG="$HUNT_DIR/cron.log"
PROMPTS="$HUNT_DIR/prompts"
AREAS_FILE="$HUNT_DIR/areas.json"
RUN_STATE_BASE="$HUNT_DIR/run-state"
PER_STEP_TIMEOUT=25m
KILL_AFTER=30s

# Notion config (reads from ~/hunt/config.md so we have one source of truth)
FLATS_DATA_SOURCE_ID=$(grep '^FLAT_TRACKER_FLATS_DATA_SOURCE_ID=' "$HUNT_DIR/config.md" | cut -d= -f2 | tr -d ' \r')
META_DATA_SOURCE_ID=$(grep '^FLAT_TRACKER_META_DATA_SOURCE_ID=' "$HUNT_DIR/config.md" | cut -d= -f2 | tr -d ' \r')
FLAT_EMAIL_TO=$(grep '^FLAT_EMAIL_TO=' "$HUNT_DIR/config.md" | cut -d= -f2 | cut -d'#' -f1 | tr -d ' \r')
OUTREACH_DIR="$HUNT_DIR/outreach"

# ---------- run_step helper ----------
# Args:
#   $1 = phase name (e.g. "init", "alerts", "scrape-rm-primary")
#   $2 = path to prompt file (already-rendered, ready to pipe)
# Side effects:
#   - Runs claude -p with $PER_STEP_TIMEOUT and writes log to $RUN_DIR/log/<step>.log
#   - Appends timing entry to $RUN_DIR/timings.json
#   - Validates the phase JSON (parse + presence)
# Returns:
#   0 on success (claude exited 0 AND $RUN_DIR/<phase>.json parses cleanly)
#   1 on JSON parse failure
#   124 on timeout (passthrough from `timeout` command)
#   non-zero on other claude exit codes
run_step() {
  local phase="$1"
  local prompt_path="$2"
  local out_json="$RUN_DIR/$phase.json"
  local step_log="$RUN_DIR/log/$phase.log"
  local start_ts end_ts duration rc

  start_ts=$(date -Iseconds)
  echo "=== STEP $phase START $start_ts ===" | tee -a "$LOG"

  # Pipe rendered prompt into claude -p; capture full stdout to step_log.
  cat "$prompt_path" | xvfb-run -a --server-args="-screen 0 1920x1080x24" \
    timeout --signal=TERM --kill-after="$KILL_AFTER" "$PER_STEP_TIMEOUT" \
    "$CLAUDE" -p --output-format text > "$step_log" 2>&1
  rc=$?

  end_ts=$(date -Iseconds)
  duration=$(( $(date -d "$end_ts" +%s) - $(date -d "$start_ts" +%s) ))

  # Append to timings.json (jq atomic-rewrite pattern).
  jq --arg phase "$phase" --arg start "$start_ts" --arg end "$end_ts" \
     --argjson dur "$duration" --argjson rc "$rc" \
     '.steps += [{"phase":$phase, "start":$start, "end":$end, "duration_s":$dur, "exit_code":$rc}]' \
     "$RUN_DIR/timings.json" > "$RUN_DIR/timings.json.tmp" && \
     mv "$RUN_DIR/timings.json.tmp" "$RUN_DIR/timings.json"

  # If claude succeeded, validate the JSON exists and parses.
  if [[ $rc -eq 0 ]]; then
    if [[ ! -f "$out_json" ]]; then
      echo "[$phase] FAIL: claude exited 0 but $out_json was not written" | tee -a "$LOG"
      jq -n --arg phase "$phase" --arg start "$start_ts" --arg end "$end_ts" \
        '{phase:$phase, started_at:$start, finished_at:$end, exit_code:1, errors:["output JSON not written"]}' \
        > "$out_json"
      rc=1
    elif ! jq . "$out_json" >/dev/null 2>&1; then
      echo "[$phase] FAIL: $out_json is not valid JSON" | tee -a "$LOG"
      mv "$out_json" "$out_json.invalid"
      jq -n --arg phase "$phase" --arg start "$start_ts" --arg end "$end_ts" \
        '{phase:$phase, started_at:$start, finished_at:$end, exit_code:1, errors:["output JSON did not parse"]}' \
        > "$out_json"
      rc=1
    fi
  fi

  # If claude failed (timeout, error), write a placeholder JSON so DIGEST can still process.
  if [[ $rc -ne 0 ]] && [[ ! -f "$out_json" ]]; then
    jq -n --arg phase "$phase" --arg start "$start_ts" --arg end "$end_ts" \
       --argjson rc "$rc" \
       '{phase:$phase, started_at:$start, finished_at:$end, exit_code:$rc, errors:["step failed; see log"]}' \
       > "$out_json"
  fi

  echo "=== STEP $phase END exit=$rc duration=${duration}s ===" | tee -a "$LOG"
  return $rc
}

# ---------- run-state setup ----------
RUN_ID="$(date -u +%Y-%m-%dT%H-%M-%S)"
RUN_DIR="$RUN_STATE_BASE/$RUN_ID"
mkdir -p "$RUN_DIR/log"
ln -sfn "$RUN_DIR" "$RUN_STATE_BASE/latest"

# Initialise empty known_urls.txt and timings.json so phase prompts can append.
: > "$RUN_DIR/known_urls.txt"
echo '{"steps":[]}' > "$RUN_DIR/timings.json"

# Prune run-state dirs older than 14 days.
find "$RUN_STATE_BASE" -maxdepth 1 -mindepth 1 -type d -mtime +14 -exec rm -rf {} +
```

- [ ] **Step 2: Verify the variable extraction picks up the right values**

Run (no claude invocation, just sourcing the variable lines):
```bash
bash -c '
  HUNT_DIR=/home/ara/hunt
  FLATS=$(grep "^FLAT_TRACKER_FLATS_DATA_SOURCE_ID=" "$HUNT_DIR/config.md" | cut -d= -f2 | tr -d " \r")
  META=$(grep "^FLAT_TRACKER_META_DATA_SOURCE_ID=" "$HUNT_DIR/config.md" | cut -d= -f2 | tr -d " \r")
  EMAIL=$(grep "^FLAT_EMAIL_TO=" "$HUNT_DIR/config.md" | cut -d= -f2 | cut -d"#" -f1 | tr -d " \r")
  echo "FLATS=$FLATS"
  echo "META=$META"
  echo "EMAIL=$EMAIL"
'
```

Expected:
```
FLATS=c417b1a4-2ebd-4686-85ec-ca1219a8b0d4
META=e914a179-d983-4730-b863-97d188843f90
EMAIL=fruiteam@gmail.com
```

If any value is empty, the grep/cut chain has a quoting issue. Fix before continuing.

- [ ] **Step 3: Verify run-hunt.sh still parses without errors**

Run:
```bash
bash -n /home/ara/hunt/run-hunt.sh && echo "syntax OK"
```

Expected: `syntax OK`. If a parse error prints, fix the script.

- [ ] **Step 4: No commit (file in `~/hunt/`, gitignored)**

---

## Task 5: Wire up the INIT step

**Files:**
- Modify: `/home/ara/hunt/run-hunt.sh`

**Why:** First real phase. Must render init.txt with `$RUN_STATE_DIR` and `$FLATS_DATA_SOURCE_ID` substituted, dispatch via `run_step`, and write `init.failed` if it fails.

- [ ] **Step 1: Add the INIT dispatch block**

Append to `/home/ara/hunt/run-hunt.sh` immediately before the `echo "=== CRON ORCHESTRATOR END ..." ===` line:

```bash
# ---------- STEP 1: INIT ----------
INIT_PROMPT_RENDERED="$RUN_DIR/log/init.prompt"
RUN_STATE_DIR="$RUN_DIR" \
FLATS_DATA_SOURCE_ID="$FLATS_DATA_SOURCE_ID" \
envsubst '$RUN_STATE_DIR $FLATS_DATA_SOURCE_ID' < "$PROMPTS/init.txt" > "$INIT_PROMPT_RENDERED"

run_step "init" "$INIT_PROMPT_RENDERED"
init_rc=$?

if [[ $init_rc -ne 0 ]]; then
  touch "$RUN_DIR/init.failed"
  echo "[orchestrator] INIT failed (rc=$init_rc) — created init.failed flag; downstream phases will skip writes" | tee -a "$LOG"
fi
```

- [ ] **Step 2: Verify envsubst is installed**

Run:
```bash
which envsubst && envsubst --version | head -1
```

Expected: `/usr/bin/envsubst` and a version line. If `envsubst` is missing, install with `sudo apt install gettext-base` and retry.

- [ ] **Step 3: Run the script and check INIT output**

Run:
```bash
/home/ara/hunt/run-hunt.sh
LATEST=$(readlink -f /home/ara/hunt/run-state/latest)
echo "Run dir: $LATEST"
ls -la "$LATEST"
echo "--- init.json ---"
jq . "$LATEST/init.json" | head -30
echo "--- known_urls.txt (first 5 + count) ---"
head -5 "$LATEST/known_urls.txt"
wc -l "$LATEST/known_urls.txt"
echo "--- init.failed flag? ---"
ls "$LATEST/init.failed" 2>/dev/null && echo "FLAG PRESENT" || echo "no flag (init succeeded)"
```

Expected:
- Run dir contains `init.json`, `known_urls.txt` (with at least 100 URLs from previous runs' rows), `log/init.log`
- `init.json` parses, `exit_code` is 0, `url_count` matches `wc -l` of known_urls.txt
- No `init.failed` flag
- The whole step completes in under 6 minutes

- [ ] **Step 4: Test the init-failed path by simulating a Notion outage**

Temporarily break the Notion MCP config and re-run:
```bash
/home/ara/.local/bin/claude mcp remove "claude.ai Notion"
/home/ara/hunt/run-hunt.sh
LATEST=$(readlink -f /home/ara/hunt/run-state/latest)
ls -la "$LATEST"
cat "$LATEST/init.json"
```

Expected:
- INIT step exits non-zero (timeout or error from missing Notion tools)
- `init.failed` flag is created
- `init.json` records the failure

Then restore Notion:
```bash
/home/ara/.local/bin/claude mcp add "claude.ai Notion" --transport http --scope user --url https://mcp.notion.com/mcp
/home/ara/.local/bin/claude mcp list | grep Notion
```

Expected: `claude.ai Notion: ✓ Connected`. If reconnection requires interactive auth, defer this destructive test until the rest of the orchestrator is built; the init.failed code path is straightforward enough to verify manually with `touch /home/ara/hunt/run-state/<some-test-id>/init.failed` and confirming downstream phases see it.

- [ ] **Step 5: No commit (file in `~/hunt/`, gitignored)**

---

## Task 6: Wire up the ALERTS step

**Files:**
- Modify: `/home/ara/hunt/run-hunt.sh`

**Why:** Second phase. Reads Gmail, dedups against the URL set INIT produced, inserts qualified rows.

- [ ] **Step 1: Add the ALERTS dispatch block**

Append after the INIT block in `/home/ara/hunt/run-hunt.sh`, before the final `echo "=== CRON ORCHESTRATOR END ..." ===` line:

```bash
# ---------- STEP 2: ALERTS ----------
ALERTS_PROMPT_RENDERED="$RUN_DIR/log/alerts.prompt"
RUN_STATE_DIR="$RUN_DIR" \
FLATS_DATA_SOURCE_ID="$FLATS_DATA_SOURCE_ID" \
envsubst '$RUN_STATE_DIR $FLATS_DATA_SOURCE_ID' < "$PROMPTS/alerts.txt" > "$ALERTS_PROMPT_RENDERED"

run_step "alerts" "$ALERTS_PROMPT_RENDERED"
```

- [ ] **Step 2: Verify the script parses**

```bash
bash -n /home/ara/hunt/run-hunt.sh && echo "syntax OK"
```

Expected: `syntax OK`.

- [ ] **Step 3: Run and inspect alerts.json**

```bash
/home/ara/hunt/run-hunt.sh
LATEST=$(readlink -f /home/ara/hunt/run-state/latest)
echo "--- alerts.json summary ---"
jq '{phase, exit_code, threads_processed, urls_extracted, inserts: (.inserted_pages|length), errors: (.errors|length)}' "$LATEST/alerts.json"
echo "--- known_urls.txt grew by ---"
wc -l "$LATEST/known_urls.txt"
echo "--- alerts.log tail ---"
tail -20 "$LATEST/log/alerts.log"
```

Expected:
- `alerts.json` exists with `exit_code: 0`
- `threads_processed` matches roughly the number of unread alert threads in the past 2 days (likely 5-30)
- `inserts` count is plausible (0-20)
- `known_urls.txt` line count has grown if any inserts happened
- The alerts log shows Gmail searches, no permission errors

- [ ] **Step 4: No commit**

---

## Task 7: Wire up the 8 SCRAPE steps in order

**Files:**
- Modify: `/home/ara/hunt/run-hunt.sh`

**Why:** The core of this redesign. Each scrape step assembles its prompt from `scrape.txt` plus the area list from `areas.json` plus runtime values, then dispatches via `run_step`.

- [ ] **Step 1: Add a helper function for scrape dispatch**

Insert this function just below the `run_step` definition (still in the helper block of `run-hunt.sh`):

```bash
# ---------- run_scrape helper ----------
# Args:
#   $1 = phase key in areas.json (e.g. "rm-primary")
# Renders scrape.txt with the area list from areas.json and dispatches.
run_scrape() {
  local key="$1"
  local phase="scrape-$key"
  local prompt_rendered="$RUN_DIR/log/$phase.prompt"

  # Build AREAS_BLOCK as one "  N. <area> | <url>" line per area, indented.
  local areas_block
  areas_block=$(jq -r --arg key "$key" '
    .[$key]
    | to_entries
    | map("  \(.key + 1). \(.value.area) | \(.value.url)")
    | .[]
  ' "$AREAS_FILE")

  local n
  n=$(jq -r --arg key "$key" '.[$key] | length' "$AREAS_FILE")

  # Determine portal and tier from the key.
  local portal tier
  case "$key" in
    rm-primary|rm-secondary)         portal="Rightmove" ;;
    zoopla-primary|zoopla-secondary) portal="Zoopla" ;;
    sr-primary|sr-secondary)         portal="SpareRoom" ;;
    or-primary|or-secondary)         portal="OpenRent" ;;
    *) echo "[run_scrape] unknown key: $key" | tee -a "$LOG"; return 2 ;;
  esac
  case "$key" in
    *-primary)   tier="primary" ;;
    *-secondary) tier="secondary" ;;
  esac

  RUN_STATE_DIR="$RUN_DIR" \
  FLATS_DATA_SOURCE_ID="$FLATS_DATA_SOURCE_ID" \
  PHASE_NAME="$phase" \
  PORTAL="$portal" \
  TIER="$tier" \
  N="$n" \
  AREAS_BLOCK="$areas_block" \
  envsubst '$RUN_STATE_DIR $FLATS_DATA_SOURCE_ID $PHASE_NAME $PORTAL $TIER $N $AREAS_BLOCK' \
    < "$PROMPTS/scrape.txt" > "$prompt_rendered"

  run_step "$phase" "$prompt_rendered"
}
```

- [ ] **Step 2: Add the 8 scrape dispatches in order**

Append to `/home/ara/hunt/run-hunt.sh` after the ALERTS block, before the final `echo "=== CRON ORCHESTRATOR END ..."`:

```bash
# ---------- STEPS 3-10: SCRAPE × 8 ----------
for key in rm-primary zoopla-primary sr-primary or-primary \
           rm-secondary zoopla-secondary sr-secondary or-secondary; do
  run_scrape "$key"
done
```

- [ ] **Step 3: Verify the prompt renders correctly for one phase before doing a full run**

Run:
```bash
RUN_DIR=/tmp/test-scrape-render-$$
mkdir -p "$RUN_DIR/log"

key="rm-primary"
phase="scrape-$key"

areas_block=$(jq -r --arg key "$key" '
  .[$key]
  | to_entries
  | map("  \(.key + 1). \(.value.area) | \(.value.url)")
  | .[]
' /home/ara/hunt/areas.json)

n=$(jq -r --arg key "$key" '.[$key] | length' /home/ara/hunt/areas.json)

RUN_STATE_DIR="$RUN_DIR" \
FLATS_DATA_SOURCE_ID="c417b1a4-2ebd-4686-85ec-ca1219a8b0d4" \
PHASE_NAME="$phase" \
PORTAL="Rightmove" \
TIER="primary" \
N="$n" \
AREAS_BLOCK="$areas_block" \
envsubst '$RUN_STATE_DIR $FLATS_DATA_SOURCE_ID $PHASE_NAME $PORTAL $TIER $N $AREAS_BLOCK' \
  < /home/ara/hunt/prompts/scrape.txt | head -30

rm -rf "$RUN_DIR"
```

Expected: the rendered prompt's first 30 lines show:
- "You are the SCRAPE subagent for Rightmove primary"
- "process EXACTLY 6 portal-area URLs"
- An "Areas (process in order):" block with 6 numbered entries (Islington, Highbury, Camden Town, Kentish Town, De Beauvoir Town, Clerkenwell), each with its full URL
- All `$VAR` placeholders are substituted (no literal `$RUN_STATE_DIR` etc.)

- [ ] **Step 4: Run the full pipeline and verify all 8 scrape JSONs are produced**

Run:
```bash
/home/ara/hunt/run-hunt.sh
LATEST=$(readlink -f /home/ara/hunt/run-state/latest)
echo "--- per-phase summary ---"
for f in "$LATEST"/scrape-*.json; do
  jq -r --arg f "$(basename "$f")" '"\($f): exit=\(.exit_code) areas=\(.areas|length) inserts=\(.inserted_pages|length)"' "$f"
done
echo "--- total inserts across all scrape phases ---"
jq -s 'map(.inserted_pages) | flatten | length' "$LATEST"/scrape-*.json
echo "--- timings ---"
jq -r '.steps[] | "\(.phase): \(.duration_s)s exit=\(.exit_code)"' "$LATEST/timings.json"
```

Expected:
- 8 `scrape-*.json` files exist
- Each has `exit=0` and an `areas` array with the right length (6, 8, 8, 1, 8, 8, 8, 1)
- Sum of inserts is plausible (could be 0-50 depending on dedup state)
- Timings show all 11 step entries (1 init + 1 alerts + 8 scrape + 0 digest yet)
- Each scrape step is under 25 min wall-clock

- [ ] **Step 5: No commit**

---

## Task 8: Wire up the DIGEST step

**Files:**
- Modify: `/home/ara/hunt/run-hunt.sh`

**Why:** Final phase. Reads all `*.json` files, composes the email, saves outreach .txt files, stamps Meta.

- [ ] **Step 1: Add the DIGEST dispatch block**

Append after the scrape loop in `/home/ara/hunt/run-hunt.sh`, before the final `echo "=== CRON ORCHESTRATOR END ..."`:

```bash
# ---------- STEP 11: DIGEST ----------
DIGEST_PROMPT_RENDERED="$RUN_DIR/log/digest.prompt"
RUN_STATE_DIR="$RUN_DIR" \
FLATS_DATA_SOURCE_ID="$FLATS_DATA_SOURCE_ID" \
META_DATA_SOURCE_ID="$META_DATA_SOURCE_ID" \
OUTREACH_DIR="$OUTREACH_DIR" \
RUN_MODE="local" \
FLAT_EMAIL_TO="$FLAT_EMAIL_TO" \
envsubst '$RUN_STATE_DIR $FLATS_DATA_SOURCE_ID $META_DATA_SOURCE_ID $OUTREACH_DIR $RUN_MODE $FLAT_EMAIL_TO' \
  < "$PROMPTS/digest.txt" > "$DIGEST_PROMPT_RENDERED"

run_step "digest" "$DIGEST_PROMPT_RENDERED"
```

- [ ] **Step 2: Run the full pipeline end-to-end**

```bash
/home/ara/hunt/run-hunt.sh
LATEST=$(readlink -f /home/ara/hunt/run-state/latest)
echo "--- digest.json summary ---"
jq '{phase, exit_code, draft_id, subject, tier_counts, outreach_count: (.outreach_files|length), failed_phases, errors_count: (.errors|length)}' "$LATEST/digest.json"
echo "--- outreach files written this run ---"
ls -la "$LATEST"/digest.json
ls /home/ara/hunt/outreach/ | tail -5
```

Expected:
- `digest.json` has `exit_code: 0`, a non-null `draft_id`, a subject like "🏠 London Flat Hunt — 2026-04-28 — N new (H:N/M:N/L:N)"
- `tier_counts` matches the sum of inserted_pages from earlier phases grouped by tier
- One outreach .txt file per HIGH listing in `~/hunt/outreach/`
- `failed_phases` is `[]` if everything ran clean, or contains phase names if any failed

- [ ] **Step 3: Verify a Gmail draft was created and the Meta stamp updated**

```bash
LATEST=$(readlink -f /home/ara/hunt/run-state/latest)
DRAFT_ID=$(jq -r '.draft_id' "$LATEST/digest.json")
echo "Draft ID: $DRAFT_ID — open https://mail.google.com/mail/u/0/#drafts to verify"
# Then check Meta last_local_run via the Notion MCP — easiest from another claude -p invocation:
/home/ara/.local/bin/claude -p "Use mcp__claude_ai_Notion__notion-search with query 'last_local_run' and data_source_url 'collection://e914a179-d983-4730-b863-97d188843f90' and filters {} to find the page; report only its Value property." --max-turns 5
```

Expected:
- Draft ID is a long string starting with `r-` or similar
- The Notion query returns today's date as the `last_local_run` Value

- [ ] **Step 4: No commit**

---

## Task 9: Add Bash-side completeness validation

**Files:**
- Modify: `/home/ara/hunt/run-hunt.sh`

**Why:** Per the spec § 6 Bash-side completeness validation, after each scrape phase, Bash must check the output JSON's `areas` count matches the input N and every `outcome` is in the documented enum. This catches silent per-area dropping by the model.

- [ ] **Step 1: Add a `validate_scrape_phase` helper**

Insert this function below `run_scrape` in `/home/ara/hunt/run-hunt.sh`:

```bash
# ---------- validate_scrape_phase helper ----------
# Args:
#   $1 = phase key (e.g. "rm-primary")
# Reads expected N from areas.json and verifies $RUN_DIR/scrape-$key.json:
#   - is parseable JSON (already done by run_step, but double-check)
#   - has areas array of length == N
#   - every areas[*].outcome ∈ {OK, EARLY-EXIT, BLOCKED, TIMEOUT, ZERO-RESULTS, SKIPPED-INIT-FAILED}
# Logs INVALID line on mismatch and writes $RUN_DIR/scrape-$key.json.invalid as a sidecar.
validate_scrape_phase() {
  local key="$1"
  local phase="scrape-$key"
  local out_json="$RUN_DIR/$phase.json"
  local expected_n actual_n bad_outcomes

  expected_n=$(jq -r --arg key "$key" '.[$key] | length' "$AREAS_FILE")
  actual_n=$(jq -r '.areas | length' "$out_json" 2>/dev/null || echo 0)

  if [[ "$expected_n" != "$actual_n" ]]; then
    echo "[$phase] INVALID: expected $expected_n areas, got $actual_n" | tee -a "$LOG"
    cp "$out_json" "$out_json.invalid"
    return 1
  fi

  bad_outcomes=$(jq -r '.areas[] | select(.outcome | test("^(OK|EARLY-EXIT|BLOCKED|TIMEOUT|ZERO-RESULTS|SKIPPED-INIT-FAILED)$") | not) | .outcome' "$out_json")
  if [[ -n "$bad_outcomes" ]]; then
    echo "[$phase] INVALID: out-of-vocabulary outcomes: $bad_outcomes" | tee -a "$LOG"
    cp "$out_json" "$out_json.invalid"
    return 1
  fi

  return 0
}
```

- [ ] **Step 2: Wire validation into the scrape loop**

Replace the existing scrape-loop block:
```bash
for key in rm-primary zoopla-primary sr-primary or-primary \
           rm-secondary zoopla-secondary sr-secondary or-secondary; do
  run_scrape "$key"
done
```

with:
```bash
for key in rm-primary zoopla-primary sr-primary or-primary \
           rm-secondary zoopla-secondary sr-secondary or-secondary; do
  run_scrape "$key"
  validate_scrape_phase "$key" || true   # don't abort the run on validation failure; flag and continue
done
```

The `|| true` keeps the loop going so DIGEST still runs and surfaces all problems together.

- [ ] **Step 3: Verify the validator catches a fake mismatch**

Create a fake bad scrape JSON, run the validator, then clean up:
```bash
LATEST=$(readlink -f /home/ara/hunt/run-state/latest)
# Backup the real one
cp "$LATEST/scrape-rm-primary.json" "$LATEST/scrape-rm-primary.json.real"
# Inject a missing area
jq '.areas |= [.[0]]' "$LATEST/scrape-rm-primary.json.real" > "$LATEST/scrape-rm-primary.json"
# Source the function and call it
RUN_DIR="$LATEST" AREAS_FILE=/home/ara/hunt/areas.json LOG=/dev/null bash -c "
  source <(sed -n '/^validate_scrape_phase()/,/^}$/p' /home/ara/hunt/run-hunt.sh)
  validate_scrape_phase rm-primary
"
echo "exit=$?"
ls "$LATEST/scrape-rm-primary.json.invalid" 2>/dev/null && echo "sidecar created"
# Restore
mv "$LATEST/scrape-rm-primary.json.real" "$LATEST/scrape-rm-primary.json"
rm -f "$LATEST/scrape-rm-primary.json.invalid"
```

Expected:
- The validator prints an `INVALID: expected 6 areas, got 1` line
- Exit code is 1
- A `scrape-rm-primary.json.invalid` sidecar file is created
- After cleanup, the real JSON is restored

- [ ] **Step 4: No commit**

---

## Task 10: Add bootstrap retry on cold-start MCP failures

**Files:**
- Modify: `/home/ara/hunt/run-hunt.sh`

**Why:** Per the spec § 6, each step's first 60 seconds are cold-start; if the first MCP call fails inside that window with a connection or auth error, Bash kills and re-runs once. This guards against transient claude.ai connector fetch failures (the same class of issue we already guard against at session-start with the health-check loop, but now scoped per-step because each `claude -p` is a fresh session).

The simplest implementation: detect the failure signature in the step log (specific error strings), and if the run took < 90s AND the log contains those signatures, retry once.

- [ ] **Step 1: Add a `should_retry_step` helper**

Insert below `validate_scrape_phase` in `/home/ara/hunt/run-hunt.sh`:

```bash
# ---------- should_retry_step helper ----------
# Args:
#   $1 = phase name
# Reads $RUN_DIR/log/<phase>.log and the timings entry; returns 0 if the
# step looks like a cold-start MCP failure (failed inside 90s with a known
# connector or auth error string), 1 otherwise.
should_retry_step() {
  local phase="$1"
  local step_log="$RUN_DIR/log/$phase.log"
  local duration

  duration=$(jq -r --arg phase "$phase" '
    [.steps[] | select(.phase == $phase)] | last | .duration_s // 9999
  ' "$RUN_DIR/timings.json")

  if [[ -z "$duration" ]] || [[ "$duration" -gt 90 ]]; then
    return 1
  fi

  # Failure signatures we know are cold-start / connector related.
  if grep -qE 'No matching deferred tools found|MCP server.*not connected|claude.ai.*connector|Authentication.*failed|API Error' "$step_log" 2>/dev/null; then
    return 0
  fi

  return 1
}
```

- [ ] **Step 2: Wire one-shot retry into `run_step`**

Modify the bottom of `run_step` so that if it fails, the helper checks for retryability and re-runs once. Replace the existing tail of `run_step` (everything from `# If claude succeeded, validate the JSON exists and parses.` onward) with:

```bash
  # If claude succeeded, validate the JSON exists and parses.
  if [[ $rc -eq 0 ]]; then
    if [[ ! -f "$out_json" ]]; then
      echo "[$phase] FAIL: claude exited 0 but $out_json was not written" | tee -a "$LOG"
      jq -n --arg phase "$phase" --arg start "$start_ts" --arg end "$end_ts" \
        '{phase:$phase, started_at:$start, finished_at:$end, exit_code:1, errors:["output JSON not written"]}' \
        > "$out_json"
      rc=1
    elif ! jq . "$out_json" >/dev/null 2>&1; then
      echo "[$phase] FAIL: $out_json is not valid JSON" | tee -a "$LOG"
      mv "$out_json" "$out_json.invalid"
      jq -n --arg phase "$phase" --arg start "$start_ts" --arg end "$end_ts" \
        '{phase:$phase, started_at:$start, finished_at:$end, exit_code:1, errors:["output JSON did not parse"]}' \
        > "$out_json"
      rc=1
    fi
  fi

  # Bootstrap retry for cold-start failures (max 1 retry per step).
  if [[ $rc -ne 0 ]] && should_retry_step "$phase"; then
    echo "[$phase] RETRY: cold-start failure detected — re-running once" | tee -a "$LOG"
    sleep 10  # give claude.ai connector layer a beat
    rm -f "$out_json"
    start_ts=$(date -Iseconds)
    cat "$prompt_path" | xvfb-run -a --server-args="-screen 0 1920x1080x24" \
      timeout --signal=TERM --kill-after="$KILL_AFTER" "$PER_STEP_TIMEOUT" \
      "$CLAUDE" -p --output-format text > "$step_log.retry" 2>&1
    rc=$?
    end_ts=$(date -Iseconds)
    duration=$(( $(date -d "$end_ts" +%s) - $(date -d "$start_ts" +%s) ))

    jq --arg phase "$phase-retry" --arg start "$start_ts" --arg end "$end_ts" \
       --argjson dur "$duration" --argjson rc "$rc" \
       '.steps += [{"phase":$phase, "start":$start, "end":$end, "duration_s":$dur, "exit_code":$rc}]' \
       "$RUN_DIR/timings.json" > "$RUN_DIR/timings.json.tmp" && \
       mv "$RUN_DIR/timings.json.tmp" "$RUN_DIR/timings.json"

    if [[ $rc -eq 0 ]] && [[ -f "$out_json" ]] && jq . "$out_json" >/dev/null 2>&1; then
      echo "[$phase] RETRY: succeeded after retry (rc=0)" | tee -a "$LOG"
    else
      echo "[$phase] RETRY: still failed (rc=$rc); giving up" | tee -a "$LOG"
    fi
  fi

  # If claude failed (timeout, error), write a placeholder JSON so DIGEST can still process.
  if [[ $rc -ne 0 ]] && [[ ! -f "$out_json" ]]; then
    jq -n --arg phase "$phase" --arg start "$start_ts" --arg end "$end_ts" \
       --argjson rc "$rc" \
       '{phase:$phase, started_at:$start, finished_at:$end, exit_code:$rc, errors:["step failed; see log"]}' \
       > "$out_json"
  fi

  echo "=== STEP $phase END exit=$rc duration=${duration}s ===" | tee -a "$LOG"
  return $rc
}
```

- [ ] **Step 3: Verify syntax still parses**

```bash
bash -n /home/ara/hunt/run-hunt.sh && echo "syntax OK"
```

Expected: `syntax OK`.

- [ ] **Step 4: Smoke-test the retry path with a synthetic cold-start failure**

The cleanest test is unit-style: set up a fake step log with a known signature, ask `should_retry_step` whether to retry. Run:
```bash
RUN_DIR=/tmp/test-retry-$$
mkdir -p "$RUN_DIR/log"
cat > "$RUN_DIR/log/fake-step.log" <<EOF
Some prelude
ToolSearch returned: No matching deferred tools found
exit
EOF
echo '{"steps":[{"phase":"fake-step","start":"2026-04-27T22:00:00+00:00","end":"2026-04-27T22:00:30+00:00","duration_s":30,"exit_code":1}]}' > "$RUN_DIR/timings.json"

bash -c "
  source <(sed -n '/^should_retry_step()/,/^}$/p' /home/ara/hunt/run-hunt.sh)
  RUN_DIR='$RUN_DIR'
  should_retry_step fake-step && echo retry || echo no-retry
"
rm -rf "$RUN_DIR"
```

Expected: prints `retry` (because the log contains "No matching deferred tools found" AND duration is 30s < 90s).

- [ ] **Step 5: Smoke-test the no-retry path (long-running step that legitimately failed)**

```bash
RUN_DIR=/tmp/test-noretry-$$
mkdir -p "$RUN_DIR/log"
cat > "$RUN_DIR/log/fake-step.log" <<EOF
Started normally
Did 4 of 6 areas
Got tired and stopped
EOF
echo '{"steps":[{"phase":"fake-step","start":"2026-04-27T22:00:00+00:00","end":"2026-04-27T22:20:00+00:00","duration_s":1200,"exit_code":1}]}' > "$RUN_DIR/timings.json"

bash -c "
  source <(sed -n '/^should_retry_step()/,/^}$/p' /home/ara/hunt/run-hunt.sh)
  RUN_DIR='$RUN_DIR'
  should_retry_step fake-step && echo retry || echo no-retry
"
rm -rf "$RUN_DIR"
```

Expected: prints `no-retry` (duration is 1200s > 90s, so retry is suppressed even if a signature were present).

- [ ] **Step 6: No commit**

---

## Task 11: Strip the orchestration block from skill-flats.md

**Files:**
- Modify: `/home/ara/dev/london-flat-hunt/london-property-hunt-public/skill-flats.md`

**Why:** Now that orchestration lives in `run-hunt.sh`, the `§ Subagent-driven scraping (orchestration)` block in the skill is misleading — it told the agent to dispatch subagents, but the agent is no longer the orchestrator. Replace it with a one-paragraph note pointing to the orchestrator.

The per-area accounting block (just below the orchestration block) stays — each scrape subagent still produces it, just one phase-tier at a time.

- [ ] **Step 1: Find the existing orchestration block bounds**

Run:
```bash
grep -n '^### Subagent-driven scraping\|^### Playwright availability probe' /home/ara/dev/london-flat-hunt/london-property-hunt-public/skill-flats.md
```

Expected: two line numbers, the orchestration heading and the next heading. The block to replace is everything between (exclusive of the next heading line).

- [ ] **Step 2: Replace the orchestration block with a one-paragraph note**

Use the Edit tool to replace the entire `### Subagent-driven scraping (orchestration)` section (from the heading down to the line just before `### Playwright availability probe`) with:

```markdown
### Subagent-driven scraping (orchestrated externally)

Scraping is dispatched by `~/hunt/run-hunt.sh` (the cron wrapper), which fires 8 sequential `claude -p` invocations — one per portal × tier (rm-primary, zoopla-primary, sr-primary, or-primary, rm-secondary, zoopla-secondary, sr-secondary, or-secondary). Each invocation receives a narrow prompt with exactly the URLs for its assigned portal-tier, plus paths to the shared run-state directory. The skill below (§ Portal search URLs, § Scraping loop, § VISUAL EXTRACTION, etc.) describes the work each scrape subagent does for its assigned URLs; the orchestration of WHICH subagent runs WHEN is no longer the agent's responsibility.

When invoked from a single-skill context (legacy or remote-WebFetch mode), this section is moot — there's no Playwright in remote mode, and the local single-skill invocation has been retired in favour of the Bash orchestrator.
```

The exact `Edit` call — first read enough context to make the `old_string` unique:

```bash
grep -n "Subagent-driven scraping (orchestration)" /home/ara/dev/london-flat-hunt/london-property-hunt-public/skill-flats.md
sed -n '210,275p' /home/ara/dev/london-flat-hunt/london-property-hunt-public/skill-flats.md > /tmp/orchestration-block.txt
wc -l /tmp/orchestration-block.txt
head -3 /tmp/orchestration-block.txt
tail -3 /tmp/orchestration-block.txt
```

Then perform the Edit using the printed text as `old_string` and the new paragraph above as `new_string`.

- [ ] **Step 3: Verify the edit landed and the surrounding sections still flow**

```bash
sed -n '/^### Per-area accounting/,/^### Portal search URLs (primary areas)/p' /home/ara/dev/london-flat-hunt/london-property-hunt-public/skill-flats.md | head -25
```

Expected: the per-area accounting section, then the new "orchestrated externally" paragraph, then the Playwright probe section, then the primary-area URL section. No dangling references to the old `Agent` tool dispatch.

- [ ] **Step 4: Run a final grep for any leftover orchestration references**

```bash
grep -n -E 'Agent tool|dispatches?.*subagent|portal.tier subagent|Subagent prompt contract|self-budget' /home/ara/dev/london-flat-hunt/london-property-hunt-public/skill-flats.md
```

Expected: zero matches. Any residual mention of the old internal-dispatch architecture should be removed or rephrased to point at the external orchestrator.

- [ ] **Step 5: Commit the skill change**

```bash
cd /home/ara/dev/london-flat-hunt/london-property-hunt-public
git add skill-flats.md
git commit -m "$(cat <<'EOF'
skill(flats): point scraping orchestration to external Bash dispatcher

The ~/hunt/run-hunt.sh wrapper now dispatches the 8 portal-tier
scrape subagents directly via claude -p invocations. The skill no
longer instructs the agent to dispatch — it describes the work each
subagent does for its assigned URLs. This removes the recurring
self-attestation problem where a single agent reading the
orchestration block decided to inline (or skip) the subagent
dispatches.

Spec: docs/superpowers/specs/2026-04-27-bash-orchestrator-design.md
Plan: docs/superpowers/plans/2026-04-27-bash-orchestrator.md

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 12: End-to-end smoke run + commit the plan

**Files:**
- (no file edits in this task — verification only)
- Verify: `/home/ara/dev/london-flat-hunt/london-property-hunt-public/docs/superpowers/plans/2026-04-27-bash-orchestrator.md` is committed alongside the spec.

- [ ] **Step 1: Final pre-run sanity check**

```bash
bash -n /home/ara/hunt/run-hunt.sh && echo "bash syntax OK"
jq . /home/ara/hunt/areas.json > /dev/null && echo "areas.json valid"
ls /home/ara/hunt/prompts/ | wc -l
```

Expected: `bash syntax OK`, `areas.json valid`, `4` (the four prompt files).

- [ ] **Step 2: Run end-to-end and capture timing**

```bash
time /home/ara/hunt/run-hunt.sh
LATEST=$(readlink -f /home/ara/hunt/run-state/latest)
echo "Run dir: $LATEST"
ls "$LATEST"
```

Expected:
- Total wall-clock between 30 and 90 minutes
- The run dir contains all 12 expected files: `init.json`, `alerts.json`, 8× `scrape-*.json`, `digest.json`, `known_urls.txt`, `timings.json`, plus the `log/` subdir

- [ ] **Step 3: Verify per-area accounting completeness**

```bash
LATEST=$(readlink -f /home/ara/hunt/run-state/latest)
echo "Expected total areas across all 8 scrape phases: 48"
jq -s 'map(.areas) | flatten | length' "$LATEST"/scrape-*.json
echo "Phases that failed validation:"
ls "$LATEST"/scrape-*.json.invalid 2>/dev/null || echo "  (none)"
echo "Phases by exit code:"
for f in "$LATEST"/scrape-*.json; do jq -r --arg f "$(basename "$f")" '"\($f): exit=\(.exit_code)"' "$f"; done
```

Expected:
- Total area count is exactly 48 (sum of `length(areas[])` across all 8 scrape phases)
- No `.invalid` sidecars
- All 8 scrape phases have `exit=0` (or, if any have non-zero, the digest's `failed_phases` lists them)

- [ ] **Step 4: Verify the Gmail draft and outreach files**

```bash
LATEST=$(readlink -f /home/ara/hunt/run-state/latest)
jq '.draft_id, .subject, .tier_counts, .outreach_files, .meta_stamped' "$LATEST/digest.json"
ls -la /home/ara/hunt/outreach/ | head -10
```

Expected:
- `draft_id` is a non-null string starting with `r`
- `subject` matches the format from § EMAIL
- `meta_stamped` is `true`
- One outreach file per HIGH listing in today's run (often zero on a typical day)

- [ ] **Step 5: Open the draft and verify the per-area accounting block is in the body**

Open `https://mail.google.com/mail/u/0/#drafts` in a browser. Find the draft from this run by subject. Check the body for:
- A per-area accounting block (one line per portal/area, all 48 expected entries)
- Aggregated tier counts
- Failed phases section (or "(none)" if all clean)

If anything is missing, update the DIGEST prompt to be more specific and re-run that step only:
```bash
LATEST=$(readlink -f /home/ara/hunt/run-state/latest)
RUN_DIR="$LATEST" /home/ara/hunt/run-hunt.sh   # WARN: this is a full re-run; for digest-only iteration, just run the digest section manually
```

- [ ] **Step 6: Commit the plan to the public repo**

```bash
cd /home/ara/dev/london-flat-hunt/london-property-hunt-public
git add docs/superpowers/plans/2026-04-27-bash-orchestrator.md
git commit -m "$(cat <<'EOF'
plan: bash-orchestrated subagent dispatch implementation

12-task implementation plan for the design at
docs/superpowers/specs/2026-04-27-bash-orchestrator-design.md.
Tasks cover areas.json + 4 prompt templates + run-hunt.sh
rewrite (run-state mgmt, run_step helper, INIT, ALERTS,
8 scrape steps, completeness validator, cold-start retry,
DIGEST), the skill-flats.md orchestration-block strip, and
end-to-end smoke verification.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Spec coverage check (self-review)

Walking through the spec section-by-section:

| Spec section | Implemented in |
|---|---|
| § 3 Pipeline (11 invocations in order) | Tasks 5, 6, 7, 8 wire up INIT, ALERTS, 8 SCRAPE, DIGEST in order |
| § 3 State sharing (file-based, run-state dir) | Task 3 sets up `run-state/<run-id>/`, symlink, log dir, pruning |
| § 3 `<phase>.json` schema | Tasks 5/6/7/8 each specify the schema in the prompt template |
| § 3 Subagent prompt template | Task 2 writes init.txt/alerts.txt/scrape.txt/digest.txt; Task 7's `run_scrape` renders scrape.txt |
| § 3 Per-step timeout (25 min) | Task 4 `run_step` uses `timeout --kill-after=30s 25m` |
| § 3 Failure handling (continue on per-step failure) | Task 4 `run_step` writes placeholder JSON and returns rc; orchestrator never aborts |
| § 3 Observability (cron.log + per-step log + json) | Task 4 `run_step` tees to cron.log; per-step log lives in `$RUN_DIR/log/`; JSON in `$RUN_DIR/` |
| § 3 File layout (run-state structure) | Task 3 creates the layout |
| § 4 `run-hunt.sh` rewrite | Tasks 3, 4, 5, 6, 7, 8 — incrementally |
| § 4 `~/hunt/prompts/` | Task 2 |
| § 4 `~/hunt/areas.json` | Task 1 |
| § 4 Skill changes (strip orchestration block) | Task 11 |
| § 5 Data flow | Tasks 5-8's prompt templates encode the data flow |
| § 6 Error handling (per failure) | Task 4 (per-step exit code), Task 5 (init.failed flag), Tasks 5-8 (per-prompt INIT-failure check), Task 9 (validation), Task 10 (cold-start retry) |
| § 6 Bash-side completeness validation | Task 9 |
| § 7 Manual smoke | Task 12 |
| § 9 Open questions / future work | Not implemented (deferred by design) |

All addressable spec sections have a concrete task. The deferred items (§ 9 future work) are deliberately not in this plan.

## Placeholder scan

- No "TBD", "TODO", "implement later", "fill in details" in any task.
- Every step has either runnable code or an explicit verify command.
- No "similar to Task N" — code is repeated where needed (e.g., the env vars in INIT/ALERTS dispatch blocks).
- All file paths are absolute.

## Type / signature consistency

- `run_step` signature: `run_step <phase-name> <prompt-path>` — used identically in Tasks 5, 6, 8 and indirectly by `run_scrape` in Task 7.
- `validate_scrape_phase` signature: `validate_scrape_phase <key>` — used in Task 9's loop.
- `should_retry_step` signature: `should_retry_step <phase-name>` — used inside `run_step` in Task 10.
- `run_scrape` signature: `run_scrape <key>` — used in Task 7's loop.
- Phase names are consistent across prompts, JSON `phase` field, and `<phase>.json` filenames: `init`, `alerts`, `scrape-rm-primary`, `scrape-zoopla-primary`, etc.
- Outcome enum is consistent: `OK | EARLY-EXIT | BLOCKED | TIMEOUT | ZERO-RESULTS | SKIPPED-INIT-FAILED` (matches spec § Per-area accounting + the SKIPPED-INIT-FAILED added per Codex review).
- Variable names in prompt templates match `envsubst` arguments in run-hunt.sh: `$RUN_STATE_DIR`, `$FLATS_DATA_SOURCE_ID`, `$META_DATA_SOURCE_ID`, `$PHASE_NAME`, `$PORTAL`, `$TIER`, `$N`, `$AREAS_BLOCK`, `$OUTREACH_DIR`, `$RUN_MODE`, `$FLAT_EMAIL_TO`.

No inconsistencies found.
