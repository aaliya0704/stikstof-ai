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
Atmospheric concentrations  
Atmospheric transport   
Deposition  
Environmental impact
```

## Natura 2000 Record-to-Site Validation

After loading and exploring the official Natura 2000 GeoPackage, the next step was to determine how individual spatial records relate to actual Natura 2000 sites.

This validation was necessary before creating any derived site-level dataset. A spatial record cannot automatically be assumed to represent one complete Natura 2000 site.

The analysis was divided into three phases:

1. Validate the record-to-site relationship
2. Analyze designation combinations
3. Investigate the spatial relationships between designation geometries

---


## Phase 1 — Validating the Record-to-Site Relationship

The `nr` attribute was investigated to determine whether it can be used to group multiple spatial records belonging to the same Natura 2000 site.

The number of spatial records associated with each `nr` was calculated using:

```python
site_record_counts = natura.groupby("nr").size().sort_values(ascending=False)

site_record_counts.head(20)
```

The result showed that several Natura 2000 sites have multiple spatial records.

The distribution of records per site was then calculated:

```python
site_record_counts.value_counts().sort_index()
```


### Result

```text
1    121
2     35
3      6
```

This means:

| Records associated with a site | Number of sites |
| -----------------------------: | --------------: |
|                              1 |             121 |
|                              2 |              35 |
|                              3 |               6 |

Therefore:

```text
121 + 35 + 6 = 162 unique sites
```

while the original dataset contains:

```text
209 spatial records
```

This establishes an important distinction:

```text
209 spatial records
        ↓
162 unique Natura 2000 sites
```

Therefore:

> **1 spatial record ≠ necessarily 1 Natura 2000 site**

Some sites are represented by multiple spatial records.

This distinction is important for **StikstofAI** because treating all 209 records as independent Natura 2000 sites could result in duplicate representation of the same real-world site during later spatial analysis or machine-learning feature engineering.

## Phase 2 — Understanding the Designation Combinations

After determining that some sites contain multiple records, the next question was:

> **Why does a single Natura 2000 site sometimes have multiple spatial records?**

The `beschermin` attribute was examined for each `nr`:

```python
designation_by_site = natura.groupby("nr")["beschermin"].apply(
    lambda x: ", ".join(sorted(x))
)

designation_by_site.head(20)
```

### Result

The first results were:

```text
nr
1     VR, VR+HR
2     HR, VR+HR
3     HR, VR+HR
4     HR, VR+HR
5     VR+HR
6     VR+HR
7     VR+HR
8     VR
9     VR+HR
10    HR, VR+HR
11    VR
12    VR
13    VR+HR
14    VR
15    VR, VR+HR
16    HR
17    HR
18    HR
19    VR
20    VR
```

The observed designation categories are:

* `VR` — Vogelrichtlijn / Birds Directive
* `HR` — Habitatrichtlijn / Habitats Directive
* `VR+HR` — associated with both Birds Directive and Habitats Directive

This shows that multiple records belonging to the same `nr` can have different designation categories.

For example:

```text
Site nr = 1
    │
    ├── VR
    └── VR+HR
```

Another site may have:

```text
Site nr = 2
    │
    ├── HR
    └── VR+HR
```

while another site may have only:

```text
Site nr = 5
    │
    └── VR+HR
```

Therefore, **multiple records are not necessarily multiple Natura 2000 sites**. They can represent different designation-related spatial records associated with the same site.

### Why This Matters

This means that immediately dissolving all records by `nr` could remove or obscure information contained in the original records.

For this reason, the raw dataset is being preserved in its original form.

If a site-level representation is required later, it will be created as a **derived dataset**, rather than modifying the original layer.

The exact aggregation strategy will be determined after understanding how the Natura 2000 layer needs to interact with nitrogen-deposition data.

## Phase 3 — Investigating the `intersects()` Result

During the earlier investigation of Hollands Diep, three spatial records were identified:

```text
VR
HR
VR+HR
```

Their geometries were extracted:

```python
vr = hollands[hollands["beschermin"] == "VR"].geometry.iloc[0]
hr = hollands[hollands["beschermin"] == "HR"].geometry.iloc[0]
vr_hr = hollands[hollands["beschermin"] == "VR+HR"].geometry.iloc[0]
```

Pairwise spatial relationships were then tested.

The geometries returned:

```python
vr.intersects(hr)
vr.intersects(vr_hr)
hr.intersects(vr_hr)
```

with all three returning:

```text
True
```

However, the corresponding intersection areas were:

```text
VR ∩ HR       = 0 m²
VR ∩ VR+HR    = 0 m²
HR ∩ VR+HR    = 0 m²
```

At first, this appears contradictory:

```text
intersects() → True
intersection area → 0 m²
```

However, these two operations answer different questions.

`intersects()` checks whether two geometries have any spatial contact.

It does **not** require them to share an area.

For example:

Two polygons can touch along a boundary without sharing any interior area.

To determine exactly what type of spatial contact was occurring, the geometry type of each intersection was inspected:

```python
print("VR ∩ HR geometry type:", vr.intersection(hr).geom_type)

print("VR ∩ VR+HR geometry type:", vr.intersection(vr_hr).geom_type)

print("HR ∩ VR+HR geometry type:", hr.intersection(vr_hr).geom_type)
```

The result was:

```text
VR ∩ HR geometry type: MultiLineString
VR ∩ VR+HR geometry type: MultiLineString
HR ∩ VR+HR geometry type: MultiLineString
```

A `MultiLineString` intersection indicates that the geometries share line-based spatial boundaries rather than a polygonal area.

This resolves the apparent contradiction between `intersects() == True` and an intersection area of `0 m²`.

---

## Key Findings from the Validation

The three phases produced several important conclusions about the Natura 2000 dataset.

### 1. A spatial record is not necessarily a complete Natura 2000 site

The dataset contains:

```text
209 spatial records
162 unique Natura 2000 sites
```

Therefore, the two concepts must be kept separate.

### 2. Some sites contain multiple spatial records

The observed distribution is:

```text
121 sites → 1 record
35 sites  → 2 records
6 sites   → 3 records
```

### 3. Multiple records can correspond to different designation categories

The `beschermin` attribute contains categories such as:

```text
VR
HR
VR+HR
```

These categories help explain why a single Natura 2000 site can have multiple spatial records.

### 4. Hollands Diep demonstrates boundary-based spatial contact

The three Hollands Diep designation geometries:

```text
VR
HR
VR+HR
```

all intersect according to Shapely's `intersects()` predicate.

However, their pairwise intersection areas are zero and their intersection geometries are `MultiLineString`.

This indicates spatial contact along boundaries rather than overlapping polygon interiors.
