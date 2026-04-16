---
title: BTW Computation — Best Transfer Winds
created: 2026-04-14
updated: 2026-04-15
type: concept
tags: [nwp, non-convective wind, parameterization, btw, wind-profile]
sources: []
---

# BTW Computation (Best Transfer Winds)

Methodology for computing Best Transfer Winds from NAM profile data, the core wind metric used by [[havoc]] for non-convective wind threat assessment.

## What BTW Measures

BTW estimates the wind speed that will be realized at the surface given the thermodynamic and kinematic profile above. It interpolates between the mean wind (0% transfer) and the strongest wind in a layer (100% transfer), weighted by the downward forcing percentage from [[havoc-downward-forcing]].

## Standard BTW Levels

Computed at 10% AGL increments from surface to 9 kft (0.9 × 10 kft):

| Level ID | AGL Range | Notes |
|----------|-----------|-------|
| 010 | 0-1 kft | Surface mixed layer |
| 020 | 1-2 kft | |
| 030 | 2-3 kft | |
| 040 | 3-4 kft | |
| 050 | 4-5 kft | Default MAX BTW cap for most locations |
| 060 | 5-6 kft | |
| 070 | 6-7 kft | |
| 080 | 7-8 kft | |
| 090 | 8-9 kft | Extended for downslope (KRNO/KRTS) |

Each level computes wind speed from NAM isobaric heights interpolated to the AGL altitude.

## Computation Steps

### 1. Extract wind profile from NAM

For each station, read U and V wind components at all 39 isobaric levels (1000-50 hPa). Convert to wind speed and direction.

### 2. Map isobaric heights to AGL

Subtract station elevation from each isobaric level's geopotential height to get AGL. Bin winds into the 010-090 layers by AGL.

### 3. Compute BTW at each level

For each AGL layer (010, 020, ..., 090):

```
btw_level = mean_wind_in_layer  (simplified starting point)
```

The BTW at each level is the representative wind speed for that layer — initially the mean, later weighted by downward forcing.

### 4. Apply downward forcing weighting

Once [[havoc-downward-forcing]] percentage (DF%) is determined:

```
if DF% ≈ 0% (strong lift):
    BTW_surface = mean(btw_010 ... btw_090)  # average of full column
    
if DF% ≈ 100% (strong subsidence):
    BTW_surface = max(btw_010 ... btw_050)   # max in low-mid levels
    # For downslope locations (KRNO/KRTS): max(btw_010 ... btw_090)
    
if DF% between 0-100%:
    BTW_surface = (1 - DF%) × mean(btw_levels) + DF% × max(btw_levels)
    # Where level depth depends on location type and DF%
```

### 5. Lapse rate correction

BTW 010 lapse rate (°C/km) is already computed and exported as `btw_010_lapse_C_km`. This captures low-level stability — steep lapse (near dry adiabatic) supports more efficient momentum transfer to the surface.

## Station-Specific Configurations

| Station | Type | BTW Depth at 100% | Rationale |
|---------|------|--------------------|-----------|
| Default | Standard | 010-050 | Most US locations |
| KRNO | Downslope | 010-090 | Sierra Nevada lee-side |
| KRTS | Downslope | 010-090 | Sierra Nevada lee-side |
| KDEN | Lee-side | 010-060 | Rocky Mtn downslope (intermediate) |

Additional downslope stations may be added as events are logged.

## Data Flow

```
NAM GRIB2 → nam_profile_extractor.py → station JSON profiles
                                              ↓
                                        BTW computation
                                              ↓
                                    downward forcing weighting
                                              ↓
                                    BTW surface wind estimate
```

## Known Issues

### cfgrib paramId Mismatch (2026-04-15)

When extracting NAM GRIB2 wind data with cfgrib, opening `u` and `v` wind components together (`paramId=[131,132]`) causes a **silent failure**: cfgrib returns an empty dataset with zero data variables and no error. This happens because most NAM isobaric variables have 39 pressure levels while u/v have 42 — the `isobaricInhPa` coordinate dimension conflicts during the merge, and cfgrib silently drops both variables.

**Fix in `nam_profile_extractor.py`:** Split the combined `isobaric_uv` query into two separate dataset queries:
- `"isobaric_u": {paramId: 131}`
- `"isobaric_v": {paramId: 132}`

Read u from `datasets["isobaric_u"]` and v from `datasets["isobaric_v"]` instead of both from the combined `datasets["isobaric_uv"]`.

**Symptom:** If BTW values or wind speeds show as `None` in station profiles, check whether the u/v dataset queries are still combined.

## Related

- [[havoc]] — HAVOC agent definition
- [[havoc-downward-forcing]] — Downward forcing scale (0-100%)
- [[nam-data-pipeline]] — Full pipeline from GRIB2 download to threats display
- [[vile-orchestrator]] — V.I.L.E. framework