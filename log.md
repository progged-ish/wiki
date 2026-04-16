# Wiki Log

> Chronological record of all wiki actions. Append-only.
> Format: `## [YYYY-MM-DD] action | subject`
> Actions: ingest, update, query, lint, create, archive, delete

## [2026-04-09] create | V.I.L.E. agent documentation wiki initialized
- Domain: Forecast approaches in meteorology
- Structure created with SCHEMA.md, index.md, log.md
- 7 sub-agent entity pages created (maul, blast, havoc, blight, drown, smash, glaze)
- Main VILE Orchestrator entity page created
- wikilinks configured between entity pages

## [2026-04-09] update | Schema defined
- File names: lowercase, hyphens, no spaces
- YAML frontmatter required on every page
- Wikilinks minimum 2 outbound links per page
- Updated index.md shows 8 total pages

## [2026-04-14] create | HAVOC downward forcing concept
- Created concepts/havoc-downward-forcing.md
- Documents Option A (omega-primary + divergence-refined) approach
- Includes 0-100% scaling framework, BTW depth selection, threshold table
- Notes divergence (paramId 155) and abs vorticity (paramId 190) extraction plan
- Documents rejected alternatives B (time-tendency) and C (grid stencil)
- Updated nam-grib2-processing skill with downward forcing architecture
- Updated index.md — now 9 pages

## [2026-04-14] create | BTW computation concept
- Created concepts/btw-computation.md
- Documents BTW level structure (010-090), computation steps, downward forcing weighting formula
- Station-specific configs: KRNO/KRTS use 010-090 depth, KDEN uses 010-060
- Links to havoc-downward-forcing and havoc entity pages
- Updated index.md — now 10 pages

## [2026-04-15] update | cfgrib wind data bug — BTW computation restored
- Bug: cfgrib silently returns empty dataset when opening paramId=[131,132] (u+v together) from NAM GRIB2
- Root cause: isobaricInhPa level-count mismatch (most vars have 39 levels, u/v have 42); cfgrib merges them into one dataset, coordinate conflict, silently drops both
- Fix: split `isobaric_uv` into separate `isobaric_u` (paramId=131) and `isobaric_v` (paramId=132) dataset queries in nam_profile_extractor.py
- Updated extraction logic (~line 498) to read u/v from separate datasets
- All 20260415 12z profiles regenerated with wind data across 39 levels
- BTW computation and threat icons restored
- Updated [[btw-computation]] with Known Issues section documenting this pitfall
- Updated [[havoc]] with data dependency note

## [2026-04-15] create | NAM data pipeline concept page
- Created concepts/nam-data-pipeline.md
- Documents the four-stage pipeline: retrieval → grid processing → profile creation → threats display
- Includes cfgrib dataset query table (all paramId/level mappings post-fix)
- Station-specific BTW depth configurations
- Known issues: cfgrib paramId split, unextracted variables, not-yet-computed indices
- File path reference table for all pipeline components
- Cross-linked with [[btw-computation]], [[havoc]], [[havoc-downward-forcing]], [[vile-orchestrator]]
- Updated [[havoc]], [[blast]], [[maul]] with pipeline cross-references
- Updated index.md — now 12 pages

## [2026-04-15] update | Enhanced WX Event Tracker with filtering/sorting options
- Added comprehensive filter options to `/api/events` backend: `start_date`, `end_date`, `min_dist`, `max_dist`, `sort_by`, `sort_order`
- Backend: `days=all` now supported to disable date cutoff entirely
- Frontend: Added Site dropdown (populated from `/api/sites`), date range pickers (start/end), distance range inputs (min/max miles), sort-by + asc/desc controls, Clear Filters button
- Sort options: `timestamp` (default), `distance`, `magnitude`, `type` — each with asc/desc toggle
- Site filter dropdown auto-populated from `support_locations` table
- Distance range filters: `e.distance_mi >= ?` and `e.distance_mi <= ?`
- Updated [[severe-weather-event-tracking]] concept page with full API parameter table and UI feature list
- Updated index.md header with total pages

## [2026-04-14] update | HAVOC downward forcing — zero-floor exponential formula
- Replaced tanh-based threshold table with zero-floor + exponential saturation
- Any omega ≤ 0 → 0% DF (no downward forcing under any lift)
- DF% = 100 × (1 − e^(−α × omega)), starting α = 0.3
- Documents why exponential over tanh, primary input level (500 hPa), calibration plan
- 5 action items updated

## [2026-04-15] create | Merging Flask servers without version control — engineering lesson
- Root cause: rogue duplicate 'if __name__ == "__main__": app.run()' block inserted at line ~2168
  during NWS Dashboard + WX Event Tracker merge, before THREATS_HTML/WIKI_HTML/SKEWT_HTML/DASHBOARD_HTML defined
- Flask started serving at line 2170; all routes referencing DASHBOARD_HTML raised NameError at runtime
- Import-time tests passed (python -c "import app") because app.run() is never called on import
- Fix: grep -n "app.run" app.py, identified duplicate block, removed it with patch tool
- Validated with app.test_client() before restarting; all routes HTTP 200
- Server operational 2026-04-15 22:07 UTC, actively serving traffic
- Memory updated with CRITICAL note about module-level HTML variable ordering
- Created concepts/merging-flask-servers-without-version-control.md
- Updated index.md — now 13 pages
- Action items: initialize git in /home/progged-ish/nws_dashboard/, add start.sh pre-flight check