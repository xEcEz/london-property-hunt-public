# Floorplan & Photo Visual Extraction — Design Spec

> **For agentic workers:** This spec extends the existing `skill-flats.md` enrichment flow. It does NOT replace any existing functionality — it adds a visual extraction step between structured JSON extraction and hard-filter application.

**Goal:** Extract size, bathtub, floor level, wood floor, light quality, and renovation state from floorplan images and listing photos during scraping enrichment, for Rightmove and Zoopla listings.

**Motivation:** Structured JSON from portals frequently has missing or garbage size data (e.g. `0.01 ha`), never reports bathtub/wood floor, and sometimes omits floor level. Floorplan images and listing photos contain this information visually. The flat hunt has a hard 60 m² size floor — accurate size data prevents false rejections and false acceptances.

---

## 1. Scope

- **Portals:** Rightmove and Zoopla only. SpareRoom and OpenRent are excluded (photos are low quality, floorplans rare, few listings in scope).
- **Trigger:** Every new listing during the scraping loop. Not conditional on missing data — always run when floorplan/photo URLs exist.
- **Fields extracted:** size (m²), floor level, bathtub, wood floor, light quality, renovation state.

## 2. Enrichment Flow (revised)

Current per-listing flow in § SCRAPING step 7:

1. `browser_navigate` to listing URL
2. `browser_evaluate` to extract structured JSON
3. Apply hard filters
4. Score + insert into Notion

New flow:

1. `browser_navigate` to listing URL
2. `browser_evaluate` to extract structured JSON **+ floorplan image URL(s) + listing photo URLs with captions**
3. **Cheap hard filters on structured data:** reject immediately on beds outside 1–2, rent > hard cap, area not in primary ∪ secondary. These filters use only structured JSON and cost nothing. If any fails, skip this listing — no visual extraction.
4. **Size-gate check:** if structured size is present and sensible (see § 5.1) AND it's below the size floor, reject immediately — no visual extraction needed.
5. **Floorplan pass** (if floorplan URL exists): `browser_navigate` to image URL → `browser_take_screenshot` → extract size (m²), floor level, bathtub hint from architectural drawing
6. **Photo pass** (if listing photos exist): screenshot a selection of up to 3 photos → extract bathtub, wood floor, light, renovation state
7. **Merge** visual data into listing data (see § 5 Merge Priority)
8. **Size hard filter (post-visual):** if the merged size (from floorplan or inference) is below the size floor, reject. This catches listings where structured data had no size but the floorplan reveals it's too small.
9. Score + insert into Notion

Steps 5–6 are skipped entirely for SpareRoom/OpenRent listings. For Rightmove/Zoopla, each step is independently skippable if no URLs are found or if navigation fails. Critically, steps 5–6 are also skipped for listings already rejected by the cheap filters in step 3–4 — visual extraction only runs on listings that survive cheap structured-data checks.

## 3. Photo Selection Strategy

### 3.1 Floorplan

- **Rightmove:** Extract `floorplanImages[0].url` from `PAGE_MODEL.propertyData`.
- **Zoopla:** Extract first floorplan URL from `__PRELOADED_STATE__`. The exact JSON path varies by Zoopla's frontend version — the implementer should inspect a live Zoopla listing page's `__PRELOADED_STATE__` or `__NEXT_DATA__` to locate the floorplan image array.
- Take 1 screenshot only (the first floorplan). Most listings have one; if multiple, the first is typically the main floor.

### 3.2 Listing Photos

- **Budget:** Up to 3 photos per listing.
- **Caption-based selection (preferred):** Scan image captions/alt text in the structured JSON for keywords: `bathroom`, `bath`, `shower`, `kitchen`, `living`, `bedroom`, `reception`, `lounge`. Pick the first match for each category, prioritizing bathroom (most valuable for bathtub detection) then living/bedroom (for wood floor).
- **Index-based fallback:** If captions don't match or aren't available, take photos at indices 0, 3, and 6 (spread across the gallery — agents lead with the hero shot, bathroom is usually mid-gallery).
- **Rightmove:** `PAGE_MODEL.propertyData.images[]` — each entry has `url` and `caption`.
- **Zoopla:** `__PRELOADED_STATE__` image array with similar structure.

### 3.3 Screenshot Budget

Hard cap: **1 floorplan + 3 listing photos = 4 screenshots max** per listing. ~1,500 tokens per image = ~6,000 tokens per listing for visual extraction.

## 4. Extraction Targets

### 4.1 From Floorplan Image

| Field | What to look for | Example |
|-------|-----------------|---------|
| **Total size** | Text like "Approx. Gross Internal Floor Area 759 sq. ft / 70.50 sq. m" at bottom of plan. May be sq ft only — convert at 1 sq ft = 0.0929 m². | 70.50 m² |
| **Floor level** | Text like "Fourth Floor", "Ground Floor", "First Floor" printed on or below the plan. | Only set "Top" when the source explicitly says top/penthouse/uppermost or when both current floor and total floors are known and match (e.g. "Fourth Floor" of a 4-story building). Otherwise: Ground/Lower Ground = "Ground", all other numbered floors = "Mid". Add `Needs-verify: floor` when the floor number is high (3+) but top-floor status can't be confirmed. |
| **Bathtub hint** | Rectangular bath shape visible in the floorplan bathroom layout (distinct from shower cubicle shape). | Yes/No/Unknown |

### 4.2 From Listing Photos

| Field | What to look for | Example |
|-------|-----------------|---------|
| **Bathtub** | Rectangular tub visible in bathroom photos. Freestanding, built-in, or shower-over-bath all count as "Yes". | Yes/No/Unknown |
| **Wood floor** | Hardwood, parquet, or engineered wood visible in room photos. Laminate that looks like wood counts as "Yes". Carpet, tile, or vinyl = "No". | Yes/No/Unknown |
| **Light** | Window size, natural light levels in photos. Large windows with visible daylight = "Good". Small/few windows or dark rooms = "Poor". Mixed = "Avg". | Good/Avg/Poor/Unknown |
| **New/Renovated** | Modern fittings, contemporary kitchen, fresh paint, clean lines = "Yes". Dated wallpaper, old fixtures, worn surfaces = "No". | Yes/No/Unknown |

## 5. Merge Priority

Visual data supplements and can override structured JSON. Priority rules:

### 5.1 Size

Highest-priority source wins:

1. **Floorplan-stated size** → `Size Source = floorplan` (most reliable — printed on architectural drawing)
2. **Structured JSON size** (when sensible — positive number in m² or sq ft, between 10 m² and 500 m², not in nonsensical units like hectares) → `Size Source = stated-text`
3. **Inference rules** (existing SIZE RESOLUTION § fallback: `inferred-2bed`, `inferred-1bed-photos`, `unknown`)

### 5.2 Boolean/Enum Fields (bathtub, wood floor, light, new/renovated, floor)

- Visual evidence **overrides** "Unknown" from structured data.
- Visual evidence **never downgrades** a "Yes" to "No" — if JSON says bathtub=Yes but photos don't show one, keep "Yes" (the bathtub might not be photographed).
- If both floorplan and photos provide a signal (e.g. both show bathtub), either "Yes" wins.
- Floor level: floorplan text (e.g. "Fourth Floor") overrides vague JSON (e.g. no floor info). Map to existing enum per § 4.1: "Top" only with explicit evidence (top/penthouse/total-floors match), "Ground" for ground/lower-ground, "Mid" for everything else. High numbered floors without top-floor confirmation get `Needs-verify: floor`.

## 6. Error Handling

- **Image navigation timeout:** 15 seconds per image. If exceeded, skip that image and continue.
- **Screenshot failure:** Skip and continue. Log "Floorplan/photo screenshot failed for {URL}".
- **Unreadable image:** If the screenshot is taken but the visual content can't be interpreted (blurry, too small, not a floorplan), extract nothing from it and continue. Don't retry.
- **No floorplan URL in JSON:** Skip floorplan pass, continue to photo pass.
- **No listing photos in JSON:** Skip photo pass, continue with structured data only.
- **Failures never block insertion.** A listing with zero successful visual extractions still gets inserted using structured data + inference rules, exactly as today.
- **No retry on image failures.** Unlike dedup (where failure risks duplicates), a failed image is just missing data.

## 7. Time Budget Changes

The existing 25-minute scraping hard kill is too tight for visual extraction. Changes:

| Setting | Old | New | Rationale |
|---------|-----|-----|-----------|
| Scraping hard kill | 25 min | 60 min | Background cron, no user waiting |
| Per-image nav timeout | N/A (new) | 15s | Images load fast; 15s is generous |
| Inter-navigation rate limit | 2s | 2s (unchanged) | Anti-block measure, not speed |

Estimated per-listing time with visual extraction: ~15-20 seconds (vs ~5 seconds without). For 30 new listings: ~10 minutes of visual extraction. Total run well within the 60-minute budget.

## 8. Token Budget

| Component | Tokens per listing |
|-----------|-------------------|
| Structured JSON extraction (existing) | ~800 |
| Floorplan screenshot | ~1,500 |
| Listing photos (up to 3) | ~4,500 |
| Extraction prompts + responses | ~200 |
| **Total per listing** | **~7,000** |

For 10–30 new listings per day: **70k–210k additional tokens per run** (~10–25% overhead on a typical 500k–1M token run).

## 9. Skill File Changes

All changes are to `skill-flats.md`. No new files or infrastructure.

### 9.1 § SIZE RESOLUTION

Update priority item 2 from the vague "Stated on floorplan → extract visually" to reference the concrete floorplan screenshot step defined in § SCRAPING step 7.

### 9.2 § SCRAPING step 7 (Enrichment)

Insert the floorplan pass and photo pass between the structured JSON extraction and the hard-filter application. Provide:
- Rightmove-specific JSON paths for floorplan and photo URLs
- Zoopla-specific JSON paths for floorplan and photo URLs
- Caption-based photo selection logic with index-based fallback
- Per-image extraction instructions (what to look for in each image type)
- Merge rules (§ 5 above)
- Error handling (§ 6 above)

### 9.3 § SCRAPING time budget

Update hard kill from 25 min → 60 min.

### 9.4 § SUCCESS CRITERIA

Add a line: "For Rightmove/Zoopla listings with floorplans, `Size Source = floorplan` should appear on ≥50% of new scraped rows after the first week."

## 10. What This Does NOT Change

- Alert ingestion flow (no images available from email alerts)
- SpareRoom / OpenRent enrichment
- Dedup logic
- Scoring weights or tier thresholds
- Notion schema (all fields already exist)
- Email digest format
