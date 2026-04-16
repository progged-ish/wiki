---
title: V.I.L.E. Orchestrator
created: 2026-04-09
updated: 2026-04-14
type: entity
tags: [orchestrator, framework]
sources: []
---

# V.I.L.E. Orchestrator

V.I.L.E. (Villaran's Inference & Logic Engine) Master Orchestrator - the central coordinator for the multi-agent weather threat analysis framework.

## Overview

V.I.L.E. orchestrates 7 specialized sub-agents, each focused on a specific weather threat type. The orchestrator handles:

- **Data triage and routing** to appropriate sub-agents
- **Sub-agent coordination and monitoring**
- **Confidence scoring and uncertainty quantification**
- **Output synthesis and report generation**
- **Conflict resolution between agents**

## Architecture

### Request Structure
VILEAnalysisRequest with:
- `location`: Location identifier (station ID, coordinates)
- `timestamp`: Analysis time (ISO format)
- `threat_types`: List of threats to analyze (None = all)
- `data_sources`: Available data sources for analysis
- `confidence_threshold`: Minimum confidence (default: 0.3)
- `time_window_hr`: Analysis time window (default: 6 hours)

### Output Structure
VILEAnalysisResult containing:
- `request`: Original analysis request
- `threat_assessments`: List of individual agent assessments
- `synthesized_report`: Consolidated report
- `overall_confidence`: Aggregate confidence score (0.0-1.0)
- `processing_time_ms`: Execution time
- `status`: success/partial/failed

## Sub-Agents

| Agent | Domain | Parameters |
|-------|--------|------------|
| [[maul]] | Tornadogenesis | LCL height, SRH, bulk shear, STP |
| [[blast]] | Convective Wind | DCAPE, dry air, delta-theta-e, lapse rates |
| [[havoc]] | Non-Convective Wind | Pressure gradients, isallobaric forcing, LLJs |
| [[blight]] | Heavy Snow | DGZ, frontogenesis, EPV, warm nose |
| [[drown]] | Flash Flood | Rainfall rates, antecedent moisture, thresholds |
| [[smash]] | Severe Hail | CAPE, updraft, graupel, radar signatures |
| [[glaze]] | Freezing Precip | Temp profiles, freezing level, accretion |

## Confidence Calculation

The overall confidence is a weighted blend:
- **70%** weight on maximum individual confidence
- **30%** weight on average confidence
- **10% bonus** for multiple high-confidence threats (>0.7 each)

## Threat Mapping

Threat type strings are mapped to agent keys:
- tornado → MAUL
- convective_wind → BLAST
- non_convective_wind → HAVOC
- heavy_snow → BLIGHT
- flash_flood → DROWN
- hail → SMASH
- freezing_precip → GLAZE

## CRON Jobs

Scheduled tasks on the Hermes default profile that feed V.I.L.E. and related systems:

| Job ID | Name | Schedule | Script | Description |
|--------|------|----------|--------|-------------|
| `01a4fb6bfa4a` | NAM GRIB2 Download | `0 3,9,15,21 * * *` | `/home/progged-ish/metar-automation/scripts/...` | Downloads NAM 12km awphys GRIB2 from AWS S3 (noaa-nam-pds), 4x daily at 03/09/15/21 PDT. Feeds V.I.L.E. model data ingestion layer. |
| `23d13a32cc59` | Shane's NWS Scraper v9 | `0 5 * * *` | `cd /home/progged-ish/projects/shanes-scraper && shanes_nws_scraper_v9.py` | Scrapes all 138 CONUS WFO AFDs daily at 0500 PDT (1200 UTC). Dual-model AI summarization: qwen2.5-7b (LM Studio) per-office + gemini-2.5-flash for regional/CONUS synthesis. Outputs HUD-style HTML with keyword navigation. |

### Cleaned Up (2026-04-14)
The following zombie crons from the Hermes **coder** profile were removed — they were hallucinated by a weak local LLM and never executed once:

| Name | Schedule | Notes |
|------|----------|-------|
| `vile-dashboard-protector` | `*/3 * * * *` | Dashboard keepalive — never ran, overlaps with actual NAM pipeline |
| `nam-retrieval-automation` | `0 */3 * * *` | NAM GRIB2 downloader — duplicate of default profile job |
| `automatic-session-logger` | `0 9-18 * * *` | Session logging — never ran, no functional output |

## Reference
- Source: `/home/progged-ish/projects/forecast_approach/vile_orchestrator.py`

[[VILE Sub-Agents|vile-sub-agents]] [[Forecast Approaches|forecast-approaches]]
