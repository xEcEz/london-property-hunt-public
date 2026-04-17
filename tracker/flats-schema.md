# Flats Tracker — Schema (Notion)

The flats variant uses two Notion databases under a parent page, accessed via the Notion MCP connector. This replaces the Google-Sheet-based tracker the project originally shipped with — the Drive MCP turned out to be read/create-only (no cell writes), and Notion MCP has full CRUD.

## Location

Any Notion workspace the user authenticates with the Notion MCP. The connector can be scoped page-level, so granting access to a single private "London Flat Hunt" page is enough.

## Databases

1. **Flats** — one page per listing
2. **Meta** — run stamps and skill metadata

Both sit under a parent page called `London Flat Hunt`.

---

## `Flats` database

| Property | Type | Notes |
|---|---|---|
| Title | title | Listing title |
| URL | url | Dedup key — query this property before insert |
| Platform | select | Rightmove / Zoopla / OpenRent / SpareRoom |
| Found On | date | First-seen date |
| Area | select | Must match one of the 16 area values (primary + secondary) |
| Price | number | £ pcm |
| Beds | number | 1 or 2 |
| Size | number | m² (may be empty if unknown) |
| Size Source | select | stated-text / floorplan / inferred-2bed / inferred-1bed-photos / unknown |
| Furnished | select | Furnished / Part / Unfurnished / Unknown |
| Available From | date | Stated availability |
| Floor | select | Top / Mid / Ground / Unknown |
| Bathtub | select | Yes / No / Unknown |
| New/Renovated | select | Yes / No / Unknown |
| Calm | select | Yes / No / Unknown |
| Light | select | Good / Avg / Poor / Unknown |
| Wooden Floor | select | Yes / No / Unknown |
| Score | number | 0–11, one decimal |
| Tier | select | HIGH (green) / MEDIUM (yellow) / LOW (red) |
| Status | select | NEW / CONTACTED / VIEWING / REJECTED / GONE |
| Reason | rich_text | One-line scoring summary |
| Needs-verify | rich_text | Comma-separated list of unresolved signals |
| Notes | rich_text | Free-form |

23 properties total. Tier select colours render HIGH/MEDIUM/LOW rows green/yellow/red automatically.

### Deduplication

Before inserting a new listing, query the Flats data source with a filter on `URL == <candidate URL>`. If any row returns, skip the insert.

---

## `Meta` database

| Property | Type | Notes |
|---|---|---|
| Key | title | `last_local_run` / `last_remote_run` / `schema_version` |
| Value | rich_text | ISO date `YYYY-MM-DD` for run stamps, integer for `schema_version` |

Three rows must exist:

| Key | Value |
|---|---|
| last_local_run | *(empty initially; stamped by local cron)* |
| last_remote_run | *(empty initially; stamped by remote fallback)* |
| schema_version | 1 |

The remote fallback trigger queries `last_local_run` before scraping. If its value equals today's date in Europe/Zurich, the run exits silently.

---

## Setup (via Notion MCP)

One-shot creation of the parent page, both databases, and the three Meta rows:

```
# 1. Create parent page (private — no parent passed)
notion-create-pages([{
  properties: {title: "London Flat Hunt"},
  icon: "🏠",
  content: "Automated daily flat hunt tracker — see skill-flats.md"
}])
# → returns parent page id, e.g. 345211c872eb81b4ad13d424e27bea4d

# 2. Create Flats database under the parent page
notion-create-database({
  title: "Flats",
  parent: {page_id: "<parent page id>"},
  schema: `CREATE TABLE (
    "Title" TITLE,
    "URL" URL,
    "Platform" SELECT('Rightmove':blue, 'Zoopla':purple, 'OpenRent':green, 'SpareRoom':orange),
    "Found On" DATE,
    "Area" SELECT('Islington':blue, 'Angel':blue, 'Camden Town':blue, 'Kentish Town':blue, 'De Beauvoir Town':blue, 'Highbury':blue, 'Canonbury':blue, 'Clerkenwell':blue, 'Tufnell Park':gray, 'Holloway':gray, 'Bloomsbury':gray, 'Russell Square':gray, 'Barbican':gray, 'Finsbury Park':gray, 'London Bridge':gray, 'Bermondsey':gray),
    "Price" NUMBER,
    "Beds" NUMBER,
    "Size" NUMBER,
    "Size Source" SELECT('stated-text':green, 'floorplan':green, 'inferred-2bed':yellow, 'inferred-1bed-photos':orange, 'unknown':gray),
    "Furnished" SELECT('Furnished':blue, 'Part':yellow, 'Unfurnished':green, 'Unknown':gray),
    "Available From" DATE,
    "Floor" SELECT('Top':green, 'Mid':yellow, 'Ground':orange, 'Unknown':gray),
    "Bathtub" SELECT('Yes':green, 'No':red, 'Unknown':gray),
    "New/Renovated" SELECT('Yes':green, 'No':red, 'Unknown':gray),
    "Calm" SELECT('Yes':green, 'No':red, 'Unknown':gray),
    "Light" SELECT('Good':green, 'Avg':yellow, 'Poor':red, 'Unknown':gray),
    "Wooden Floor" SELECT('Yes':green, 'No':red, 'Unknown':gray),
    "Score" NUMBER,
    "Tier" SELECT('HIGH':green, 'MEDIUM':yellow, 'LOW':red),
    "Status" SELECT('NEW':red, 'CONTACTED':yellow, 'VIEWING':blue, 'REJECTED':gray, 'GONE':default),
    "Reason" RICH_TEXT,
    "Needs-verify" RICH_TEXT,
    "Notes" RICH_TEXT
  )`
})
# → returns Flats data source id, e.g. c417b1a4-2ebd-4686-85ec-ca1219a8b0d4

# 3. Create Meta database
notion-create-database({
  title: "Meta",
  parent: {page_id: "<parent page id>"},
  schema: `CREATE TABLE (
    "Key" TITLE,
    "Value" RICH_TEXT COMMENT 'ISO date for run stamps, integer for schema_version'
  )`
})
# → returns Meta data source id, e.g. e914a179-d983-4730-b863-97d188843f90

# 4. Seed Meta rows
notion-create-pages({
  parent: {type: "data_source_id", data_source_id: "<meta data source id>"},
  pages: [
    {properties: {Key: "last_local_run", Value: ""}},
    {properties: {Key: "last_remote_run", Value: ""}},
    {properties: {Key: "schema_version", Value: "1"}}
  ]
})

# 5. Paste the three IDs into config.md:
#    FLAT_TRACKER_NOTION_PARENT_PAGE_ID, FLAT_TRACKER_FLATS_DATA_SOURCE_ID, FLAT_TRACKER_META_DATA_SOURCE_ID
```

## Historical note

Earlier versions of this repo used a Google Sheet tracker. That approach didn't survive remote execution: the claude.ai Google Drive MCP connector can read and create files but cannot modify cells in an existing Sheet, so neither local nor remote runs could persist listings reliably. The Notion MCP has full CRUD and cleanly supports both write paths.
