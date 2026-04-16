---
title: H.A.V.O.C. - Mesocyclone Analysis & Updraft Locator
created: 2026-04-09
updated: 2026-04-15
type: entity
tags: [agent, non-convective wind, parameterization]
sources: []
---

# H.A.V.O.C. - Specialized sub-agent for non-convective wind analysis in the V.I.L.E. framework.

Specialized agent for non-convective wind analysis within the [[VILE Orchestrator|vile-orchestrator]] framework.

## Acronym
**H.A.V.O.C.** - High-Altitude Velocity & Orographic Calculator

## Focus Parameters

- **Omega** (Pa/s): Primary vertical motion indicator — integrated result of all QG forcing
- **Divergence** (s⁻¹): Refines WHERE in the column forcing concentrates (5 standard levels)
- **Absolute vorticity** (s⁻¹): NVA identification (5 standard levels)
- **BTW** (Best Transfer Winds): Bunkers-style interpolated wind profiles by AGL layer

## Downward Forcing Scale (0-100%)

Controls BTW depth and weighting. See [[havoc-downward-forcing]] for full methodology.

| Forcing % | Regime | BTW Method |
|-----------|--------|------------|
| 0% | Strong lift | Average BTW 010-090 |
| 100% | Strong subsidence/NVA | MAX BTW 010-050 (010-090 for downslope) |

Omega is primary metric. Divergence refines layer weighting.

## Data Dependency

BTW computation requires U and V wind components extracted from NAM GRIB2 via `nam_profile_extractor.py`. A known cfgrib bug (see [[btw-computation]]) silently drops wind data when u and v are opened together (`paramId=[131,132]`). They must be extracted as separate datasets. If BTW values show `None`, verify the extraction pipeline.

## Pipeline

HAVOC parameters flow through the [[nam-data-pipeline]] — GRIB2 download → cfgrib extraction → station profiles → BTW computation → threats grid display. The cfgrib u/v split bug (see [[btw-computation]]) broke this pipeline at the extraction stage.

## Reference
- Source: /home/progged-ish/projects/forecast_approach/agents/havoc_agent.py

[[VILE Orchestrator|vile-orchestrator]]
[[VILE Sub-Agents|vile-sub-agents]]
[[nam-data-pipeline]]
