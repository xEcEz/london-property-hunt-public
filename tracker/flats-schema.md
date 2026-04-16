# Flats Tracker — Schema (Google Sheet)

The flats variant of the skill reads and writes a Google Sheet (not an xlsx file). This lets the local laptop cron and the remote Anthropic-hosted fallback trigger share state.

## File location

Stored in the Google Drive account authenticated with the Drive MCP. Sheet ID is set in `config.md` as `FLAT_TRACKER_SHEET_ID`.

## Tabs

1. `Flats` — one row per listing
2. `Meta` — run stamps and skill metadata

---

## `Flats` tab

| # | Column | Type | Notes |
|---|---|---|---|
| 1 | URL | text | Dedup key (hyperlink) |
| 2 | Platform | enum | Rightmove / Zoopla / OpenRent / SpareRoom |
| 3 | Found On | date | First-seen date |
| 4 | Title | text | Listing title |
| 5 | Area | text | Must match a value in `FLAT_PRIMARY_AREAS` or `FLAT_SECONDARY_AREAS` |
| 6 | Price | int | £ pcm |
| 7 | Beds | int | 1 or 2 |
| 8 | Size | number | m² (may be empty if size unknown) |
| 9 | Size Source | enum | stated-text / floorplan / inferred-2bed / inferred-1bed-photos / unknown |
| 10 | Furnished | enum | Furnished / Part / Unfurnished / Unknown |
| 11 | Available From | date | Stated availability |
| 12 | Floor | enum | Top / Mid / Ground / Unknown |
| 13 | Bathtub | enum | Yes / No / Unknown |
| 14 | New/Renovated | enum | Yes / No / Unknown |
| 15 | Calm | enum | Yes / No / Unknown |
| 16 | Light | enum | Good / Avg / Poor / Unknown |
| 17 | Wooden Floor | enum | Yes / No / Unknown |
| 18 | Score | number | 0–11, one decimal |
| 19 | Tier | enum | HIGH / MEDIUM / LOW |
| 20 | Status | enum | NEW 🔴 / CONTACTED / VIEWING / REJECTED / GONE |
| 21 | Reason | text | One-line scoring summary |
| 22 | Needs-verify | text | Comma-separated list of unresolved signals |
| 23 | Notes | text | Free-form |

### Row colours (tier-based)

| Tier | Fill hex |
|---|---|
| HIGH | `#E2EFDA` (green) |
| MEDIUM | `#FFFFC7` (yellow) |
| LOW | `#FCE4D6` (red) |

### Deduplication

Skill loads all values from column 1 (`URL`) into a set at the start of each run. New rows with duplicate URLs are silently dropped — every run is idempotent.

---

## `Meta` tab

| Cell | Purpose |
|---|---|
| `A1` | label: `Key` |
| `B1` | label: `Value` |
| `A2` | `last_local_run` |
| `B2` | ISO date string (`YYYY-MM-DD`) of most recent successful local run |
| `A3` | `last_remote_run` |
| `B3` | ISO date string (`YYYY-MM-DD`) of most recent successful remote run |
| `A4` | `schema_version` |
| `B4` | `1` |

The remote fallback trigger compares `B2` against today's date. If equal → no-op. If not → run flats skill in WebFetch mode, then write today's date to `B3`.

---

## Setup

One-time workbook creation is automated via a small Python script (Google Sheets API) or can be done manually:

1. Create a new Google Sheet titled "London Flat Hunt"
2. Rename the default tab to `Flats`, add a second tab named `Meta`
3. Paste column headers into row 1 of `Flats` (see table above)
4. Fill `Meta!A1:B4` per the cell map above (leave `B2` and `B3` empty initially)
5. Freeze row 1 on `Flats`; enable filter on row 1
6. Copy the Sheet ID from the URL into `config.md` as `FLAT_TRACKER_SHEET_ID`

The skill creates missing tabs and headers on first run if the workbook is empty, but starting from the correct structure avoids first-run friction.
