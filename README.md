# StikstofAI

**StikstofAI** is an experimental geospatial machine-learning project focused on exploring nitrogen-related environmental risk in the Netherlands.

The project aims to combine geospatial environmental data, nitrogen deposition data, emission-related variables, atmospheric observations, and other relevant factors to investigate patterns of nitrogen deposition around protected Natura 2000 areas.

> **Project status:** *Phase 5 — historical deposition analysis completed so far; site-level integration is next.*

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

1    121

2     35

3      6

```

This means:

| Records associated with a site | Number of sites |

| -----------------------------: | --------------: |

|                              1 |             121 |

|                              2 |              35 |

|                              3 |               6 |

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

1     VR, VR+HR

2     HR, VR+HR

3     HR, VR+HR

4     HR, VR+HR

5     VR+HR

6     VR+HR

7     VR+HR

8     VR

9     VR+HR

10    HR, VR+HR

11    VR

12    VR

13    VR+HR

14    VR

15    VR, VR+HR

16    HR

17    HR

18    HR

19    VR

20    VR

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

VR ∩ HR       = 0 m²

VR ∩ VR+HR    = 0 m²

HR ∩ VR+HR    = 0 m²

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

35 sites  → 2 records

6 sites   → 3 records

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

## Phase 4 — Construct Site-Level Natura 2000 Dataset

### Objective

The original Natura 2000 dataset contains multiple spatial records for some protected areas. Therefore, the original dataset cannot be directly treated as a one-row-per-site dataset.

The objective of this phase was to construct a clean **site-level Natura 2000 dataset** containing one spatial record for each unique Natura 2000 site.

The resulting dataset will provide the spatial foundation for connecting Natura 2000 areas with nitrogen deposition and other environmental variables in later phases.

---

### 4.1 Why a Site-Level Dataset Was Needed

The original PDOK Natura 2000 dataset contains:

* **209 spatial records**

* **162 unique Natura 2000 sites**

This means that a spatial record does not necessarily represent one complete real-world Natura 2000 site.

Some sites occur across multiple records because different spatial records represent different designation categories.

For example, **Hollands Diep** contains three spatial records:

* `VR` — Birds Directive

* `HR` — Habitats Directive

* `VR+HR` — Birds + Habitats Directive

The three records share the same Natura 2000 site number but contain separate geometries and designation information.

This structure was investigated in Phase 1–3 before constructing the site-level representation.

---

### 4.2 Site-Level Representation

The site-level dataset was constructed using the Natura 2000 site number (`nr`) as the grouping key.

The geometry was dissolved by `nr` so that all spatial records belonging to the same Natura 2000 site were combined into a single geometry.

Site attributes were aggregated separately rather than relying on a generic dissolve operation. This was necessary because directly dissolving the complete dataset could discard or obscure information contained in fields such as designation type and site codes.

The resulting dataset contains:

| Column              | Description                                     |

| ------------------- | ----------------------------------------------- |

| `nr`                | Natura 2000 site number                         |

| `naam_n2k`          | Natura 2000 site name                           |

| `designation_types` | Designation categories associated with the site |

| `geometry`          | Dissolved site-level geometry                   |

| `site_area_m2`      | Calculated site geometry area in square metres  |

| `site_area_ha`      | Calculated site geometry area in hectares       |

---

### 4.3 Geometry Construction

The geometry was constructed separately from the attribute aggregation:

```python
natura_site = natura[["nr", "geometry"]].dissolve(by="nr", as_index=False)
```

This produces one geometry for each unique Natura 2000 site number.

The site names were then added separately:

```python
site_names = natura.groupby("nr")["naam_n2k"].first().reset_index()
```

This was considered appropriate because the previous validation showed that every Natura 2000 site number corresponds to exactly one unique site name.

Designation information was also aggregated explicitly:

```python

designation_by_site = (

    natura.groupby("nr")["beschermin"]

    .apply(lambda x: ", ".join(sorted(x.unique())))

    .reset_index(name="designation_types")

)

```

The resulting tables were then merged into the site-level GeoDataFrame.

---

### 4.4 Site Area Calculation

Because the dataset uses the projected CRS **EPSG:28992 (Amersfoort / RD New)**, the geometry coordinates are measured in metres.

Therefore, geometry area can be calculated directly in square metres:

```python
natura_site["site_area_m2"] = natura_site.geometry.area
```

The area was also converted to hectares:

```python
natura_site["site_area_ha"] = natura_site["site_area_m2"] / 10_000
```

These are **areas calculated from the derived site-level geometries**. They should not automatically be interpreted as official published area values.

---

### 4.5 Final Validation

The completed site-level dataset was validated before proceeding to the next phase.

The following checks were performed:

```python
print("Number of site records:", len(natura_site))

print("Unique site IDs:", natura_site["nr"].nunique())

print("Duplicate site IDs:", natura_site["nr"].duplicated().sum())

print("Missing site names:", natura_site["naam_n2k"].isna().sum())

print("Missing designation types:", natura_site["designation_types"].isna().sum())

print("Missing geometries:", natura_site.geometry.isna().sum())

print("Invalid geometries:", (~natura_site.geometry.is_valid).sum())

print("Sites with area <= 0:", (natura_site["site_area_m2"] <= 0).sum())

print("CRS:", natura_site.crs)

print(natura_site.geometry.geom_type.value_counts())
```

The validation produced the expected results:

* **162 site records**

* **162 unique Natura 2000 site IDs**

* **0 duplicate site IDs**

* **0 missing site names**

* **0 missing designation types**

* **0 missing geometries**

* **0 invalid geometries**

* **0 sites with non-positive area**

* CRS: **EPSG:28992**

* Valid polygonal site geometries

This confirms that the site-level representation is structurally consistent and ready for integration with additional spatial datasets.

---

### 4.6 Result

Phase 4 transformed the original record-level Natura 2000 dataset into a clean site-level spatial dataset.

The transformation can be summarized as:

```text

Original PDOK dataset

        │

        ├── 209 spatial records

        │

        ├── Multiple records can belong to one site

        │

        └── Different designation categories

                    │

                    ▼

          Group by Natura 2000 `nr`

                    │

                    ▼

            Dissolve geometries

                    │

                    ▼

        Aggregate site attributes

                    │

                    ▼

       Calculate site-level areas

                    │

                    ▼

        Validate resulting dataset

                    │

                    ▼

          162 Natura 2000 sites

```

The resulting `natura_site` GeoDataFrame will serve as the **site-level spatial foundation** for subsequent nitrogen-deposition analysis.

---

---

# Phase 5 — RIVM Historical Nitrogen Deposition Analysis

## Objective

Phase 5 introduces the RIVM historical nitrogen deposition dataset and examines how modeled deposition values vary across the Netherlands over time.

The analysis uses the RIVM historical series for 2005–2024 and the existing Natura 2000 site-level dataset as preparation for future spatial integration.

**Important:** The RIVM grid represents modeled large-scale deposition estimates. These results are exploratory analyses of the supplied grid data, not direct measurements at every grid cell and not official local-scale compliance assessments. StikstofAI does not replace AERIUS or other official assessment systems.

## 5.1 Data Source and Structure

The downloaded GeoPackage is stored locally at:

```text
data/raw/RIVM-MIL_HistorischeReeksStikstofdepositie_20251007.gpkg
```

Raw data is kept outside version control through the repository's `.gitignore` configuration.

The GeoPackage contains 20 yearly layers:

```text
ndep_historisch_2005
ndep_historisch_2006
...
ndep_historisch_2024
```

Each layer contains the following fields:

| Column | Description |
|---|---|
| `gdn_grid_cell_id` | Identifier for an RIVM deposition grid cell |
| `substance_name` | Substance represented by the layer |
| `year` | Year associated with the deposition estimate |
| `deposition` | Modeled nitrogen deposition value |
| `geometry` | Polygon geometry of the grid cell |

The inspected layers use **EPSG:28992 (Amersfoort / RD New)**, the same projected coordinate reference system as the Natura 2000 dataset. Each grid cell is a 1 km × 1 km polygon, with an area of 1,000,000 m².

The inspected `substance_name` value is `Totaal stikstof (N)`.

The RIVM product describes a large-scale GDN grid and is not intended to provide the detailed local-scale deposition picture used for official site-specific assessment. Deposition units should be confirmed against the official layer metadata before being hard-coded into final labels or calculations.

## 5.2 Validation of the Yearly Layers

Before combining the yearly layers, the data was checked for structural consistency.

The checks confirmed that:

- The GeoPackage contains the expected 20 annual layers for 2005–2024.
- Each yearly layer contains 43,959 grid cells.
- The `year` field agrees with the year indicated by the layer name.
- The inspected layers identify the substance as `Totaal stikstof (N)`.
- All yearly layers contain the same set of `gdn_grid_cell_id` values.
- The 2005 and 2024 grid geometries were found to be identical in the comparison performed.
- The inspected 2005 layer contained no missing deposition values, no duplicate grid-cell IDs, and no invalid geometries.

These checks support comparisons between years using stable grid-cell identifiers and consistent spatial boundaries.

## 5.3 Combined Historical Dataset

The 20 annual layers were combined into a single historical GeoDataFrame named `rivm_historical`.

The resulting dataset contains:

| Property | Result |
|---|---:|
| Years covered | 2005–2024 |
| Annual layers | 20 |
| Grid cells per year | 43,959 |
| Total rows | 879,180 |
| Unique grid-cell IDs | 43,959 |
| CRS | EPSG:28992 |

Validation confirmed that:

- There are no duplicate `(gdn_grid_cell_id, year)` combinations.
- Each grid cell has one record for each of the 20 years.
- There are no missing deposition values in the combined dataset.
- No invalid geometries were found in the combined dataset.

The combined data therefore provides a consistent grid-cell-year structure for historical analysis.

## 5.4 National Grid-Level Trend Analysis

Yearly summary statistics were calculated to examine how the distribution of modeled deposition values changed over time. The statistics included the mean, median, minimum, maximum, and standard deviation for each year.

The mean modeled deposition value across grid cells was:

| Year | Mean deposition value |
|---:|---:|
| 2005 | 1,702.63 |
| 2022 | 1,106.94 |
| 2024 | 1,244.71 |

Between 2005 and 2024, the mean changed by approximately `-457.92`, or **-26.90%** relative to the 2005 mean.

The mean decreased between 2005 and 2022, then increased between 2022 and 2024. The series also contains year-to-year variation, so the trend should not be interpreted as a smooth or uninterrupted decline.

These are arithmetic summaries of the grid-cell values. They are not an official national deposition inventory or a direct calculation of total nitrogen mass deposited.

## 5.5 Spatial Inspection of Selected Years

The 2005, 2022, and 2024 layers were mapped to inspect the spatial distribution of deposition values.

The layers use the same grid-cell IDs and geometries, making spatial comparison possible. However, maps of absolute deposition values should use a consistent colour scale when they are compared visually; otherwise, independently scaled maps can make differences appear larger or smaller than they are.

The highest-value cells were also inspected for each selected year. Several cells appeared among the highest-value cells in multiple years, indicating that high modeled values were spatially concentrated in similar parts of the grid across the selected years. This observation alone does not establish the cause of those values.

## 5.6 Cell-Level Comparison Between 2005, 2022, and 2024

A cell-level comparison GeoDataFrame named `cell_change` was constructed by aligning the selected-year deposition values using `gdn_grid_cell_id`.

The comparison includes:

| Column | Description |
|---|---|
| `gdn_grid_cell_id` | Stable grid-cell identifier |
| `deposition_2005` | Deposition value for 2005 |
| `deposition_2022` | Deposition value for 2022 |
| `deposition_2024` | Deposition value for 2024 |
| `change_2005_2022` | Absolute change from 2005 to 2022 |
| `change_2005_2024` | Absolute change from 2005 to 2024 |
| `percent_change_2005_2022` | Percentage change from 2005 to 2022 |
| `percent_change_2005_2024` | Percentage change from 2005 to 2024 |
| `change_2022_2024` | Absolute change from 2022 to 2024 |
| `geometry` | Grid-cell polygon |

Absolute change is calculated as the later value minus the earlier value:

```python
change = later_deposition - earlier_deposition
```

Percentage change is calculated relative to the earlier value:

```python
percent_change = ((later_deposition - earlier_deposition) / earlier_deposition) * 100
```

A negative change indicates a decrease, a positive change indicates an increase, and zero indicates no change between the selected endpoints.

Validation confirmed that the comparison dataset contains 43,959 unique grid cells, with no duplicate grid-cell IDs and no missing values in the calculated change columns.

## 5.7 Results of Cell-Level Change Analysis

### 2005–2022

| Change category | Grid cells | Share of grid cells |
|---|---:|---:|
| Decreased | 43,957 | 99.9955% |
| Increased | 2 | 0.0045% |
| Unchanged | 0 | 0.0000% |

The mean absolute change across grid cells was approximately `-595.69`. The largest decrease was `-4,016`, while the largest increase was `+12`.

### 2005–2024

| Change category | Grid cells | Share of grid cells |
|---|---:|---:|
| Decreased | 43,880 | 99.8203% |
| Increased | 78 | 0.1774% |
| Unchanged | 1 | 0.0023% |

The mean absolute change across grid cells was approximately `-457.92`. The largest decrease was `-3,602`, while the largest increase was `+611`.

The ten cells with the highest deposition values in 2005 were all below their 2005 values in both 2022 and 2024. Some of these cells had higher values in 2024 than in 2022, despite remaining below their 2005 levels.

### 2022–2024

| Change category | Grid cells | Share of grid cells |
|---|---:|---:|
| Increased | 43,648 | 99.2933% |
| Decreased | 293 | 0.6665% |
| Unchanged | 18 | 0.0409% |

The mean grid-cell value increased from approximately `1,106.94` in 2022 to `1,244.71` in 2024. This corresponds to an increase of approximately 12.45% relative to the 2022 mean.

These results describe differences between selected annual modeled values. They do not, by themselves, establish why deposition changed or whether the change persisted beyond 2024.

## 5.8 Mapping Cell-Level Changes

Three change maps were created using the `cell_change` GeoDataFrame:

1. **2005–2022:** change relative to the 2005 baseline.
2. **2005–2024:** change relative to the 2005 baseline.
3. **2022–2024:** change between the two more recent years.

The maps use a diverging colour scale centred on zero:

- Blue represents negative changes.
- White or a light colour represents values near zero.
- Red represents positive changes.

The 2005–2022 and 2005–2024 maps use the same colour limits, allowing the magnitudes of change to be compared visually. The 2022–2024 map uses a separate scale centred on zero because its range of changes differs. Consequently, colour intensity should not be compared directly between that map and the first two maps.

### Spatial findings

- The 2005–2022 comparison shows decreases in 43,957 of 43,959 grid cells. The map displays these changes using varying shades of blue, with cells whose changes are closer to zero appearing pale blue or nearly white.
- The 2005–2024 comparison shows decreases in 43,880 grid cells. Its spatial pattern broadly resembles the 2005–2022 comparison, although the magnitude of change varies across the Netherlands.
- The 2022–2024 comparison shows increases in 43,648 grid cells. The map displays varying shades of red for positive changes, while cells with smaller changes may appear close to white.
- These maps show the direction and magnitude of change in modeled deposition. The numerical proportions of increasing and decreasing cells should not be confused with the visible proportion of blue, white, or red in the maps.

The map validation checks confirmed:

- 43,959 grid cells.
- No missing geometries.
- No missing values in the three change columns.
- CRS: EPSG:28992.
- No invalid geometries.

## 5.9 Scientific Interpretation and Limitations

The Phase 5 results are exploratory and should be interpreted within the limits of the source data.

- The deposition values are modeled estimates, not direct measurements at every grid cell.
- The RIVM grid represents large-scale deposition patterns and is not a substitute for official local-scale assessment.
- A decrease relative to 2005 does not establish that a cell or protected site is below an ecological critical deposition threshold.
- A widespread increase between 2022 and 2024 does not establish its cause or prove a persistent long-term reversal.
- Counting cells that increased or decreased is not the same as calculating a change in total nitrogen deposition mass.
- Aggregating grid values to Natura 2000 sites will require explicit treatment of cells that only partially intersect a site.
- Deposition, emissions, atmospheric concentrations, and ecological impacts are distinct concepts and should not be treated as interchangeable.

## Phase 5 Status

**Completed so far:** RIVM historical layers were inspected and validated; the annual layers were combined; yearly statistics and selected-year maps were produced; cell-level changes were calculated and validated; and three maps of spatial changes were generated.

**Next:** Phase 5 continues with integrating the RIVM deposition grid and the 162-site Natura 2000 dataset, including careful handling of partial grid-cell intersections, before preparing site-level deposition summaries.
