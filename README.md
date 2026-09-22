# StikstofAI

**StikstofAI** is an experimental geospatial machine-learning project focused on exploring nitrogen-related environmental risk in the Netherlands.

The project aims to combine geospatial environmental data, nitrogen deposition data, emission-related variables, atmospheric observations, and other relevant factors to investigate patterns of nitrogen deposition around protected Natura 2000 areas.

> **Project status:** Early-stage data exploration and geospatial preprocessing.

StikstofAI is an experimental research and decision-support project. It is **not intended to replace official Dutch nitrogen assessment, permitting, or regulatory systems such as AERIUS**.

---

# Project Motivation

The Netherlands faces significant challenges related to nitrogen emissions and nitrogen deposition, particularly around protected nature areas.

Nutrients and nitrogen compounds such as nitrogen oxides (NOx) and ammonia (NH3) can contribute to nitrogen deposition in ecosystems. Excessive deposition can affect sensitive habitats and species.

StikstofAI explores whether geospatial data and machine-learning techniques can be used to:

- Analyze historical nitrogen deposition patterns.
- Investigate relationships between nitrogen deposition and potential emission sources.
- Identify geographical patterns around Natura 2000 protected areas.
- Estimate nitrogen-related environmental risk.
- Explore hypothetical emission-reduction scenarios.
- Provide an interactive interface for investigating these patterns.

The project is being developed progressively, beginning with understanding and preparing the underlying geospatial datasets before introducing machine-learning models.

---

# Important Scientific Distinction

A central concept in StikstofAI is the distinction between:

```text
Emissions
    ↓
Atmospheric concentrations
    ↓
Atmospheric transport
    ↓
Deposition
    ↓
Environmental impact

## Natura 2000 Record-to-Site Validation

After loading and exploring the official Natura 2000 GeoPackage, the next step was to determine how individual spatial records relate to actual Natura 2000 sites.

This validation was necessary before creating any derived site-level dataset. A spatial record cannot automatically be assumed to represent one complete Natura 2000 site.

The analysis was divided into three phases:

1. Validate the record-to-site relationship
2. Analyze designation combinations
3. Investigate the spatial relationships between designation geometries

---

### Phase 1 — Validate the Record-to-Site Relationship

The `nr` attribute was investigated to determine whether it can be used to group multiple spatial records belonging to the same Natura 2000 site.

The number of spatial records associated with each `nr` was calculated using:

```python
site_record_counts = (
    natura.groupby("nr")
    .size()
    .sort_values(ascending=False)
)

site_record_counts.head(20)