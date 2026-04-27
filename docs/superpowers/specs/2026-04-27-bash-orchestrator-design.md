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
  known_urls.txt          # one URL per line, written by INIT
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
Read it once at start; treat each line as a URL. Skip any candidate
URL that appears in the set.

For each URL in your assigned list:
  1. Run the scraping loop in § Scraping loop.
  2. Apply hard filters from § HARD FILTERS.
  3. Enrich Rightmove/Zoopla per § VISUAL EXTRACTION.
  4. Score per § SCORING.
  5. Insert qualified rows directly to Notion via
     mcp__claude_ai_Notion__notion-create-pages with
     parent.data_source_id = {{FLATS_DATA_SOURCE_ID}}.

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

Each `claude -p` invocation is wrapped in `timeout --signal=TERM --kill-after=30s 15m`. 15 min per step is generous; typical steps run 3–8 min. If a step times out, Bash logs `<phase>: TIMEOUT (15m exceeded)`, writes a placeholder `.json` with `exit_code=124`, and proceeds.

The existing 90 min wrapper-level timeout in `run-hunt.sh` is removed — it's now per-step (effectively 11 × 15 = 165 min upper bound, ~50–75 min typical).

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
| Step exceeds 15-min timeout            | `kill -TERM` + 30s grace then SIGKILL, placeholder `.json`, continue    |
| Step writes invalid JSON               | DIGEST treats as failed phase, surfaces in `## Failed phases`           |
| INIT fails                             | Subsequent scrape steps still run with empty `known_urls.txt` (degraded mode — risk of duplicate rows; DIGEST flags this) |
| Notion MCP outage                      | Detected by INIT's first call; INIT writes `outage.flag`, all subsequent steps no-op + log; DIGEST writes a "Notion unavailable, run aborted" email if it runs at all |
| Playwright unavailable                 | First scrape step's Playwright probe fails; that step writes `outcome=BLOCKED` for all its areas; subsequent scrape steps re-probe and likely also fail; DIGEST flags scraping unavailable |
| `xvfb-run` itself fails                | Bash detects the unusual exit signature; logs and proceeds with whatever steps can run (alerts, digest) without Playwright |

The pattern: every failure produces a structured outcome rather than aborting the run.

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

- **Run-state dir compaction.** 14-day retention is a guess; if dir count balloons we'll cap at N runs instead.
- **Phase-split for enrichment** (option A2 from brainstorming). If a single portal-tier subagent handling 6–8 areas + per-listing enrichment becomes the new bottleneck (e.g., one subagent runs 30 min and times out), we'll split scraping into "search" (per-portal-tier) and "enrich" (batches of N listings). Not built day one.
- **Areas.json self-healing.** When a portal changes its URL scheme (Zoopla has churned 3 times in two weeks), the orchestrator could detect ZERO-RESULTS across all areas of a portal and fire a "discovery" subagent to find the new pattern. Day-two enhancement.
- **Cost telemetry.** Track per-step input/output tokens and aggregate per-run cost. Useful for tuning timeouts and detecting prompt-size regressions.

---

## 10. Decision log

- **2026-04-27**: chose option A (externally orchestrate scraping only — INIT/alerts/digest stay in their own claude invocations) over option C (post-run validator with fixup), because every observed run skips something, so C ends up being A with extra detection ceremony.
- **2026-04-27**: chose granularity A1 (8 portal-tier subagents) over A2 (phase-split search/enrich) and A3 (4 portal subagents). A1 mirrors the existing skill structure (cheap migration), 6–8 areas per subagent is small enough that skipping is structurally hard, and A2's phase-split adds new moving parts we don't need yet.
- **2026-04-27**: file-based state (run-state dir + json + log files) over stdout parsing. LLM stdout is unreliable; file-based gives Bash a clean contract and gives humans an audit trail.
