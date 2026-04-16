---
title: NAM Data Pipeline — Retrieval to Threats Display
created: 2026-04-15
updated: 2026-04-15
type: concept
tags: [nwp, parameterization, model-resolution, obs-data]
sources: []
---

# NAM Data Pipeline

End-to-end pipeline from NAM 12km GRIB2 data download through station profile extraction to the threats display grid on the dashboard. Four stages, cron-automated.

## Stage 1 — Retrieval

**Script:** `scripts/grib2_download/nam_smart_downloader.py`
**Shell orchestrator:** `scripts/grib2_download/nam_pipeline.sh`
**Check script:** `scripts/grib2_download/checkers/check_nam_cycle.py`

- Checks whether a new NAM 12z/00z/06z/18z cycle is available on NOAA NOMADS
- Downloads full cycle: `nam.tHHz.awphysFF.tm00.grib2` (f00–f84, 3-hourly, 29 files total)
- Destination: `/mnt/d/weather_data/nam_grib2/`
- Checkpoint file: `.nam_download_checkpoint` tracks last completed cycle
- Retries on failure, monitors until completion
- Runs via cron aligned with NAM release (~3 hours after cycle start)

## Stage 2 — Grid Processing (cfgrib)

**Script:** `nam_profile_extractor.py` (main extractor)
**Helper:** `nam_grib2_processing.py` (legacy processing)

Opens each GRIB2 file via cfgrib/xarray with `filter_by_keys` queries. The Lambert conformal grid (428×614) has 2D lat/lon coordinate arrays. Station nearest-neighbor is computed via great-circle distance to find the closest grid point.

### cfgrib Dataset Queries (post-fix)

Each GRIB2 file is opened with multiple `filter_by_keys` queries to isolate level types:

| Query Key | paramId | Variable | Levels |
|-----------|---------|----------|--------|
| `surface_instant` | — | SP, T, VIS, GUST, CAPE, CIN, LSM, OROG | Surface |
| `wind_10m` | — | u10, v10 | 10m AGL |
| `mean_sea_level` | — | MSLP | MSL |
| `isobaric_temp` | 130 | Temperature | 39 isobaric |
| `isobaric_gh` | 156 | Geopotential height | 39 isobaric |
| `isobaric_u` | 131 | U-wind component | 39 isobaric* |
| `isobaric_v` | 132 | V-wind component | 39 isobaric* |
| `isobaric_omega` | 135 | Omega (vertical velocity) | 39 isobaric |
| `isobaric_rh` | 157 | Relative humidity | 39 isobaric |
| `isobaric_div` | 155 | Divergence | 5 levels (1000,850,700,500,250) |
| `isobaric_absv` | 190 | Absolute vorticity | 5 levels (1000,850,700,500,250) |

*See [[btw-computation]] Known Issues for the cfgrib paramId split that was required.*

### Station Extraction

- Reads tracked stations from `station_master.csv` (Tracked = "y")
- Nearest-neighbor interpolation on the Lambert grid for each station lat/lon
- Unit conversions: K→°C, Pa→hPa, m/s→kt, m→km
- Output per cycle: `profiles/nam_{date}_{cycle}z_info.json` + per-station JSON files

## Stage 3 — Profile Creation

Each station JSON contains all forecast hours (f00–f84, 3-hourly) with:

- Full isobaric sounding (T, GH, U, V, omega, RH at 39 pressure levels)
- 10m wind (u10, v10)
- Surface variables (SP, T, VIS, GUST, CAPE, CIN, LSM, OROG)
- Tropopause and max-wind level data
- PBL height

The BTW computation ([[btw-computation]]) runs at profile-read time in the dashboard (`app.py`) — not during extraction. This means profile JSONs contain raw data, and BTW/output metrics are computed per-request.

## Stage 4 — Threats Display

**Dashboard route:** `/threats`
**API endpoint:** `/api/nam/threats`
**Template:** `THREATS_HTML` (inline in app.py)

The threats page renders a station×forecast-hour grid:

- Each cell is a glyph icon representing the threat level for one station at one fhr
- Stations are rows, forecast hours are columns
- BTW values computed by `_compute_btw_for_station()` drive the wind threat icons
- Downward forcing percentage from [[havoc-downward-forcing]] determines BTW depth selection
- Color-coded by severity (wind speed thresholds)
- Station tooltips show station code + name

### Threat Agent Mapping

Each station row maps to the V.I.L.E. sub-agent responsible for that threat type:

- [[havoc]] produces non-convective wind threat assessments (BTW, omega, divergence)
- [[maul]] evaluates tornadogenesis potential
- [[blast]] handles convective wind
- Other agents (blight, drown, smash, glaze) have their own parameters but share the same grid structure

### Station-Specific BTW Depth

| Station | Depth Type | BTW Levels at 100% DF | Rationale |
|---------|-----------|----------------------|-----------|
| Default | standard | 010–050 | Most US locations |
| KRNO | downslope | 010–090 | Sierra Nevada lee-side |
| KRTS | downslope | 010–090 | Sierra Nevada lee-side |
| KDEN | lee-side | 010–070 | Rocky Mtn downslope (intermediate) |

## Known Issues

### cfgrib paramId Mismatch (2026-04-15)

See [[btw-computation]] for full details. When u and v wind are opened together (`paramId=[131,132]`), cfgrib silently returns an empty dataset. Fixed by splitting into separate `isobaric_u` (131) and `isobaric_v` (132) queries.

### Unextracted Variables

The following NAM variables are available in the GRIB2 but not yet extracted for threat computation:

- Divergence (paramId 155) — 5 standard levels, used by [[havoc]] for downward forcing refinement
- Absolute vorticity (paramId 190) — 5 standard levels, used by [[havoc]] for NVA identification
- Relative vorticity (paramId 44) — not yet extracted

### Computed Variables Not Yet Implemented

- Equivalent potential temperature (θe) — requires T, RH, P at each level
- Lapse rate (already computed as `btw_010_lapse_C_km` for BTW layer)
- Stability indices: LI, SWEAT, TT — not yet in the pipeline

## File Paths

| Component | Path |
|-----------|------|
| GRIB2 data | `/mnt/d/weather_data/nam_grib2/nam.*.grib2` |
| Profiles output | `/mnt/d/weather_data/nam_grib2/profiles/` |
| Station master | `metar-automation/station_master.csv` |
| Profile extractor | `metar-automation/nam_profile_extractor.py` |
| Pipeline shell | `metar-automation/scripts/grib2_download/nam_pipeline.sh` |
| Smart downloader | `metar-automation/scripts/grib2_download/nam_smart_downloader.py` |
| Dashboard app | `nws_dashboard/app.py` |
| Dashboard log | `/var/log/nws_dashboard/dashboard.log` |

## Related

- [[btw-computation]] — BTW level computation and surface weighting
- [[havoc-downward-forcing]] — 0-100% downward forcing scale
- [[havoc]] — HAVOC agent (non-convective wind)
- [[vile-orchestrator]] — V.I.L.E. framework overview