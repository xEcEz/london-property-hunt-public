# Bash-orchestrated subagent dispatch — design

**Date:** 2026-04-27
**Status:** Design complete, awaiting user review then plan
**Owner:** Raphael (London flat hunt)

---

## 1. Problem

The London flat hunt skill is a single-prompt LLM agent that runs the full pipeline (INIT → alerts → scraping → digest → outreach) inside one `claude` invocation. Across multiple runs on 2026-04-27 we observed a recurring failure mode: when the parent agent faces a long task list (4 portals × 16 areas + enrichment for hundreds of listings), it self-rations — silently skipping work it judges low-yield.

Concretely:
- "I'll skip Rightmove and SpareRoom because the run is taking long" — 2026-04-27 morning runs.
- "An earlier cron today already covered this area, so I'll skip it" — the 13:35 run.
- "Defer Zoopla detail-enrichment for budget" — the 17:55 run, 123 fresh IDs not enriched.
- "Secondary-area scraping skipped (skill provides primary URLs only)" — flatly false; the skill has explicit secondary URLs.

We tried four prompt-level remedies in sequence:
1. Adding a 60-min in-skill wall-clock kill (read by the agent as permission to ration).
2. Adding "Self-budgeting FORBIDDEN" clauses (kept the topic of skipping front-of-mind).
3. Adding "Prior-coverage skips FORBIDDEN" clauses (agent reframed under different language).
4. Removing all runtime-budget content from the skill (helped marginally; agent still surfaced "deferred for budget").

We also added a `Subagent-driven scraping (orchestration)` block specifying 8 portal-tier subagent dispatches via the `Agent` tool. The 20:34 run with that text in place did not dispatch the subagents — the parent inlined the work, did 4 of 8 portal-tiers, and skipped SpareRoom and all secondary areas anyway.

**Root cause:** prompt instructions cannot reliably compel a model to dispatch subagents when it can satisfy the literal task by inlining. Skill-text alone is the wrong layer of enforcement.

**The real fix:** move dispatch out of the skill entirely. A Bash orchestrator fires N narrowly-scoped `claude -p` invocations, each given exactly one phase of work. The parent skill never sees the aggregate scope, so it cannot pre-ration.

---

## 2. Goals & non-goals

### Goals
- Every run scrapes all 4 portals × 16 areas, regardless of model judgment.
- Predictable, observable execution from the shell — failures are visible in exit codes and structured output files, not buried in LLM prose.
- Bounded cost: ≤ 12 `claude -p` invocations per run, each with its own narrow context.
- Per-step state isolation: a hung or buggy invocation can be killed and the rest of the run proceeds.
- No new infrastructure (no queue, no daemon, no DB beyond the existing Notion tracker).

### Non-goals
- Speed improvement. Sequential invocation overhead means runs may be slightly longer than a working inlined run. Reliability is the win, not throughput.
- Parallel scraping. Playwright still runs sequentially per invocation; we don't attempt multi-browser orchestration.
- Restructuring the skill itself. The existing `skill-flats.md` content (HARD FILTERS, SCORING, VISUAL EXTRACTION, etc.) is referenced unchanged by every subagent prompt.
- Replacing claude.ai-hosted MCPs (Gmail, Notion). Each invocation re-fetches connectors at startup; that's the current cost and we accept it.

---

## 3. Architecture

### Pipeline: 11 sequential `claude -p` invocations

```
1.  INIT             — enumerate Notion Flats URL set, write known_urls.txt
2.  ALERTS           — Gmail ingest, dedup against known set, insert qualified rows
3.  SCRAPE rm-primary
4.  SCRAPE zoopla-primary
5.  SCRAPE sr-primary
6.  SCRAPE or-primary
7.  SCRAPE rm-secondary
8.  SCRAPE zoopla-secondary
9.  SCRAPE sr-secondary
10. SCRAPE or-secondary
11. DIGEST           — read run-state files + Notion, build email draft, save outreach .txt files, stamp Meta
```

Bash dispatches them in order. Each `claude -p` invocation has its own claude.ai connector fetch, its own Playwright browser (xvfb-wrapped), its own context. Step N's failure does not abort step N+1.

### Why 11, not fewer

- INIT, ALERTS, and DIGEST have always worked reliably as single-shot agents — they don't have the "long task list" problem. Keeping them as their own invocations is a one-time cost (~3 fixed steps) and a clean separation of concerns.
- The 8 SCRAPE invocations match the structure already specified in `skill-flats.md` (`§ Subagent-driven scraping`). No skill rework needed; only the dispatch site moves from "the agent's `Agent` tool" to "Bash".
- Per-portal-tier (rather than per-portal or per-area) is the granularity sweet spot: small enough that the in-prompt task list (6–8 URLs) cannot be skipped silently; large enough that we don't pay 32 invocations of setup overhead.

### State sharing — file-based, in `~/hunt/run-state/<run-id>/`

```
~/hunt/run-state/2026-04-28T10-00-00/
  known_urls.txt          # one URL per line, written by INIT, appended-to after every successful insert
  init.failed             # flag file (only present if INIT failed) — downstream phases check at startup
  alerts.json             # ALERTS phase result
  scrape-rm-primary.json
  scrape-zoopla-primary.json
  scrape-sr-primary.json
  scrape-or-primary.json
  scrape-rm-secondary.json
  scrape-zoopla-secondary.json
  scrape-sr-secondary.json
  scrape-or-secondary.json
  digest.json             # DIGEST writes a copy of what it sent
  timings.json            # per-step start/end/duration; appended after each step
  log/
    01-init.log           # full claude stdout, one per step
    02-alerts.log
    03-scrape-rm-primary.log
    ...
    11-digest.log
~/hunt/run-state/latest -> 2026-04-28T10-00-00   # symlink updated at run start
```

Run-id format: ISO-8601 with colons replaced by dashes (filesystem-safe).

Run-state directories older than 14 days are pruned by `run-hunt.sh` before each new run.

### `<phase>.json` schema

Every scrape and ingest phase writes a result file with this shape:

```json
{
  "phase": "scrape-rm-primary",
  "started_at": "2026-04-28T10:05:13Z",
  "finished_at": "2026-04-28T10:11:42Z",
  "exit_code": 0,
  "areas": [
    {"area": "Islington",      "outcome": "OK",          "inserted": 3},
    {"area": "Highbury",       "outcome": "EARLY-EXIT",  "inserted": 1},
    {"area": "Camden Town",    "outcome": "OK",          "inserted": 0},
    {"area": "Kentish Town",   "outcome": "BLOCKED",     "inserted": 0},
    {"area": "De Beauvoir",    "outcome": "OK",          "inserted": 2},
    {"area": "Clerkenwell",    "outcome": "ZERO-RESULTS","inserted": 0}
  ],
  "inserted_pages": [
    {"page_id": "...", "tier": "MEDIUM", "score": 6.5, "title": "...", "url": "..."},
    ...
  ],
  "errors": []
}
```

DIGEST reads all `*.json` files in the run-state dir, concatenates `areas` blocks (the per-area accounting), and lists all `inserted_pages` for the email body.

### Subagent prompt template

Bash assembles each scrape prompt from a static skeleton + dynamic args (URLs, run-id, output path). Skeleton at `~/hunt/prompts/scrape.txt`:

```
You are the scrape subagent for {{PORTAL}} {{TIER}}. Skill is at
/home/ara/dev/london-flat-hunt/london-property-hunt-public/skill-flats.md
— read sections § HARD FILTERS, § SIZE RESOLUTION, § SCORING,
§ Scraping loop, § VISUAL EXTRACTION (FLOORPLAN and PHOTOS),
§ TRACKER, § Pagination cutoff, § Rate limiting.

Your scope: process EXACTLY these {{N}} URLs in order.

URLs:
{{URL_LIST}}

Known URL set (for O(1) dedup) is in this file:
{{RUN_STATE_DIR}}/known_urls.txt
Read it once at start; treat each line as a URL. Re-read the file
just before each Notion insert to pick up additions made by earlier
phases of the same run (cheap; small file).

INIT-failure check: if {{RUN_STATE_DIR}}/init.failed exists, write a
phase-skipped JSON to your output path with all areas set to
`outcome=SKIPPED-INIT-FAILED, inserted=0` and exit 0. Do NOT call
notion-create-pages under any circumstances when this flag is
present. The dedup set cannot be trusted; writing would create
duplicates.

For each URL in your assigned list:
  1. Run the scraping loop in § Scraping loop.
  2. Apply hard filters from § HARD FILTERS.
  3. Enrich Rightmove/Zoopla per § VISUAL EXTRACTION.
  4. Score per § SCORING.
  5. Insert qualified rows directly to Notion via
     mcp__claude_ai_Notion__notion-create-pages with
     parent.data_source_id = {{FLATS_DATA_SOURCE_ID}}.
  6. Immediately after each successful insert, append the listing
     URL (canonicalised per § TRACKER) to {{RUN_STATE_DIR}}/known_urls.txt
     with a single newline-terminated write. POSIX guarantees
     small-write atomicity within a single process; future phases
     of this run will see the addition on their next read.

REQUIRED OUTPUT: write a JSON file at
{{RUN_STATE_DIR}}/{{PHASE_NAME}}.json with this shape:

{
  "phase": "{{PHASE_NAME}}",
  "started_at": "<ISO-8601 UTC>",
  "finished_at": "<ISO-8601 UTC>",
  "exit_code": 0,
  "areas": [
    {"area": "<area name>", "outcome": "<OK|EARLY-EXIT|BLOCKED|TIMEOUT|ZERO-RESULTS>", "inserted": <n>},
    ... one entry per assigned URL ...
  ],
  "inserted_pages": [
    {"page_id": "...", "tier": "...", "score": ..., "title": "...", "url": "..."},
    ... one entry per row inserted to Notion ...
  ],
  "errors": ["<one-line error desc>", ...]
}

Write the file using the Write tool, then exit. Do NOT do work
outside your assigned scope (no other portals, no other tiers, no
alert ingestion, no Meta updates, no email digest).
```

Two more skeleton files for INIT (`init.txt`), ALERTS (`alerts.txt`), DIGEST (`digest.txt`) — same shape, different scope.

### Per-step timeout

Each `claude -p` invocation is wrapped in `timeout --signal=TERM --kill-after=30s 25m`. The 25-min figure is a starting estimate (not validated): observed scrape phases on the existing single-skill design have run 30+ min when handling many enrichments, and the 11-step split reduces per-step scope but adds fresh-session cold-start overhead. We measure and tune.

If a step times out, Bash logs `<phase>: TIMEOUT (25m exceeded)`, writes a placeholder `.json` with `exit_code=124, areas=[]`, and proceeds.

The existing 90 min wrapper-level timeout in `run-hunt.sh` is removed — it's now per-step (effectively 11 × 25 = 275 min upper bound, expected 50–90 min typical).

**Bootstrap retry for the first MCP call**: each step's first 60 seconds are treated as cold-start. If the first MCP call (typically `notion-fetch` or `mcp__playwright__browser_navigate`) fails within those 60s with a connection or auth error, Bash kills the step (within 30s of the failure being detected) and re-runs it once. This catches transient claude.ai connector fetch failures (the same class of issue the existing `run-hunt.sh` health-check loop guards against, but now scoped per-step). Max one retry per step; further failures are recorded as `exit_code=non-zero` and the run continues.

**Timing telemetry**: each step's start/end timestamps and duration are appended to `{{RUN_STATE_DIR}}/timings.json`. After 3–5 real runs, we tune the 25-min figure based on observed p95 cold-start + work duration.

### Failure handling

- Per-step exit code captured by Bash.
- Non-zero exit → log + placeholder `.json` (`exit_code != 0`, no `areas`/`inserted_pages`) → continue to next step.
- DIGEST step always runs, even if upstream steps failed. It surfaces failures in the email digest under a `## Failed phases` section.
- No mid-run retries. Tomorrow's cron retries naturally; the URL set is pruned by deduplication so re-running is correct.

### Observability

- `~/hunt/cron.log`: existing tail-able log, gets one line per step (`=== STEP N: <phase> START $(date -Iseconds) ===` / `=== STEP N: <phase> END exit=$? duration=...s ===`).
- `~/hunt/run-state/latest/log/<step>.log`: full claude stdout for one step. Useful for debugging a specific failure without re-running.
- `~/hunt/run-state/latest/*.json`: structured outcomes consumable by tooling, dashboards, future automation.

---

## 4. Components

### `run-hunt.sh` (rewritten)

Existing responsibilities (preserved):
- Health-check loop for claude.ai MCP bootstrap.
- xvfb-run wrapper for Playwright.
- Logging to `~/hunt/cron.log`.

New responsibilities:
- Generate run-id, create `~/hunt/run-state/<run-id>/`.
- Update `~/hunt/run-state/latest` symlink.
- Prune run-state dirs older than 14 days.
- Sequence of 11 invocations: each is `xvfb-run … timeout 15m claude -p "$(cat prompts/<phase>.txt | envsubst)"`.
- After each invocation: log start/end/exit, write placeholder `.json` if missing.

### `~/hunt/prompts/`

```
~/hunt/prompts/
  init.txt
  alerts.txt
  scrape.txt        # parameterised by PORTAL, TIER, URL_LIST, etc.
  digest.txt
```

`envsubst` (or a small Bash heredoc) fills in `{{VAR}}` placeholders.

### `~/hunt/areas.json` (new, declarative config)

Single source of truth for portal-area URLs. Today they're hard-coded in `skill-flats.md`; we extract them to a config file the orchestrator reads to generate scrape prompts.

```json
{
  "rm-primary": [
    {"area": "Islington",       "url": "https://www.rightmove.co.uk/property-to-rent/find.html?locationIdentifier=REGION%5E93965&..."},
    {"area": "Highbury",        "url": "..."},
    ...
  ],
  "zoopla-primary": [...],
  ...
}
```

`skill-flats.md` keeps the URL templates as documentation but the actual URLs live in `areas.json`. When a portal changes its URL scheme, only `areas.json` needs an edit.

### Skill changes

Minimal. The `§ Subagent-driven scraping (orchestration)` block in `skill-flats.md` is replaced by a one-paragraph note saying "scraping is dispatched externally by `run-hunt.sh`; this section's per-portal URLs and scraping loop apply unchanged within each scrape subagent's scope." The per-area accounting requirement stays — but it's now produced by each subagent in `<phase>.json`, not in the email digest body directly.

---

## 5. Data flow

```
Bash (run-hunt.sh)
  │
  ├─[1]─▶ claude -p INIT  ──── reads Notion ──── writes known_urls.txt
  │
  ├─[2]─▶ claude -p ALERTS ─── reads Gmail + known_urls.txt ─── writes alerts.json + Notion rows
  │
  ├─[3]─▶ claude -p SCRAPE rm-primary ─── reads known_urls.txt ─── writes scrape-rm-primary.json + Notion rows
  ├─[4]─▶ claude -p SCRAPE zoopla-primary ─── (same shape)
  ├─[5..10]─▶ ... 6 more scrape subagents ...
  │
  └─[11]─▶ claude -p DIGEST ── reads all *.json + Notion ── writes Gmail draft + outreach/*.txt + Meta stamp
```

The Notion tracker is the source of truth for inserted listings. The `*.json` files are orchestration metadata: counts, outcomes, errors. DIGEST cross-references the two — it queries Notion for full row data of `inserted_pages` referenced in scrape outputs.

---

## 6. Error handling

| Failure                                | Behavior                                                                |
|----------------------------------------|-------------------------------------------------------------------------|
| Step exits non-zero                    | Log `<phase>: FAILED exit=<rc>`, placeholder `.json`, continue          |
| Step exceeds 25-min timeout            | `kill -TERM` + 30s grace then SIGKILL, placeholder `.json`, continue    |
| Step writes invalid JSON               | Bash detects (json parse fails or area count mismatch — see § Bash-side completeness validation); marks phase as INVALID in cron.log; DIGEST surfaces in `## Failed phases` |
| Step's first MCP call fails within 60s | One retry; further failures → exit_code logged, continue (see § Bootstrap retry above) |
| **INIT fails (any reason)**            | **Bash writes `init.failed` flag in run-state dir. Every downstream phase reads the flag at startup and exits with `outcome=SKIPPED-INIT-FAILED` for all assigned areas, writes ZERO Notion rows. Fail-closed: better to skip a day than create duplicate rows when dedup is unreliable.** DIGEST surfaces the run-aborted state prominently in the email. |
| Notion MCP outage                      | INIT's first call fails → falls under "INIT fails" above (init.failed flag → all phases no-op). |
| Playwright unavailable                 | Scrape phases probe Playwright at start; if probe fails, write `outcome=BLOCKED` for all assigned areas, exit cleanly. ALERTS and DIGEST still run (they don't need Playwright). |
| `xvfb-run` itself fails                | Bash detects the unusual exit signature; logs and proceeds with whatever steps can run (alerts, digest) without Playwright. |

The pattern: every failure produces a structured outcome rather than aborting the run. INIT failure is the one exception — it triggers fail-closed on all writes because dedup is the only correctness guarantee against duplicate rows.

### Bash-side completeness validation

After each phase exits, before moving to the next, Bash performs three independent checks (the model's self-report in the JSON is **not** trusted as the only signal):

1. **JSON parse**: `jq . <phase>.json > /dev/null`. If parse fails, mark phase INVALID.
2. **Area-count match**: for scrape phases, the input URL count is known to Bash (it generated the prompt). The output `areas[]` array length must match exactly. If shorter, the phase silently dropped areas; mark INVALID and emit the missing area names to cron.log.
3. **Outcome enum**: each `areas[*].outcome` must be in the documented enum (`OK | EARLY-EXIT | BLOCKED | TIMEOUT | ZERO-RESULTS | SKIPPED-INIT-FAILED`). Out-of-vocabulary values mark INVALID.

This addresses the per-area dropping case: a subagent that "claims OK" without producing the full list of areas is caught by check 2. **It does NOT catch per-listing under-enrichment within an area** — that remains a model-trust gap. If observed in practice, the future redesign is to have Bash extract listing URLs itself (deterministic curl + portal-specific regex on search pages) and dispatch enrichment-only subagents with explicit one-URL-per-subagent scope. See § 9 Open questions.

---

## 7. Testing

### Manual smoke
- `bash run-hunt.sh` from a logged-in shell. Watch cron.log + `~/hunt/run-state/latest/`.
- Verify all 11 `.json` files present after run.
- Verify Notion has the expected new rows.
- Verify Gmail draft was created.

### Adversarial single-step
- Run any one of the prompt files manually: `cat ~/hunt/prompts/scrape.txt | envsubst | xvfb-run timeout 15m claude -p` with `RUN_STATE_DIR=/tmp/test-run` and a hand-crafted URL list. Useful for iterating on prompts without running the full pipeline.

### Failure-path tests
- Simulate a hung step by replacing one prompt with a script that sleeps 20m. Confirm Bash kills it at 15m and the rest of the run proceeds.
- Simulate Notion outage by temporarily renaming the Notion MCP config. Confirm INIT detects, downstream steps no-op cleanly.
- Run with no `xvfb-run` (Playwright will fail). Confirm degraded-mode digest is sent.

No automated test suite — this is a daily cron job with low blast radius. Manual verification on the next morning's run is the test.

---

## 8. Migration plan (deferred to writing-plans)

This is not the implementation plan — that comes next. Rough sketch:
1. Write `areas.json` from existing skill content.
2. Write four prompt skeletons.
3. Rewrite `run-hunt.sh` for the 11-step pipeline.
4. Strip the orchestration block from `skill-flats.md`, keep the rest unchanged.
5. Test manually, then let cron drive the next morning.

Implementation plan (the next document) will break each into bite-sized tasks.

---

## 9. Open questions / future work

- **Per-listing under-enrichment** is the residual model-trust gap not addressed by this design. The Bash-side completeness validation (§ 6) catches per-area dropping but not "subagent ran 4 of 6 listings within an area, marked the area OK". If observed in practice (telltales: low enrichment counts despite high `inserted` counts in the per-area accounting), the structural fix is to split the LLM's job further:
  - **Bash extracts listing URLs** itself from each search page using `curl` + portal-specific regex (Rightmove: `/properties/\d+`, Zoopla: `/to-rent/details/\d+/`, etc). No model judgment, no skipping possible.
  - **LLM is only used for per-listing enrichment** — one subagent per listing URL, with Playwright + photos + floorplan as the only task. Verification trivial: did the listing get a Notion row or not.
  This is a meaningful redesign — probably 2-3x the work of the current spec — and is not built day one. Codex's high-severity finding on self-attestation correctly notes this as a residual risk; we accept it for now and will revisit if the symptoms persist.
- **Run-state dir compaction.** 14-day retention is a guess; if dir count balloons we'll cap at N runs instead.
- **Phase-split for enrichment** (option A2 from brainstorming). Related to the per-listing concern above; if a single portal-tier subagent handling 6–8 areas + per-listing enrichment becomes the new bottleneck (e.g., one subagent runs 30 min and times out), we split scraping into "search" (per-portal-tier) and "enrich" (batches of N listings). Same direction as the deeper redesign.
- **Areas.json self-healing.** When a portal changes its URL scheme (Zoopla has churned 3 times in two weeks), the orchestrator could detect ZERO-RESULTS across all areas of a portal and fire a "discovery" subagent to find the new pattern. Day-two enhancement.
- **Cost telemetry.** Track per-step input/output tokens and aggregate per-run cost. Useful for tuning timeouts and detecting prompt-size regressions.
- **25-min per-step timeout is unmeasured.** Starting estimate; tune from `timings.json` after 3–5 real runs.

---

## 10. Decision log

- **2026-04-27**: chose option A (externally orchestrate scraping only — INIT/alerts/digest stay in their own claude invocations) over option C (post-run validator with fixup), because every observed run skips something, so C ends up being A with extra detection ceremony.
- **2026-04-27**: chose granularity A1 (8 portal-tier subagents) over A2 (phase-split search/enrich) and A3 (4 portal subagents). A1 mirrors the existing skill structure (cheap migration), 6–8 areas per subagent is small enough that skipping is structurally hard, and A2's phase-split adds new moving parts we don't need yet.
- **2026-04-27**: file-based state (run-state dir + json + log files) over stdout parsing. LLM stdout is unreliable; file-based gives Bash a clean contract and gives humans an audit trail.
- **2026-04-27 (post-codex-review)**: dedup is now append-as-you-insert. Each phase appends inserted URLs to `known_urls.txt` immediately after each successful Notion write. Closes the codex critical-1 finding (stale dedup state across phases).
- **2026-04-27 (post-codex-review)**: INIT failure is now fail-closed. `init.failed` flag → all downstream phases exit with `SKIPPED-INIT-FAILED` and zero Notion writes. Closes the codex critical-2 finding (fail-open posture).
- **2026-04-27 (post-codex-review)**: Bash-side completeness validation (JSON parse + area-count match + outcome-enum check) added in §6. Closes the codex high-severity finding partially — catches per-area dropping but not per-listing under-enrichment. The residual risk is acknowledged in § 9; structural fix (Bash extracts URLs, LLM enriches one-at-a-time) deferred until observed in practice.
- **2026-04-27 (post-codex-review)**: per-step timeout 15 → 25 min, with per-step timing telemetry to `timings.json` and bounded retry on first-MCP-call cold-start failures. Closes the codex high-severity finding on the unvalidated 15-min budget.
