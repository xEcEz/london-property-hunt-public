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
YOUR_PROFESSION=Software Engineer
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
YOUR_EMAIL=you@gmail.com
GMAIL_ACCOUNT_INDEX=0
```

`GMAIL_ACCOUNT_INDEX`: Gmail MCP uses account index to identify which Gmail account to use.
- `0` = your first/primary Google account signed in
- `1` = second account (useful if your hunt email is different from your main account)

## File paths

```
YOUR_HUNT_DIR=~/my-room-hunt
```

The skill will:
- Read/write `$YOUR_HUNT_DIR/london_room_hunt.xlsx`
- Save outreach files to `$YOUR_HUNT_DIR/outreach/`

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

### Tracker (Google Sheet)

```
FLAT_TRACKER_SHEET_ID=1aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789AbCdEf
FLAT_TRACKER_FLATS_TAB=Flats
FLAT_TRACKER_META_TAB=Meta
```

Create a Google Sheet titled "London Flat Hunt" in the Drive account authenticated with Drive MCP. Paste its ID (from the URL, between `/d/` and `/edit`) above.

### Outreach

```
FLAT_OUTREACH_STYLE=agent   # agent | landlord
FLAT_OUTREACH_AVAILABILITY=weekday evenings and weekends
```
