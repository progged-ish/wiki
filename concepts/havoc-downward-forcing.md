---
title: HAVOC Downward Forcing Scale
created: 2026-04-14
updated: 2026-04-14
type: concept
tags: [nwp, non-convective wind, parameterization, btw, omega, divergence]
sources: []
---

# HAVOC Downward Forcing Scale (0-100%)

Quantifies how much high-altitude momentum transfers to the surface via vertical motion. Controls the BTW (Best Transfer Wind) depth and weighting in [[havoc]] analysis.

## The Scale

| Downward Forcing % | Synoptic Regime | BTW Computation |
|-------------------|-----------------|-----------------|
| 0% | Strong lift (PVA ahead of a trough, strong negative omega) | Average BTW across all levels 010-090 |
| 50% | Neutral (omega ≈ 0) | Weighted blend of average and max |
| 100% | Strong subsidence (NVA behind shortwave, strong positive omega) | MAX BTW in column |

### Depth Selection at 100%

- **Most locations**: MAX BTW in 010-050 (0-5 kft AGL)
- **Downslope locations** (KRNO/KRTS, Sierra NV): MAX BTW in 010-090 (0-9 kft AGL) — strong downslope flow taps deeper column

## Option A: Omega-Primary + Divergence Refinement

**Chosen approach.** Uses only point-profile data from NAM.

### Why omega is primary

Omega (Pa/s) is the *integrated result* of all QG forcing:
- Vorticity advection (positive = upward motion)
- Thermal advection (warm advection = upward motion)
- Friction and orographic forcing
- Diabatic heating effects

No need to reconstruct individual forcing terms from 2D gradients — omega already carries their combined signal.

### Why divergence refines

Divergence tells you *where in the column* the forcing concentrates:
- Classic subsidence: upper-level convergence (neg div) stacked over lower-level divergence (pos div)
- Sharper couplet → stronger downward momentum transfer
- Can weight the BTW depth selection based on which layers show strongest downward forcing

### What we extract from NAM GRIB2

| Parameter | paramId | Levels in GRIB2 | Extraction Status |
|-----------|---------|-----------------|-------------------|
| Omega (Pa/s) | 135 | 39 isobaric (1000-50 hPa) | ✅ Already extracted |
| Divergence (s⁻¹) | 155 | 5 levels (1000, 850, 700, 500, 250 hPa) | ❌ Needs new query |
| Absolute vorticity (s⁻¹) | 190 | 5 levels (1000, 850, 700, 500, 250 hPa) | ❌ Needs new query |

### Extractor additions (nam_profile_extractor.py)

```python
CFGRIB_QUERIES = {
    # ... existing queries ...
    "isobaric_div":   {"typeOfLevel": "isobaricInhPa", "paramId": 155},   # divergence (5 levels)
    "isobaric_absv":  {"typeOfLevel": "isobaricInhPa", "paramId": 190},   # absolute vorticity (5 levels)
}
```

Output JSON keys: `div_s1` and `absvort_s1` at each level where available (1000, 850, 700, 500, 250 hPa).

## Downward Forcing Function (Zero-Floor + Exponential)

Any upward motion (omega ≤ 0) → 0% downward forcing. No exceptions.
The scale only operates on positive omega (subsidence).

### Formula

```
if omega ≤ 0:
    DF% = 0
else:
    DF% = 100 × (1 − e^(−α × omega))
```

α controls how quickly the curve saturates toward 100%. Starting value: **α = 0.3**.

### Mapping at α = 0.3

| 500 hPa Omega (Pa/s) | DF% | Synoptic Interpretation |
|----------------------|-----|------------------------|
| ≤ 0 | 0% | Any lift or neutral — no downward forcing |
| +0.5 | 14% | Barely subsiding |
| +1.0 | 26% | Weak subsidence |
| +2.0 | 45% | Moderate subsidence |
| +3.0 | 59% | Behind moderate shortwave |
| +5.0 | 78% | Strong subsidence |
| +7.0 | 88% | Behind strong shortwave |
| +10.0 | 95% | Extreme subsidence |

### Why exponential over tanh

Original consideration was tanh (symmetric S-curve). Rejected because:
- Negative omega should map to exactly 0%, not a gradual decrease
- A symmetric curve gives identical weight to weak lift and weak subsidence (physically wrong)
- Exponential with zero floor correctly represents: *no subsidence = no downward forcing, period*

### Primary input level

**500 hPa** — where QG omega forcing is most representative of synoptic-scale vertical motion.
Alternative: column-mean omega weighted 850-300 hPa (more robust, but 500 hPa is standard diagnostics).

### Calibration Plan

α = 0.3 is a starting estimate. Calibrate by:

1. Log omega_500 + observed surface wind events across NAM cycles
2. Compare forecast DF% against actual strong wind events (hit rate)
3. Compare forecast DF% against null events (false alarm rate)
4. Adjust α if systematic bias appears (lower → more gradual, higher → saturates sooner)
5. Switch to piecewise-linear if data shows poor fit

**Action items:**
1. Implement the zero-floor exponential function in nam_profile_extractor.py
2. Log omega_500 values + observed surface wind events across multiple NAM cycles
3. Validate against downslope wind events at KRNO/KRTS (Sierra Nevada)
4. Validate against frontal passage events at KPDX, KDEN
5. Adjust α based on observed data

## Rejected Alternatives

### Option B: Vorticity Time-Tendency
Compare η(t+3h) − η(t) at a point → approximates NVA. Rejected because:
- 3-hour temporal resolution is too coarse
- Mixes true advection with local diabatic effects
- Omega already carries the vorticity-advection signal

### Option C: Grid Stencil Extraction
Extract 3×3 or 5×5 grid around each station, compute finite-difference gradients. Rejected because:
- 9-25× more data per station
- More complex extraction logic
- Omega + divergence give equivalent signal at point locations
- May revisit if NVA computation becomes critical

## Derived Fields (computable from existing profiles)

- **θe profiles**: from T, RH, P at each isobaric level → identifies elevated instability and moisture transport
- **Lapse rates**: already computing BTW 010 lapse (°C/km)
- **Lifted Index, SWEAT, Total Totals**: from T/RH profiles

## Related

- [[havoc]] — HAVOC agent definition
- [[vile-orchestrator]] — V.I.L.E. framework
- [[btw-computation]] — BTW calculation methodology