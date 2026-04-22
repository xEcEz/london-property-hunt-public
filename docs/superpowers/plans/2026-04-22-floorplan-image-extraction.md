# Floorplan & Photo Visual Extraction — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add visual extraction of floorplan images and listing photos to the scraping enrichment flow in `skill-flats.md`, for Rightmove and Zoopla listings.

**Architecture:** Single-file edit to `skill-flats.md`. The enrichment step (§ SCRAPING step 7) gets restructured: cheap hard filters run first on structured data, then floorplan/photo screenshots are taken and analyzed for size/bathtub/floor/wood floor/light/renovation, then a post-visual size filter runs, then scoring and Notion insert. No new files or infrastructure.

**Tech Stack:** Playwright MCP (`browser_navigate`, `browser_take_screenshot`, `browser_evaluate`), Notion MCP, skill markdown.

**Spec:** `docs/superpowers/specs/2026-04-22-floorplan-image-extraction-design.md`

---

## File Map

- **Modify:** `skill-flats.md`
  - Lines 43–52: § SIZE RESOLUTION — update item 2
  - Lines 198–206: § SCRAPING step 7 (Enrichment) — restructure into 9-step flow with visual extraction
  - Line 222: § SCRAPING rate limiting — bump hard kill 25→60 min
  - Lines 312–326: § SUCCESS CRITERIA — add visual extraction criterion

No files created. No files deleted.

---

### Task 1: Restructure enrichment step 7 — cheap filters before visual extraction

**Files:**
- Modify: `skill-flats.md:198-206`

This task replaces the current step 7 monolith with the new 9-step flow from spec § 2. It inserts the cheap-filter gates (steps 3–4) and the post-visual size filter (step 8), and adjusts the existing structured-JSON extraction (step 2) to also extract floorplan and photo URLs. The actual floorplan/photo pass content (steps 5–6) is added as stubs here and filled in by Task 2.

- [ ] **Step 1: Replace the enrichment step 7 block**

In `skill-flats.md`, replace lines 198–206 (from `7. **Enrichment:**` through the `listing page unreachable` fallback) with:

```markdown
7. **Enrichment (Rightmove / Zoopla):**
   a. Call `mcp__playwright__browser_navigate` with the listing URL.
   b. Call `mcp__playwright__browser_evaluate` with a portal-specific snippet that pulls title / price / beds / size / floor / furnished / available-from / description text / amenity hints, **plus floorplan image URL(s) and listing photo URLs with captions**. Examples:
      - Rightmove: `() => { const m = window.PAGE_MODEL; if (!m) return null; const p = m.propertyData || {}; return { ...p, _floorplanUrl: (p.floorplans || [])[0]?.url || null, _photos: (p.images || []).map(i => ({ url: i.url, caption: i.caption })) }; }` then parse the result.
      - Zoopla: `() => { const d = window.__PRELOADED_STATE__ || JSON.parse(document.querySelector('script#__NEXT_DATA__')?.textContent || '{}'); return d; }` then locate the listing object, floorplan URL (path varies — look for a `floorPlanImages`, `floorPlan`, or `floorplan` key in the listing data), and photo array.
   c. **Cheap hard filters (structured data only):** reject immediately if beds outside 1–2, or rent > [FLAT_BUDGET_HARD_CAP], or area not in primary ∪ secondary. These use only structured JSON. If any fails → skip this listing entirely, no visual extraction.
   d. **Size-gate check:** if the structured JSON provides a sensible size (positive number in m² or sq ft, between 10 m² and 500 m², not in nonsensical units like hectares) AND that size is below [FLAT_SIZE_FLOOR_M2] → reject immediately, no visual extraction.
   e. **Floorplan pass (Rightmove / Zoopla only, if floorplan URL exists):** see § VISUAL EXTRACTION — FLOORPLAN below.
   f. **Photo pass (Rightmove / Zoopla only, if listing photos exist):** see § VISUAL EXTRACTION — PHOTOS below.
   g. **Merge visual data** into the listing record. Priority rules:
      - **Size:** floorplan-stated size wins (set `Size Source=floorplan`). If no floorplan size, use sensible structured JSON size (set `Size Source=stated-text`). Otherwise fall back to § SIZE RESOLUTION inference rules.
      - **Boolean/enum fields** (bathtub, wood floor, light, new/renovated, floor): visual evidence overrides "Unknown" from structured data. Visual evidence never downgrades a "Yes" to "No" (the feature might not be photographed). If both floorplan and photos provide a signal, "Yes" wins.
      - **Floor level:** floorplan text overrides vague JSON. Only set "Top" when the source explicitly says top/penthouse/uppermost, or when both current floor and total floors are known and match. Ground/Lower Ground = "Ground". All other numbered floors = "Mid". Add `Needs-verify: floor` when the floor number is ≥3 but top-floor status can't be confirmed.
   h. **Post-visual size filter:** if the merged size (from floorplan, structured data, or inference) is below [FLAT_SIZE_FLOOR_M2] → reject. Do NOT create a Notion row.
   i. Score per § SCORING. Call `mcp__claude_ai_Notion__notion-create-pages` with parent `{type: "data_source_id", data_source_id: "[FLAT_TRACKER_FLATS_DATA_SOURCE_ID]"}` and properties: Title, URL, Platform, Found On (today's date, Europe/Zurich), Area, Price, Beds, Size, Size Source, Furnished, Available From, Floor, Bathtub, New/Renovated, Calm, Light, Wooden Floor, Score, Tier, Status=`NEW`, Reason, Needs-verify, Source=`scraped-local`. Do NOT set Human Tier or Human Rationale.
   - If listing-page navigate times out or the portal returns an empty/error page for this specific listing, create a minimal Notion row with Title + URL + Platform + Source=`scraped-local` + Needs-verify=`listing page unreachable`, and leave Score + Tier blank.

   **Enrichment (SpareRoom / OpenRent):**
   - No floorplan or photo visual extraction. Use the existing SpareRoom/OpenRent enrichment approach: inspect the rendered page via `mcp__playwright__browser_snapshot` and read visible text for price / beds / size / address / furnished / available-from, plus `<meta property="og:title">`, `<meta name="description">`, and any JSON-LD `<script type="application/ld+json">` block. If fields can't be confidently extracted, insert a minimal row (Title + URL + Platform + Source=`scraped-local` + Needs-verify=`extraction incomplete`) rather than dropping the listing.
   - Apply HARD FILTERS (beds in 1–2, rent ≤ [FLAT_BUDGET_HARD_CAP], area in primary ∪ secondary, stated size ≥ [FLAT_SIZE_FLOOR_M2] or size-inferred per § SIZE RESOLUTION). If any filter fails, skip — do NOT create a Notion row.
   - Score and insert as per step 7i above.
```

- [ ] **Step 2: Verify the edit**

Read `skill-flats.md` and confirm:
- Step 7 now has sub-steps a through i for Rightmove/Zoopla
- SpareRoom/OpenRent has a separate enrichment block with no visual extraction
- Cheap filters (7c, 7d) come before visual extraction (7e, 7f)
- Post-visual size filter (7h) comes after merge (7g)
- References to `§ VISUAL EXTRACTION — FLOORPLAN` and `§ VISUAL EXTRACTION — PHOTOS` exist (these sections are added in Task 2)

- [ ] **Step 3: Commit**

```bash
git add skill-flats.md
git commit -m "skill(flats): restructure enrichment with cheap filters before visual extraction"
```

---

### Task 2: Add floorplan pass and photo pass sections

**Files:**
- Modify: `skill-flats.md` (insert new sections after the enrichment step 7, before § Pagination cutoff)

This task adds the two new sections referenced by step 7e and 7f: `§ VISUAL EXTRACTION — FLOORPLAN` and `§ VISUAL EXTRACTION — PHOTOS`. These contain the detailed instructions for the LLM to screenshot and interpret images.

- [ ] **Step 1: Insert the VISUAL EXTRACTION sections**

After the enrichment step 7 block (after the `listing page unreachable` fallback line) and before `### Pagination cutoff`, insert:

```markdown
### VISUAL EXTRACTION — FLOORPLAN

This section is called from enrichment step 7e. It runs for Rightmove and Zoopla listings only, when the structured JSON contains a floorplan image URL.

1. Extract the floorplan image URL:
   - **Rightmove:** `_floorplanUrl` from the evaluate result (populated from `PAGE_MODEL.propertyData.floorplans[0].url`).
   - **Zoopla:** locate the floorplan URL in the evaluate result. Look for keys named `floorPlanImages`, `floorPlan`, `floorplan`, or `floorPlanImage` in the listing data object. Take the first URL found.
   - If no floorplan URL is found, skip this section — continue to step 7f (photo pass).

2. Call `mcp__playwright__browser_navigate` with the floorplan image URL. Timeout: 15 seconds. If navigation fails or times out, log "Floorplan navigation failed for {listing URL}" and skip to step 7f.

3. Call `mcp__playwright__browser_take_screenshot`. If the screenshot fails, log and skip to step 7f.

4. Examine the screenshot and extract:
   - **Total floor area:** look for text at the bottom or top of the floorplan image, typically "Approx. Gross Internal Floor Area NNN sq. ft / NN.NN sq. m" or just "NNN sq ft" or "NN m²". If the area is in sq ft only, convert: `m² = sq_ft × 0.0929`. Round to 1 decimal place.
   - **Floor level:** look for text like "Fourth Floor", "Ground Floor", "First Floor", "Basement", "Lower Ground Floor" printed on or near the plan. Map to the Floor enum:
     - "Top" — only if the text explicitly says "top floor", "penthouse", or "uppermost", OR if both the current floor number and total building floors are known and they match.
     - "Ground" — if it says "Ground Floor" or "Lower Ground Floor".
     - "Mid" — all other numbered floors.
     - If the floor number is ≥3 but you cannot confirm it's the top floor, set Floor="Mid" and append `floor` to the Needs-verify field.
   - **Bathtub hint:** look for a rectangular bath shape in the bathroom area of the plan (distinct from the smaller square/quadrant shower cubicle shape). Set Bathtub="Yes" if clearly visible, "No" if the bathroom only shows a shower, "Unknown" if unclear.

5. Return the extracted values (size_m2, floor, bathtub) to the merge step (7g).

### VISUAL EXTRACTION — PHOTOS

This section is called from enrichment step 7f. It runs for Rightmove and Zoopla listings only, when the structured JSON contains listing photo URLs.

1. Select up to 3 photos from the listing's photo array:
   - **Caption-based selection (preferred):** scan the `_photos` array (Rightmove) or equivalent Zoopla photo array for entries whose `caption` (case-insensitive) contains any of: `bathroom`, `bath`, `shower room`, `kitchen`, `living`, `bedroom`, `reception`, `lounge`. Select photos in this priority order:
     a. First photo with a bathroom/bath/shower caption (highest value — bathtub detection).
     b. First photo with a living/reception/lounge/bedroom caption (wood floor, light).
     c. First photo with a kitchen caption (renovation state).
   - **Index-based fallback:** if fewer than 3 photos matched by caption, fill remaining slots from indices 0, 3, 6 of the photo array (spread across the gallery). Skip any index that was already selected by caption or that exceeds the array length.
   - Hard cap: 3 photos total. If the listing has fewer than 3 photos, take all available.

2. For each selected photo:
   a. Call `mcp__playwright__browser_navigate` with the photo URL. Timeout: 15 seconds. If navigation fails or times out, skip this photo — continue to the next.
   b. Call `mcp__playwright__browser_take_screenshot`. If the screenshot fails, skip this photo.
   c. Examine the screenshot and extract the relevant fields based on what the photo shows:
      - **Bathroom photo:** Bathtub — "Yes" if a rectangular tub is visible (freestanding, built-in, or shower-over-bath), "No" if only a shower stall/cubicle is visible, "Unknown" if unclear. Also note Light and New/Renovated if assessable.
      - **Room photo (living/bedroom/reception):** Wood floor — "Yes" if hardwood, parquet, or engineered-wood flooring is visible (laminate that looks like wood counts as "Yes"), "No" if carpet, tile, or vinyl is visible, "Unknown" if unclear. Light — "Good" if large windows with visible daylight, "Poor" if small/few windows or dark, "Avg" if mixed, "Unknown" if photo doesn't show windows. New/Renovated — "Yes" if modern fittings, contemporary style, fresh paint, "No" if dated fixtures/wallpaper/worn surfaces, "Unknown" if unclear.
      - **Kitchen photo:** New/Renovated — "Yes" if modern kitchen with contemporary units/appliances, "No" if dated, "Unknown" if unclear.

3. Consolidate across all photos: for each field (bathtub, wood floor, light, new/renovated), if any photo yielded a definitive "Yes" or "No", use that. If multiple photos disagree, prefer "Yes" over "No" (the negative photo might show a different room). If all photos yielded "Unknown" for a field, keep "Unknown".

4. Return the consolidated values (bathtub, wood_floor, light, new_renovated) to the merge step (7g).
```

- [ ] **Step 2: Verify the edit**

Read `skill-flats.md` and confirm:
- `### VISUAL EXTRACTION — FLOORPLAN` section exists after step 7, before `### Pagination cutoff`
- `### VISUAL EXTRACTION — PHOTOS` section exists after the floorplan section
- Floorplan section references `mcp__playwright__browser_navigate` and `mcp__playwright__browser_take_screenshot` (correct tool names)
- Photo section has caption-based selection with bathroom priority, index fallback, 3-photo hard cap
- Both sections have 15-second timeouts and skip-on-failure behavior
- Floor mapping uses the strict "Top requires evidence" rule

- [ ] **Step 3: Commit**

```bash
git add skill-flats.md
git commit -m "skill(flats): add floorplan and photo visual extraction sections"
```

---

### Task 3: Update SIZE RESOLUTION, time budget, and SUCCESS CRITERIA

**Files:**
- Modify: `skill-flats.md:43-52` (SIZE RESOLUTION)
- Modify: `skill-flats.md:222` (rate limiting — hard kill)
- Modify: `skill-flats.md:312-326` (SUCCESS CRITERIA)

- [ ] **Step 1: Update § SIZE RESOLUTION item 2**

In `skill-flats.md`, replace lines 45–46:

```
1. Stated in listing text → use.
2. Stated on floorplan → extract visually.
```

with:

```
1. Stated in listing text (structured JSON, when sensible — positive number, 10–500 m², not in nonsensical units) → use, set `Size Source=stated-text`.
2. Stated on floorplan image → extracted via § VISUAL EXTRACTION — FLOORPLAN screenshot during scraping enrichment step 7e. Set `Size Source=floorplan`. This is the most reliable source — floorplan-stated size overrides structured JSON when both are available.
```

- [ ] **Step 2: Update scraping hard kill**

In `skill-flats.md`, find the line:

```
- Typical scrape runtime: 6–12 minutes. **Hard kill at 25 minutes** of scraping wall-clock
```

Replace with:

```
- Typical scrape runtime: 15–30 minutes (with visual extraction). **Hard kill at 60 minutes** of scraping wall-clock
```

- [ ] **Step 3: Update § SUCCESS CRITERIA**

In `skill-flats.md`, after the line:

```
- HIGH tier caps respected for size-unknown 1-beds.
```

Insert:

```
- For Rightmove/Zoopla scraped listings with floorplan images, `Size Source = floorplan` should appear on the majority of new rows. If consistently low, investigate whether floorplan URL extraction or screenshot reading is failing silently.
- Visual extraction (floorplan + photos) attempted for every new Rightmove/Zoopla listing that passes cheap hard filters. Digest records count of successful floorplan reads vs. total attempts.
```

- [ ] **Step 4: Verify all three edits**

Read `skill-flats.md` and confirm:
- § SIZE RESOLUTION item 2 references `§ VISUAL EXTRACTION — FLOORPLAN` and step 7e
- Hard kill is 60 minutes
- § SUCCESS CRITERIA has two new visual-extraction lines

- [ ] **Step 5: Commit**

```bash
git add skill-flats.md
git commit -m "skill(flats): update SIZE RESOLUTION, time budget (60 min), and SUCCESS CRITERIA for visual extraction"
```

---

## Self-Review

**Spec coverage check:**

| Spec Section | Plan Task |
|-------------|-----------|
| § 1 Scope (RM+Zoopla only) | Task 1 (SpareRoom/OpenRent separated), Task 2 (sections scoped to RM+Zoopla) |
| § 2 Enrichment Flow | Task 1 (full 9-step flow) |
| § 3 Photo Selection | Task 2 (caption-based + index fallback) |
| § 4 Extraction Targets | Task 2 (floorplan: size/floor/bathtub; photos: bathtub/wood/light/reno) |
| § 5 Merge Priority | Task 1 step 7g (merge rules inline) |
| § 6 Error Handling | Task 2 (15s timeout, skip-on-failure, no retry) |
| § 7 Time Budget | Task 3 (25→60 min) |
| § 8 Token Budget | Informational — no skill-file change needed |
| § 9.1 SIZE RESOLUTION | Task 3 |
| § 9.2 SCRAPING step 7 | Task 1 + Task 2 |
| § 9.3 Time budget | Task 3 |
| § 9.4 SUCCESS CRITERIA | Task 3 |
| § 10 No-change list | Verified — plan touches only skill-flats.md |

**Placeholder scan:** No TBD/TODO. All code blocks are complete. All tool names use the correct `mcp__playwright__browser_*` prefix.

**Consistency check:** `_floorplanUrl` and `_photos` field names match between Task 1 (evaluate snippet) and Task 2 (extraction sections). Floor mapping rules match between Task 2 (floorplan section) and Task 1 (merge step 7g). Timeout values (15s) match between Task 2 and spec § 6.
