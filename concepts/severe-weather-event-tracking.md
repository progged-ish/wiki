# Severe Weather Event Tracking for WCONUS Locations

## Overview
A system for tracking severe weather events near specific support locations across the Western Continental United States. Integrates real-time event data from NWS/Iowa State LSR and persists events to a running database for validation of the V.I.L.E. forecast framework.

---

## Data Sources

### pull_weather_data.py
**Location:** `/mnt/d/share/share/pull_weather_data.py`

**Ingestion window:** 12Z day-of → 12Z next day (24h)

**Sources:**
- **METAR** — Iowa State MESONET ASOS (`mesonet.agron.iastate.edu`)
  - Accepts :50–:59 minute reports only
  - Extracts: tmpf, dwpf, drct, sknt, gust, vsby, mslp, p01i, ceiling, clouds, peak wind gusts
- **LSR** — Iowa State GeoJSON (`mesonet.agron.iastate.edu/geojson/lsr.geojson`)
  - Search radius: 10 miles per site
  - Matched to METAR at top of hour *following* LSR timestamp

**Historical coverage:** March 2026–present (~10 dated JSON files in `data/`)

---

## Support Locations

**50 sites** across WA, OR, MT, ID, WY, UT, CA, NV, CO, AZ, NM. Zones: **N** (North), **C** (Central), **S** (South).

| Zone | States | Count |
|------|--------|-------|
| N | WA, OR, MT, ID, WY, UT | ~14 |
| C | CA, NV, CO | ~16 |
| S | CA, AZ, NM | ~20 |

Key sites: K1YT (Yakima TC), KPDX (Portland), KGTF (Great Falls), KSLC (Salt Lake), KBOI (Boise), KCYS (Cheyenne), KABQ (Kirtland), KPHX (Phoenix), KTUS (Tucson), KEDW (Edwards), and others.

*Full list in `/mnt/d/share/config/locations.json`*

---

## Historical Event Counts (March 2026)

| Date | Sites | METAR Obs | LSRs |
|------|-------|-----------|------|
| 2026-03-13 | 50 | 960 | 16 |
| 2026-03-14 | 50 | 1003 | **35** |
| 2026-03-15 | 50 | 996 | 26 |
| 2026-03-16 | 50 | 1016 | 12 |
| 2026-03-17 | 50 | 1007 | 3 |

Event types observed (2026-03-14 sample):
- `SNOW`: 15
- `NON-TSTM WND GST`: 12
- `SNOW SQUALL`: 8

*Note: March was still winter — convective severe (hail, tornado, tstm wind) season ramps up Apr–Sep*

---

## Proposed Database Schema (SQLite)

### Table: `events`
```sql
CREATE TABLE events (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp       TEXT NOT NULL,          -- ISO 8601 UTC (LSR valid time)
    event_type      TEXT NOT NULL,           -- SNOW, HAIL, TORNADO, etc.
    magnitude       REAL,                   -- e.g., wind mph, hail inches
    magnitude_unit  TEXT,                   -- mph, inches, etc.
    latitude        REAL NOT NULL,
    longitude       REAL NOT NULL,
    distance_mi     REAL NOT NULL,          -- from nearest support site
    source          TEXT NOT NULL,           -- LSR, NWS_WARN, METAR_PEAK
    site_id         TEXT NOT NULL,          -- nearest support location ID
    metar_timestamp TEXT,                   -- matched METAR observation time
    remarks         TEXT,
    raw_json        TEXT,                   -- full LSR payload for replay
    created_at      TEXT DEFAULT (datetime('now'))
);
```

### Table: `support_locations`
```sql
CREATE TABLE support_locations (
    site_id     TEXT PRIMARY KEY,            -- matches locations.json ID
    name        TEXT,
    state       TEXT,
    zone        TEXT,                       -- N / C / S
    latitude    REAL,
    longitude   REAL,
    elevation   REAL
);
```

### Table: `model_profiles`
```sql
CREATE TABLE model_profiles (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    cycle_time      TEXT NOT NULL,          -- YYYY-MM-DD HHZ
    model           TEXT NOT NULL,          -- ERA5, NAM12, HRRR
    profile_path    TEXT,                   -- file path to profile data
    site_id         TEXT,                   -- NULL = grid point only
    latitude        REAL,
    longitude       REAL,
    variables       TEXT,                   -- JSON list of extracted vars
    created_at      TEXT DEFAULT (datetime('now'))
);
```

### Table: `vile_predictions`
```sql
CREATE TABLE vile_predictions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    cycle_time      TEXT NOT NULL,
    site_id         TEXT NOT NULL,
    agent           TEXT NOT NULL,          -- MAUL, BLAST, SMASH, etc.
    threat_type     TEXT,                   -- hail, wind, tornado, etc.
    probability     REAL,                   -- 0.0–1.0
    severity        TEXT,                   -- LOW, MOD, HIGH, EXTREME
    lead_time_hrs   REAL,
    created_at      TEXT DEFAULT (datetime('now'))
);
```

---

## Model Profile Linking Strategy

For each event, find the closest forecast cycle *prior to* the event timestamp:

| Model | Cycles | Forecast Hours | Resolution |
|-------|--------|---------------|------------|
| ERA5 reanalysis | Hourly | `analysis` (00Z lag) | 0.25° (~28km) |
| NAM GRIB2 | 00/06/12/18Z | f00–f84 | 12km |
| HRRR | 00/18Z (opt) | f00–f18 | 3km |

**Linking logic:**
1. Take event timestamp → round down to nearest model cycle hour
2. For ERA5: extract nearest grid point to event lat/lon
3. For NAM: match to fff based on event time − cycle time
4. Store profile path + key variables (CAPE, SRH, lapse rates, etc.) in `model_profiles`

---

## V.I.L.E. Validation Metrics

| Metric | Formula | Interpretation |
|--------|---------|----------------|
| POD | TP / (TP + FN) | Did VILE catch it? |
| FAR | FP / (TP + FP) | VILE cry wolf rate |
| CSI | TP / (TP + FP + FN) | Overall skill |
| Lead time | Event time − Alert time | Usable warning window |
| Miss rate | FN / (TP + FN) | What VILE missed |

Computed per: event type, site/region, month, lead time bucket (0–6h, 6–12h, 12–24h)

---

## System Architecture

```
pull_weather_data.py (D:\share\)
        ↓  (writes to data/wx_obs_{date}.json)
   SQLite DB ← events extracted from JSON, deduplicated
        ↓
   model_profiles ← ERA5/NAM/HRRR grid extraction
        ↓
   vile_predictions ← forecast_approach outputs
        ↓
   Validation Layer → POD/FAR/CSI per event type & location
        ↓
   Dashboard / Report
```

---

## Dashboard (Flask) — Filterable Event Browser

**Location:** `/home/progged-ish/wx_events/dashboard/`
**Port:** 5001
**Template:** `templates/events_dashboard.html`

The dashboard serves as the primary UI for browsing and filtering the event database. It connects to `wx_events.db` (SQLite) via a Flask REST API.

### API: `/api/events`

Supports comprehensive filtering and sorting:

| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Event type filter (e.g. `HAIL`, `TORNADO`, `WIND_GUST`) |
| `site` | string | Support site ID (e.g. `KPDX`, `KSLC`) |
| `state` | string | State abbreviation (e.g. `CO`, `UT`) |
| `days` | int or `all` | Lookback window; `all` disables date cutoff |
| `start_date` | YYYY-MM-DD | Custom range start (overrides `days`) |
| `end_date` | YYYY-MM-DD | Custom range end |
| `min_dist` | float | Minimum distance from site (miles) |
| `max_dist` | float | Maximum distance from site (miles) |
| `sort_by` | string | Sort field: `timestamp`, `distance`, `magnitude`, `type` |
| `sort_order` | string | `asc` or `desc` |
| `limit` | int | Results per page (default 200) |
| `offset` | int | Pagination offset |

### Dashboard UI Features

- **Type filter** — dropdown for all canonical event types
- **Site filter** — populated dynamically from `support_locations` table
- **State filter** — populated dynamically from site data
- **Date range pickers** — custom `start_date` / `end_date` inputs (YYYY-MM-DD)
- **Distance range** — `min_dist` / `max_dist` inputs (miles)
- **Sort controls** — field selector + asc/desc toggle
- **Days quick-select** — 7/14/30/60/90/365/all presets
- **Clear Filters** button — resets all filters to defaults
- **Pagination** — prev/next + page indicator

### Related Routes

| Route | Description |
|-------|-------------|
| `GET /api/telemetry` | Summary stats (total events, 7d/30d counts, type breakdown) |
| `GET /api/sites` | All support locations with event counts |
| `GET /api/summary` | Monthly event breakdown by type |
| `GET /api/status` | Service health + DB path |

---

## System Architecture

```
pull_weather_data.py (D:\share\)
        ↓  (writes to data/wx_obs_{date}.json)
   SQLite DB ← events extracted from JSON, deduplicated
        ↓
   model_profiles ← ERA5/NAM/HRRR grid extraction
        ↓
   vile_predictions ← forecast_approach outputs
        ↓
   Validation Layer → POD/FAR/CSI per event type & location
        ↓
   Dashboard / Report
```

---
*Concept started: 2026-04-15*
*Updated: 2026-04-15 (added real locations, historical counts, schema draft)*
