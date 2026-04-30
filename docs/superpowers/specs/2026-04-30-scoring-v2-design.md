# Scoring v2 — design

**Date:** 2026-04-30
**Status:** Design complete, awaiting user review then plan
**Owner:** Raphael (London flat hunt)

---

## 1. Problem

The user applied for their first flat (Pewter Court, N7) on 2026-04-29. After viewing, two issues surfaced that the listing didn't reveal:

1. **Fully furnished interior was dirty** and felt like sleeping in someone else's bed.
2. **Both bedrooms had beds**, leaving no room for the user's desk in the second bedroom.

The landlord refused to remove the furniture. The application is unlikely to land. The user wants the system to learn from this so similar mismatches surface earlier — ideally as a downgrade from HIGH/auto-shortlist before a viewing is booked.

In parallel, three latent issues with the current scoring system need addressing:

- **Furnishing isn't currently a gate** — `Furnished` listings flow into HIGH if other features are strong. The Pewter Court class of failure can repeat.
- **No green-space proximity signal** — the user mentioned "closeness to a big park, and/or canal, for walks/runs" as a meaningful daily-life criterion, but it's not in scoring.
- **Tier system collapses uncertainty into MEDIUM** — listings that score ≥ HIGH-threshold but fail 1–2 of the existing gates (size verified, avail aligned, 1-bed open-kitchen, etc.) currently get capped to MEDIUM, where they sit alongside listings that are genuinely middling. The user noted this dilutes attention: high-potential-but-uncertain candidates need their own visibility.

This spec defines a single coherent v2 of the scoring system that addresses all three.

---

## 2. Goals & non-goals

### Goals
- **Furnished is a gate**: a `Furnished`-only listing cannot auto-HIGH regardless of score.
- **Green-space proximity feeds scoring**: park/canal-adjacent listings get a meaningful score bump.
- **High-potential-but-uncertain listings have their own tier**: a new `WORTH-CHECKING` tier between HIGH and MEDIUM surfaces them for action.
- **Outreach files generated for both HIGH and WORTH-CHECKING** — both tiers warrant a same-day contact.
- All changes must be reversible (config-driven where possible) and observable (Reason text records every modifier and gate decision).

### Non-goals
- **No new schema columns** beyond adding `WORTH-CHECKING` as an option to the existing `Tier` select. The Reason text is the audit trail.
- **No retroactive re-scoring** of existing rows. New rules apply to listings inserted after the rules ship.
- **No automated dirty-detection** — the cleanliness issue at Pewter Court isn't visible from listing photos / descriptions reliably enough to score. We rely on Furnished-as-a-correlated-proxy.
- **No address-level distance API** for green features — we use a small hardcoded postcode list as fallback to text scan. Geographic libraries / external APIs are out of scope.

---

## 3. Architecture

Three coordinated changes, all in `skill-flats.md` and `~/hunt/config.md`.

### 3.1 Furnished gate

**§ SCORING / Tiers** gains a new gate (#5 in the existing list):

> **Not-fully-furnished gate:** `Furnished ∈ {Part, Unfurnished, Unknown}`. Listings explicitly marked `Furnished` cap at MEDIUM regardless of score, with `furnished` flag in the Reason. `Unknown` is permissive (most agents skip the field; we don't penalise that).

Pool impact (from a 127-row sample on 2026-04-30):
- Furnished: 35% of all listings, 66% of known-state pool
- Part: 11% / 22%
- Unfurnished: 7% / 12%
- Unknown: 47% / —

A hard reject of Furnished would drop the effective pool to ~⅓ of known-state plus the Unknown pile. The MEDIUM-cap keeps Furnished listings visible for manual review without auto-promoting them.

### 3.2 Green-space proximity (parks + canal)

**New extraction step 7d3** in the enrichment loop, called after the description-text size pass (7d) and tube proximity pass (7d2). Pure text scan + postcode lookup — no Playwright, no extra navigation.

**Step 7d3 logic:**

1. **Text scan**: combine title + structured JSON description + key features list. Search for green-feature mentions:
   - `<N>\s*min(s|ute(s)?)?\s*walk\s*to\s*<Park>` / `<N>\s*mins?\s*to\s*<Park>`
   - `near\s+<Park>` / `overlooks\s+<Park>` / `adjacent\s+to\s+<Park>`
   - `canal[-\s]?side` / `canal\s+view` / `overlooks\s+(the\s+)?(Regent'?s\s+)?Canal`
   - `<N>\s*min(s)?\s*to\s*(the\s+)?canal`

2. **Postcode fallback** (when text scan finds nothing): match the listing's postcode against the `FLAT_GREEN_FEATURES` config list (defined in 3.2.1 below). Hit → mark `green-adjacent[<feature>]`.

3. **Resolve to scoring buckets**:
   - Confirmed walk-distance ≤ 5 min to a named big park: bucket `park-close`
   - Confirmed canal-side / canal-view / walk-distance ≤ 3 min to the canal: bucket `canal-close`
   - Both park-close AND canal-close: bucket `park-and-canal`
   - Postcode fallback only (no explicit walk-distance): bucket `green-adjacent`
   - No match: bucket `green-none`

4. Record the bucket and any matched feature names. Append to Reason text.

**Scoring modifiers (in § SCORING / Modifiers):**
- `park-close`: **+1**
- `canal-close`: **+1**
- `park-and-canal`: **+1.5** (cap, not additive)
- `green-adjacent`: **+0.5**
- `green-none`: **0**

#### 3.2.1 `FLAT_GREEN_FEATURES` config

New config block in `config.example.md` and the user's `~/hunt/config.md`. Default seeded with North London features:

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

Format: `<type>:<name>:<comma-separated postcode prefixes or street names>`. `<type>` is `park` or `canal`. `<name>` is the feature label that appears in the Reason text. The third field is matched against the listing's postcode (longest-prefix wins) or, for the canal, against the listing's street address.

Postcode prefixes are deliberately broad — exact `green-adjacent` boundaries are fuzzy in real life. The text scan handles tighter "5 min walk" cases; the postcode list is the soft fallback.

### 3.3 WORTH-CHECKING tier

**Notion schema change**: add `WORTH-CHECKING` as a new option to the existing `Tier` select property. Color: `purple` (between green=HIGH and yellow=MEDIUM in saturation; visually distinct).

**Tier rule (replaces the current Tiers section in § SCORING):**

The gate set:
1. **Size verified**: `Size Source ∈ {stated-text, floorplan}`
2. **Avail aligned**: `Available From ≤ MOVE_IN_TARGET + 14 days` (or `Available From` unknown)
3. **1-bed verified size ≥ 60 m²** (only counted as an active gate when beds = 1)
4. **1-bed open kitchen** (`open_kitchen = true`; only counted as an active gate when beds = 1)
5. **Not fully furnished**: `Furnished ∈ {Part, Unfurnished, Unknown}`

For 2-beds: 3 active gates (1, 2, 5). For 1-beds: 5 active gates (1, 2, 3, 4, 5).

Tier mapping (after gates evaluated, after all modifiers applied to score):

```
IF score ≥ HIGH_THRESHOLD AND zero gates failed:
  → HIGH

ELIF score ≥ HIGH_THRESHOLD AND 1–2 gates failed:
  → WORTH-CHECKING (Reason includes `unlock-by: <comma-separated gate names>`)

ELIF score ≥ HIGH_THRESHOLD AND 3+ gates failed:
  → MEDIUM (too uncertain to highlight)

ELIF MEDIUM_THRESHOLD ≤ score < HIGH_THRESHOLD:
  → MEDIUM

ELSE (score < MEDIUM_THRESHOLD):
  → LOW
```

The "1–2 vs 3+" threshold is intentionally absolute (not relative to active gate count) because:
- 2-beds only have 3 active gates total — failing 2 of 3 is meaningful uncertainty
- 1-beds have 5 active gates — failing 2 of 5 is similar relative uncertainty
- A single threshold is easier to reason about than a per-bed-count threshold
- If the absolute threshold turns out to be wrong, we tune it from observation

**Outreach generation**: HIGH **and** WORTH-CHECKING both produce `~/hunt/outreach/<date>_<slug>.txt` files in the DIGEST step. The user wants to act on WORTH-CHECKING listings too — they're "verify and decide" candidates, not "just maybe".

**Digest layout**: the email digest renders three named sections in this order: HIGH, WORTH-CHECKING, MEDIUM, then a collapsed LOW summary. WORTH-CHECKING entries display the `unlock-by` hint inline so the user can see at a glance what would need verifying.

---

## 4. Schema impact

| Component | Change | Where |
|---|---|---|
| Notion `Tier` select | Add `WORTH-CHECKING` option (purple) | Manual: user adds via Notion UI |
| Notion other columns | None | — |
| `~/hunt/config.md` | Add `FLAT_GREEN_FEATURES` block | New section |
| `config.example.md` | Add `FLAT_GREEN_FEATURES` example | Public repo |
| `skill-flats.md` § SCORING | Update Modifiers + Tier rule + add green-space scoring | Public repo |
| `skill-flats.md` § Scraping enrichment | Add step 7d3 between 7d2 (tube) and 7e (size-gate) | Public repo |
| `skill-flats.md` new § Green-space proximity extraction | Mirror of § Tube proximity extraction | Public repo |
| `skill-flats.md` § OUTREACH | Update to mention WORTH-CHECKING also produces .txt | Public repo |
| Prompt template `digest.txt` (`~/hunt/prompts/`) | Update digest body template to render WORTH-CHECKING section | Local |
| Prompt template `scrape.txt` (`~/hunt/prompts/`) | Reference §3 of skill (now extends to step 7d3 + new tiers) | Local |

The Notion column add is the only manual user step. Everything else is editable text.

---

## 5. Scoring modifier reference (after this change)

| Modifier | Weight | Source |
|---|---|---|
| Top floor | +3 (config) | Existing |
| New build / refurb ≤5y | +3 (config) | Existing |
| Calm street | +2 (config) | Existing |
| Bathtub | +1 (config) | Existing |
| Light/windows | +1 (config) | Existing |
| Wooden floor | +1 (config) | Existing |
| Area in primary | +1 | Existing |
| Area in secondary | −1 | Existing |
| Unfurnished/part-furnished | +0.5 | Existing |
| Rent stretch (above sweet, below hard cap) | −1 | Existing |
| Carpet detected | −1 | Existing |
| Open-plan kitchen | +2 | Existing (bumped from +1 yesterday) |
| Noise hotspot match | −2 + force Calm=No | Existing |
| Avail-aligned (±14 days of MOVE_IN_TARGET) | +1 | Existing |
| Avail-early (15–30 days before) | 0 | Existing |
| Avail-very-early (>30 days before) | −0.5 | Existing |
| Avail-late (>14 days after) | −2 | Existing |
| Tube ≤5 min | +1 | Existing |
| Tube 6–10 min | 0 | Existing |
| Tube 11–15 min | −1 | Existing |
| Tube >15 min | −2 | Existing |
| **Park-close (≤5min walk)** | **+1** | **NEW (3.2)** |
| **Canal-close (≤3min walk / canal-side)** | **+1** | **NEW (3.2)** |
| **Park AND canal close** | **+1.5 (cap)** | **NEW (3.2)** |
| **Green-adjacent (postcode fallback)** | **+0.5** | **NEW (3.2)** |

Score is then clamped to 0–11 (existing rule).

---

## 6. Gate reference (after this change)

| Gate | Active for | Pass condition | Failure → |
|---|---|---|---|
| 1. Size verified | All | `Size Source ∈ {stated-text, floorplan}` | Reason: `unverified-size` |
| 2. Avail aligned | All | `Available From ≤ MOVE_IN_TARGET + 14d` OR unknown | Reason: `avail-too-late` |
| 3. 1-bed size ≥60m² | beds=1 only | Verified size ≥ 60 | Reason: `1bed-borderline-size` |
| 4. 1-bed open kitchen | beds=1 only | `open_kitchen=true` | Reason: `1bed-no-open-kitchen` |
| 5. **Not Furnished** | All | `Furnished ∈ {Part, Unfurnished, Unknown}` | Reason: `furnished` |

---

## 7. Edge cases & open questions

- **WORTH-CHECKING for 1-beds vs 2-beds, asymmetric gate counts.** A 1-bed has 5 gates; failing 2 of 5 is "40% uncertain". A 2-bed has 3 gates; failing 2 of 3 is "67% uncertain" — much higher. Keeping the threshold absolute (≤2 failed) means a 1-bed and 2-bed reach WORTH-CHECKING at different relative uncertainty levels. **Decision: accept this asymmetry**; it's simpler and matches how the user thinks about the failure modes (they care about specific gates, not gate count).
- **Furnished + WORTH-CHECKING combinatorics.** A Furnished 2-bed scoring 9 with all other gates passing would WORTH-CHECKING (failed: gate 5 only). A Furnished 1-bed scoring 9 with size verified, avail aligned, but missing open_kitchen would have 2 failures (4 + 5) → also WORTH-CHECKING. Both correct outcomes.
- **`Unknown` Furnished is permissive.** Half the pool is Unknown. We don't penalise Unknown, but we also can't auto-HIGH a listing where we don't know the furnishing state. **Decision: Unknown passes the gate** (no penalty for incomplete data; the user can still see the listing in HIGH if all known signals are good).
- **Green-feature postcode fallback false positives.** A listing with postcode `NW1 5` near Regent's Park might actually be 15 min walk from the park edge. The +0.5 fallback bucket is intentionally low-weight to limit damage from over-broad postcode matches.
- **Canal addresses by street name.** A listing on "Vincent Terrace, N1 8…" matches both the postcode and the street — we don't double-count; pick the more specific (street) match for the Reason text but the score modifier is the same `+0.5`.
- **WORTH-CHECKING outreach files written to disk** even if user never sends them. Same pattern as today's HIGH-only outreach. User cleans up unused .txt files manually.

---

## 8. Migration plan summary (deferred to writing-plans)

Outline only — full plan in the next document:

1. User manually adds `WORTH-CHECKING` option to Notion `Tier` select (one-time UI step).
2. Update `skill-flats.md` § SCORING (Modifiers + Tiers + Gate reference).
3. Add new § Green-space proximity extraction section to `skill-flats.md`.
4. Add step 7d3 to the enrichment loop in `skill-flats.md`.
5. Update § OUTREACH to include WORTH-CHECKING.
6. Add `FLAT_GREEN_FEATURES` to `config.example.md` (public repo) and `~/hunt/config.md` (local).
7. Update `~/hunt/prompts/digest.txt` to render the new tier in the email body.
8. Update `~/hunt/prompts/scrape.txt` (no change to template, but verify it still references skill-flats.md sections that exist).
9. Smoke test on tomorrow's cron.

---

## 9. Decision log

- **2026-04-30**: chose Furnished policy `c` (hard cap at MEDIUM) over `a` (hard reject — too aggressive, gut pool to ⅓), `b` (−2 score penalty — borderline Furnished could still reach HIGH), `d` (only target 2-bed-with-second-bed — too narrow, dirtiness/extra-furniture is broader signal).
- **2026-04-30**: chose green-space detection `c` (text scan + postcode fallback) over `a` (text scan only — would miss listings where agent doesn't mention the feature; ~50% of listings have generic descriptions), `b` (postcode-only — coarse, no walk-distance differentiation).
- **2026-04-30**: chose tier redesign `b` (add WORTH-CHECKING tier) over `a` (sort MEDIUM by distance-to-HIGH — same info, less visible), `c` (5 tiers — adds bands without addressing confidence concern), `d` (2D Tier × Confidence — schema change for marginal gain).
- **2026-04-30**: WORTH-CHECKING also generates outreach .txt files. The user wants to act on uncertainty-flagged listings, not just see them.
- **2026-04-30**: Open-kitchen weight stays at +2 (bumped yesterday). Tube proximity weights stay (+1/0/−1/−2). The Furnished/green-space changes don't conflict with yesterday's gate work.
- **2026-04-30**: WORTH-CHECKING threshold is absolute (≤2 gates failed) not relative to active gate count. Simpler reasoning; tune from observation if it produces the wrong distribution.
