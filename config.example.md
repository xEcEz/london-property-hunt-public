# Property Hunt — Personal Config

> Copy this file to `config.md` and fill in your details.
> `config.md` is gitignored — your personal info stays local.

---

## Mode

```
HUNT_MODE=rooms
```

`HUNT_MODE` selects which skill variant runs:
- `rooms` — shared-room hunt (uses `skill.md`, xlsx tracker)
- `flats` — whole-unit 1–2 bed hunt (uses `skill-flats.md`, Google Sheet tracker)

All fields below that begin with `FLAT_*` or are under a `## Flats mode` section are ignored in `rooms` mode, and vice versa.

---

## About you

```
YOUR_NAME=Alex
YOUR_AGE=29
YOUR_NATIONALITY=British
YOUR_PROFESSION=Software Engineer
YOUR_EMPLOYER_TYPE=a fintech startup
YOUR_PROFILE_DESCRIPTION=easy-going, clean, tidy, cooks, gym, hybrid WFH 2 days/week, no parties, professional
YOUR_PROFILE_SUMMARY=Clean, tidy, reliable — hybrid WFH, permanent contract
```

## Commute

```
YOUR_WORK_POSTCODE=EC2A 1NT
PARTNER_AREA=Hackney Central
TUBE_LINE=Victoria Line
```

## Target areas

List your areas in priority order, comma-separated.
The skill will use these to set priority tiers and generate the SpareRoom/OpenRent search URLs.

```
PRIMARY_AREAS=Hackney, Shoreditch, Bethnal Green, Clapton
SECONDARY_AREAS=Dalston, Stoke Newington, Islington
```

Tip: pick areas on the same tube/rail line as your work and social anchor points.
The shorter the line change, the better.

## Budget

```
ROOM_BUDGET=1500
ROOM_BUDGET_NO_BILLS=1700
STUDIO_BUDGET=1900
```

## Dates

```
MOVE_IN_DATE=1 June 2026
MOVE_IN_MONTH=June
```

## Flatmate preferences

```
FLATMATE_MIN_AGE=28
FLATMATE_MAX_AGE=40
```

## Email

```
YOUR_EMAIL=you@gmail.com             # sender — the Gmail account authenticated in the MCP
FLAT_EMAIL_TO=you@gmail.com          # destination — where the hunt digest is delivered (flats variant; can differ from YOUR_EMAIL if you want the digest to land in a different inbox)
GMAIL_ACCOUNT_INDEX=0
```

`GMAIL_ACCOUNT_INDEX`: Gmail MCP uses account index to identify which Gmail account to use.
- `0` = your first/primary Google account signed in
- `1` = second account (useful if your hunt email is different from your main account)

`FLAT_EMAIL_TO` is the flats-variant-specific destination for the daily digest. If you want the hunt email to land in your main inbox while being sent from a dedicated hunt Gmail, set this to your main address. If you want the sender and recipient to be the same, set it equal to `YOUR_EMAIL`.

## File paths

```
YOUR_HUNT_DIR=~/my-room-hunt
```

The skill will:
- **Rooms variant:** read/write `$YOUR_HUNT_DIR/london_room_hunt.xlsx`
- **Flats variant:** read/write the Notion databases referenced by `FLAT_TRACKER_*_DATA_SOURCE_ID` (tracker is not local)
- Save outreach files to `$YOUR_HUNT_DIR/outreach/` (both variants, local mode only)

Make sure the directory exists before running:
```bash
mkdir -p ~/my-room-hunt/outreach
```

---

## SpareRoom search URLs

After filling in your areas above, generate your SpareRoom room search URLs.
Format: `https://www.spareroom.co.uk/flatshare/london/[area_slug]?max_rent=[ROOM_BUDGET]&sort=posted_date&mode=list`

Area slugs use underscores: `elephant_and_castle`, `canada_water`, `bethnal_green`, etc.

```
SPAREROOM_ROOM_URLS=
  https://www.spareroom.co.uk/flatshare/london/hackney?max_rent=1500&sort=posted_date&mode=list
  https://www.spareroom.co.uk/flatshare/london/shoreditch?max_rent=1500&sort=posted_date&mode=list
  https://www.spareroom.co.uk/flatshare/london/bethnal_green?max_rent=1500&sort=posted_date&mode=list
```

SpareRoom studio/flat search URLs:
```
SPAREROOM_STUDIO_URLS=
  https://www.spareroom.co.uk/flats-to-rent/london/hackney?max_rent=1900&sort=posted_date&mode=list
  https://www.spareroom.co.uk/flats-to-rent/london/shoreditch?max_rent=1900&sort=posted_date&mode=list
```

---

## Notes

- You can update this file any time — just re-run the skill and it picks up changes
- If you find a place and want to stop the scheduled runs, delete the schedule: `/schedule list` then `/schedule delete [id]`

---

## Flats mode (only used if `HUNT_MODE=flats`)

### Unit criteria

```
FLAT_BEDROOMS_MIN=1
FLAT_BEDROOMS_MAX=2
FLAT_SIZE_FLOOR_M2=60
FLAT_FURNISHED_PREFERENCE=unfurnished   # unfurnished | furnished | any
```

### Budget

```
FLAT_BUDGET_SWEET_MAX=3200
FLAT_BUDGET_HARD_CAP=3500
FLAT_STRETCH_PENALTY=1
```

### Areas (ordered by priority)

```
FLAT_PRIMARY_AREAS=Islington, Angel, Camden Town, Kentish Town, De Beauvoir Town, Highbury, Canonbury, Clerkenwell
FLAT_SECONDARY_AREAS=Tufnell Park, Holloway, Bloomsbury, Russell Square, Barbican, Finsbury Park, London Bridge, Bermondsey
```

### Noise hotspots (immediate-proximity penalty applied at scoring time)

Listings whose title / address / description matches any of these substrings (case-insensitive) get `Calm=No` forced and a −2 score penalty. Use specific landmark names, not whole areas — area-level proximity is already captured by primary/secondary lists. Default targets are major event venues where match-day noise is significant. Edit per personal tolerance.

```
FLAT_NOISE_HOTSPOTS=Arsenal, Emirates Stadium, Highbury Stadium Square, Drayton Park, Wembley, White Hart Lane, Tottenham Hotspur
```

### Green-space features (proximity scoring — parks and canal)

Listings whose description mentions or whose postcode/street matches one of these features get a scoring bonus and an annotation in the Reason text. Each line follows the format `<type>:<name>:<comma-separated postcode prefixes or street names>`, where `<type>` is `park` or `canal`. The skill's text scan handles explicit "5 min walk to <Park>" cases on top of this list; this list is the postcode/street fallback for listings where the agent didn't bother to mention the feature.

```
FLAT_GREEN_FEATURES=
  park:Regent's Park:NW1 4,NW1 5,NW1 6,NW1 7
  park:Highbury Fields:N5 1,N5 2,N5 3
  park:Hampstead Heath:NW3 1,NW3 2,NW3 5,NW3 7,NW5 1,N6 4
  park:Clissold Park:N16 0,N16 9
  park:Victoria Park:E2 9,E3 5,E9 5,E9 7
  park:Hyde Park:W1J,W2 2,SW1X
  canal:Regent's Canal:N1 7,N1 8,NW1 8,E2 8,E8 4
  canal:Regent's Canal towpath addresses:Vincent Terrace,Noel Rd,Wharf Rd,Baltic Pl,Wenlock Basin,Battlebridge Basin
```

### Scoring weights (overrides — defaults in skill-flats.md)

```
FLAT_WEIGHT_TOP_FLOOR=3
FLAT_WEIGHT_NEW_BUILD=3
FLAT_WEIGHT_CALM=2
FLAT_WEIGHT_BATHTUB=1
FLAT_WEIGHT_LIGHT=1
FLAT_WEIGHT_WOODEN_FLOOR=1
FLAT_TIER_HIGH_THRESHOLD=7
FLAT_TIER_MEDIUM_THRESHOLD=4
```

### Tracker (Notion)

```
FLAT_TRACKER_NOTION_PARENT_PAGE_ID=00000000000000000000000000000000
FLAT_TRACKER_FLATS_DATA_SOURCE_ID=00000000-0000-0000-0000-000000000000
FLAT_TRACKER_META_DATA_SOURCE_ID=00000000-0000-0000-0000-000000000000
```

Create a Notion page titled "London Flat Hunt" (private-scope), then under it create two databases named `Flats` and `Meta` per `tracker/flats-schema.md`. Paste the parent page ID and each database's data source ID above.

A one-shot setup script is available in `tracker/flats-schema.md` that shows the exact DDL / Notion MCP calls to create both databases with the right schemas and the three Meta rows (`last_local_run`, `last_remote_run`, `schema_version=1`).

### Outreach

```
FLAT_OUTREACH_STYLE=agent   # agent | landlord
FLAT_OUTREACH_AVAILABILITY=weekday evenings and weekends
```
