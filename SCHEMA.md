# Wiki Schema

## Domain
Forecast approaches in meteorology — numerical weather prediction (NWP), statistical forecasting, ensemble methods, post-processing techniques, and forecast verification.

## Conventions
- File names: lowercase, hyphens, no spaces (e.g., `ensemble-perturbation.md`)
- Every wiki page starts with YAML frontmatter (see below)
- Use `[[wikilinks]]` to link between pages (minimum 2 outbound links per page)
- When updating a page, always bump the `updated` date
- Every new page must be added to `index.md` under the correct section
- Every action must be appended to `log.md`
- Use standard meteorological abbreviations: NWP, GFS, ECMWF, WRF,ENS, etc.

## Frontmatter
```yaml
---
title: Page Title
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: entity | concept | comparison | query | summary
tags: [from taxonomy below]
sources: [raw/articles/source-name.md]
---
```

## Tag Taxonomy
### Forecast Types
- nwp, ensemble, statistical, machine-learning, climate-forecast

### Forecast Components
- initial-condition, boundary-condition, parameterization, physics, model-resolution

### Data Sources
- gfs, ecmwf, wrf, mesoscale, obs-data, reanalysis

### Techniques
- ensemble-perturbation, model-averaging, post-processing, downscaling, bias-correction

### Verification
- validation, verification, statistical-score, error-analysis, verification-system

### Meta
- comparison, history, review, controversy, guideline

## Page Thresholds
- **Create a page** when a concept appears in 2+ sources OR is central to one source
- **Add to existing page** when a source mentions something already covered
- **DON'T create a page** for passing mentions or minor details outside the domain
- **Split a page** when it exceeds ~200 lines — break into sub-topics with cross-links
- **Archive a page** when content is fully superseded — move to `_archive/`, remove from index

## Entity Pages
One page per notable model/approach:
- Overview / what it is
- Key developers/institutions
- Key parameters and configuration
- Strengths and limitations
- Relationships to other approaches ([[wikilinks]])
- Source references

## Concept Pages
One page per concept or technique:
- Definition / explanation
- Mathematical basis (if applicable)
- Current state of knowledge
- Open questions or debates
- Related concepts ([[wikilinks]])

## Update Policy
When new information conflicts with existing content:
1. Check the dates — newer sources generally supersede older ones
2. If genuinely contradictory, note both positions with dates and sources
3. Mark the contradiction in frontmatter: `contradictions: [page-name]`
4. Flag for user review in the lint report
