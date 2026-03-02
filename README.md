# Bivariate Choropleth of Household Burdens

*From univariate poverty map to bivariate housing × transport choropleth with eviction overlay*

A single-file browser application that maps housing cost burden against transportation cost burden at the census tract and county level across all 50 states and DC. An optional eviction rate overlay adds a third dimension via SVG hatching.

**Data sources:** Census ACS 2022 (5-year) · HUD Location Affordability Index v3 · Eviction Lab 2000–2018  
**License:** [PolyForm Noncommercial License 1.0.0](https://polyformproject.org/licenses/noncommercial/1.0.0)

---

## Table of Contents

- [Quick Start](#quick-start)
- [Part I: Project Timeline](#part-i-project-timeline)
- [Part II: Analytical Decisions](#part-ii-analytical-decisions)
  - [2.1 Housing Burden: Why Not Poverty Rate?](#21-housing-burden-why-not-poverty-rate)
  - [2.2 Bracket-Counting Housing Burden](#22-bracket-counting-housing-burden)
  - [2.3 Transportation Cost: The HUD LAI](#23-transportation-cost-the-hud-lai)
  - [2.4 Adaptive Household Profile Selection](#24-adaptive-household-profile-selection)
  - [2.5 Color Scheme: Continuous OKLAB Interpolation](#25-color-scheme-continuous-oklab-interpolation)
  - [2.6 Dynamic Color Ranges](#26-dynamic-color-ranges)
  - [2.7 Eviction Rate: Metric Choice and Normalization](#27-eviction-rate-metric-choice-and-normalization)
  - [2.8 Sparse Tract Filtering](#28-sparse-tract-filtering)
  - [2.9 Geographic Boundaries and the MAUP](#29-geographic-boundaries-and-the-maup)
- [Part III: Engineering Log](#part-iii-engineering-log)
  - [3.1 CSV Parsing](#31-csv-parsing)
  - [3.2 ArcGIS Pagination](#32-arcgis-pagination)
  - [3.3 Connecticut's Planning Regions](#33-connecticuts-planning-regions)
  - [3.4 JavaScript Timing and Async State](#34-javascript-timing-and-async-state)
- [Part IV: Application Architecture](#part-iv-application-architecture)
  - [4.1 High-Level Structure](#41-high-level-structure)
  - [4.2 Data Sources and API Details](#42-data-sources-and-api-details)
  - [4.3 Key Global State Variables](#43-key-global-state-variables)
  - [4.4 Limitations and Future Directions](#44-limitations-and-future-directions)
  - [4.5 Development Process](#45-development-process)
- [Part V: Licensing](#part-v-licensing)

---

## Quick Start

1. Open `bivariate_cost_map_hud_eviction.html` in any modern browser — no build step, no server required.
2. Get a free [Census API key](https://api.census.gov/data/key_signup.html) and paste it into the key field. HUD LAI data requires no key.
3. Select a state and click **Load County Data**. Click any county to drill down to tract level.
4. Toggle the eviction rate overlay with the switch in the sidebar (available for ~45 states).

---

## Part I: Project Timeline

| Phase | Milestone | Key Decisions & Changes |
|-------|-----------|------------------------|
| 1 | Univariate DC poverty map | **Metric:** Poverty rate (B17001). **Geography:** Census tracts, DC only. First version used server-side MCP calls; hit CORS restrictions when ported to browser, requiring switch to direct Census API with manual key entry. |
| 2 | Maryland expansion + county drill-down | **Multi-level geography:** County overview with click-through to tract detail. County names drawn from Census `NAME` field. **Boundary service:** TIGERweb discovered as the canonical Census-hosted GeoJSON endpoint; legacy tigerWMS endpoints retired. |
| 3 | Multi-state selector | **State-level navigation:** Dropdown with all 50 states + DC. Census API key moved to user-supplied password input (CORS prevents server-side proxying). **Sparse tract filtering** introduced: tracts with <100 residents or <50 housing units rendered transparent. |
| 4 | Bivariate housing burden axis | **Metric change:** Poverty rate replaced by housing cost burden. **Renter/owner merge:** B25070 and B25091 combined via tenure-weighted mean. **Thresholds:** HUD definitions adopted (<20% affordable, 20–30% burdened, >30% severely burdened). |
| 5 | Childcare cost axis (exploratory) | **First bivariate attempt:** Housing burden × childcare cost. Data from DOL National Database of Childcare Prices. Later superseded by transportation cost — childcare costs are county-level only and zero in many counties. |
| 6 | HUD LAI transportation cost axis | **Metric:** HUD Location Affordability Index v3 — modeled transportation spending as % of household income. **Adaptive profiles:** LAI provides eight pre-computed household profiles per tract; application matches each tract to its best-fit profile. **LAI pagination bug:** State-wide ArcGIS query silently truncated at `maxRecordCount`; fixed by per-county fetching with offset pagination. |
| 7 | School catchment / boundary exploration | **Geographic alternative:** NCES School Attendance Boundary Survey explored as an activity-based boundary system. Maryland: 99.4% coverage; DC: 100%. **MAUP discussion:** Census tracts designed for statistical consistency (~4,000 people each), not community representation. Not implemented but documented as a future direction. |
| 8 | Eviction Lab overlay — initial | **Design choice:** SVG hatching rather than a third color axis. Three hatch densities: sparse diagonal (5–10%), medium (10–20%), dense crosshatch (>20%). Data: Eviction Lab per-state S3 CSVs. |
| 9 | Eviction overlay — national dataset migration | **Data source switch:** Migrated to Eviction Lab national proprietary CSVs. Two files fetched once and cached globally. **Coverage logic:** States enabled based on ≥67% tract or county coverage threshold. States with thin county coverage aggregate from tracts using population-weighted mean. |
| 10 | Connecticut planning region fixes | **Three-way boundary mismatch:** CT 2022 ACS uses planning region FIPS (110–190); HUD LAI uses legacy county FIPS (001–015); Eviction Lab uses legacy county prefixes in tract GEOIDs. Three separate workarounds implemented — see [Section 3.3](#33-connecticuts-planning-regions). |
| 11 | Timing and async fixes | Hatch layer render-before-data bug fixed. Background Census fetch of all-county tract population data added for population-weighted aggregation. Toggle persistence across state changes implemented. |
| 12 | Bracket-counting housing burden | **Metric overhaul:** Replaced tenure-weighted median percentages (B25071/B25092) with direct household counts from B25070/B25091 bracket tables. New metric: share of cost-computable households spending 30%+ of income on housing. Typical values shift from 20–30% range to 25–50%. |
| 13 | Continuous color + dynamic ranges | **OKLAB interpolation:** Replaced 3×3 hard-bucket assignment with bilinear interpolation across nine anchor colors in OKLAB perceptual color space, rendering the legend as a smooth 120×120 canvas gradient. **Dynamic ranges:** Color scale adapts per view using 10th–90th percentiles of the loaded data. Color palette refined across multiple iterations — see [Section 2.5](#25-color-scheme-continuous-oklab-interpolation). |

---

## Part II: Analytical Decisions

### 2.1 Housing Burden: Why Not Poverty Rate?

The project began with poverty rate (Census B17001) as the primary measure of economic stress. Poverty rate counts the share of residents with household income below the federal poverty line, which is set nationally without geographic cost-of-living adjustment. This makes it a poor measure of housing affordability: a family earning ×2 the federal poverty line in San Francisco faces more housing stress than a family earning the same amount in rural Mississippi, but poverty rate treats them identically.

Housing cost burden — the share of income spent on housing — directly captures the affordability squeeze at the household level regardless of absolute income. HUD defines three tiers: below 20% (affordable), 20–30% (cost-burdened), above 30% (severely cost-burdened). These thresholds are used in federal housing policy and are well-understood reference points for practitioners.

### 2.2 Bracket-Counting Housing Burden

The application computes housing burden by counting burdened households directly from the ACS bracket tables rather than using median percentages:

```
burden = (renter_burdened + owner_burdened) / (renter_computable + owner_computable)
```

*Burdened* means spending ≥30% of income on housing (the HUD severe cost-burden threshold). *Computable* excludes households with zero or negative income (recorded as "not computed" in both tables). Variables used:

| Variable | Description |
|----------|-------------|
| `B25070_007–010E` | Renter households in the 30–34.9%, 35–39.9%, 40–49.9%, and 50%+ brackets |
| `B25070_001E` | Total renters |
| `B25070_011E` | Renters: not computed |
| `B25091_008–011E` | Owners with mortgage, 30%+ brackets |
| `B25091_015–019E` | Owners without mortgage, 30%+ brackets |
| `B25091_001E` | Total owners |
| `B25091_012E` + `B25091_020E` | Owners: not computed (mortgage and non-mortgage) |

This approach correctly handles bimodal tenure distributions that the median-based method cannot. In a gentrifying tract where half the renters are longtime residents at 15% burden and half are new arrivals at 45%, the median lands near 30% — right at the threshold — obscuring that half the renter population is severely burdened. The bracket count captures that directly.

### 2.3 Transportation Cost: The HUD LAI

Transportation cost is the less-intuitive axis of the bivariate map. The motivation is that housing and transportation costs are not independent: households often trade one for the other. Moving to cheaper suburban housing typically increases commute distance and vehicle costs; dense urban neighborhoods with high housing costs often have lower transportation costs due to transit access and walkability. A map showing only housing burden systematically understates the burden on car-dependent communities.

The [HUD Location Affordability Index v3](https://hudgis-hud.opendata.arcgis.com/datasets/location-affordability-index-v3) provides modeled transportation cost estimates at the census tract level. These are outputs of a regression model trained on travel diary data, calibrated to household demographics, incorporating auto ownership rates, annual vehicle miles traveled, fuel prices, insurance costs, and public transit expenditure. They reflect what a representative household of a given type *typically* spends — not what any specific household actually spends.

### 2.4 Adaptive Household Profile Selection

The LAI provides estimates for eight pre-defined household types, ranging from very low income singles (hh2, 12.7% of area median income) to high-income families (hh8, 150% AMI). The application matches each tract to the most demographically appropriate profile using a weighted distance function:

```
dist = 3×(Δincome)² + 1×(ΔhhSize)² + 1×(Δcommuters)²
```

The income ratio (tract median income ÷ area median income) is weighted three times more heavily than household size or commuter count, reflecting its greater influence on transportation spending. All three inputs are available directly from the LAI service, making the selection self-contained.

If the selected profile's estimate is null or suppressed, the application falls back to the nearest available profile by income proximity. The tooltip identifies which profile was used and whether a fallback occurred.

### 2.5 Color Scheme: Continuous OKLAB Interpolation

The bivariate map uses a 3×3 grid of anchor colors interpolated continuously in [OKLAB](https://bottosson.github.io/posts/oklab/) perceptual color space. OKLAB produces equal-looking steps for equal numerical distances across the full hue and lightness range — avoiding the muddy mid-tones that appear when interpolating across hue boundaries in sRGB or linear RGB.

The palette was designed following bivariate cartographic research ([Joshua Stevens](https://www.joshuastevens.net/cartography/make-a-bivariate-choropleth-map/), Cynthia Brewer / ColorBrewer). Key principles:

- **Low-low corner:** desaturated neutral grey with faint warm tint — reads as "baseline / no stress" per Stevens convention
- **High-housing corner (terracotta):** high saturation, warm orange-red — immediately legible as housing stress
- **High-transport corner (slate blue):** cool, saturated — clearly distinct from the housing axis
- **High-high corner (dark eggplant purple):** sits perceptually between the sienna and slate hue families — visually distinctive as a compound stress signal
- **Mid-mid cell:** pale lavender-grey, deliberately light and desaturated — reads as "moderate on both"

#### 3×3 Anchor Grid

Rows = housing burden axis (high → low, top to bottom). Columns = transport cost axis (low → high, left to right).

|  | Low transport | Mid transport | High transport |
|--|:---:|:---:|:---:|
| **High housing** | `#c05828` deep sienna | `#8c3868` deep rose | `#4a1a6e` dark eggplant |
| **Mid housing** | `#e8c098` warm sand | `#d4c0cc` pale lavender | `#4e6898` mid slate blue |
| **Low housing** | `#e0dbd4` warm grey | `#c8d4d8` cool grey | `#7aaab8` slate blue |

Corner interpretations:

| Corner | Color | Meaning |
|--------|-------|---------|
| Low housing / low transport | `#e0dbd4` warm grey | Baseline — low burden on both dimensions |
| Low housing / high transport | `#7aaab8` slate blue | Car-dependent but housing-affordable |
| High housing / low transport | `#c05828` deep sienna | Expensive housing, good transit access |
| High housing / high transport | `#4a1a6e` dark eggplant | Both high — broad-based cost burden |

### 2.6 Dynamic Color Ranges

Rather than applying fixed breakpoints globally, the color scale adapts to the data in view. After each state or county load, the application computes the 10th and 90th percentile of housing burden and transport cost across the loaded features, rounds outward to the nearest 5% for clean legend labels, and enforces a minimum span (15 points for housing, 5 for transport) so homogeneous geographies still show contrast.

At county view, the range is computed across all counties in the state. On drill-down to tract view, it is recomputed across the tracts in the selected county. On return to county view, it reverts to the state-level range.

> **Trade-off:** Dynamic ranges make individual views visually rich, but colors are not comparable across states. A deep sienna in Vermont and a deep sienna in California represent different absolute burden levels. Analysts making cross-state comparisons should use the numeric values in the sidebar table.

### 2.7 Eviction Rate: Metric Choice and Normalization

The application uses the filing rate from the Eviction Lab national proprietary dataset — filings per 100 renter households — because it has the broadest geographic coverage and is the most consistently measured variable across states. Filing rate can exceed 100% in rare cases because a single household can be filed against multiple times in a year.

Hatch thresholds were set at 5%, 10%, and 20% of renter households. The national average eviction filing rate is approximately 4–6%, so these thresholds place the middle band around the national median and the upper band at roughly 2× the median. The latest available year per GEOID is used (data range 2000–2018).

### 2.8 Sparse Tract Filtering

Tracts with fewer than 100 total residents (`B01003_001E`) or fewer than 50 housing units (`B25001_001E`) are flagged as sparse. Sparse tracts are rendered transparent on the map, excluded from the sidebar table, and not included in county-level aggregations. This catches military bases, airports, industrial zones, and prisons while preserving low-density rural tracts with genuine residential populations.

### 2.9 Geographic Boundaries and the MAUP

Census tracts are designed for statistical consistency — each targets approximately 4,000 residents — not to represent meaningful communities. This creates the [Modifiable Areal Unit Problem](https://en.wikipedia.org/wiki/Modifiable_areal_unit_problem) (MAUP): the same underlying data produces different visual patterns and statistics depending on the boundary system used.

School catchment areas were identified as a promising alternative because they reflect actual family activity patterns. The [NCES School Attendance Boundary Survey](https://nces.ed.gov/programs/edge/spatialdata/SABSII) provides catchment polygons for most of the US (Maryland: 99.4% coverage; DC: 100%), though the most recent SABS data is from the 2015–16 school year.

Overlaying census ACS data onto non-census boundaries requires spatial interpolation (areal weighting or dasymetric mapping). The tool was not extended to school catchment boundaries in the current implementation, but the architecture of the tract-level data cache and the existing county→tract aggregation logic would support a future extension.

---

## Part III: Engineering Log

### 3.1 CSV Parsing

| Bug / Symptom | Root Cause | Fix |
|---------------|-----------|-----|
| Illinois, California: zero tract entries | **Quoted-field comma:** `"Cook County, Illinois"` splits into extra columns with naive `split(',')` | State-machine parser tracking `inQuote` state, skipping `\r`, pushing on unquoted commas only |
| All states: last-column GEOID mismatches | **`\r\n` line endings:** trailing `\r` remains on last field | Normalize: `.replace(/\r\n/g,'\n').replace(/\r/g,'\n')` before splitting |
| MD, DC: no county matches | **Double-prefix:** `FIPS_county` column already contains a full 5-char GEOID; state prepend built a 7-char key matching nothing | Use `FIPS_county` as-is; `padStart(5,'0')` only to restore stripped leading zeros |

### 3.2 ArcGIS Pagination

| Bug / Symptom | Root Cause | Fix |
|---------------|-----------|-----|
| Large states: most tracts show N/A transport | **`maxRecordCount` truncation:** State-wide ArcGIS query silently truncated; no error raised | Per-county queries with `resultOffset` pagination loop |
| CA, TX, PA: incomplete after first county | **`exceededTransferLimit` not checked:** Pagination loop not implemented | Loop until `exceededTransferLimit === false` or empty response |

### 3.3 Connecticut's Planning Regions

#### Background

Connecticut abolished county governments in 1960. Its eight legacy counties (FIPS 001–015) existed only as statistical boundaries. In June 2022, the Census Bureau formally replaced them with nine Councils of Governments (COGs) as county-equivalent units (FIPS 110–190). The Census 2022 ACS reflects this change. Tract boundaries were not redrawn — only the county-level code embedded in every tract's 11-character GEOID changed.

This creates a three-way incompatibility:

| Data Source | County System Used | Implication |
|-------------|-------------------|-------------|
| Census API (2022 ACS) | Planning region FIPS 110–190 | Source of housing burden data and tract census data |
| HUD LAI service | Legacy county FIPS 001–015 | Built before the 2022 change; queries for 110–190 return nothing |
| Eviction Lab CSVs | Legacy county prefix in 11-char tract GEOIDs | 2000–2018 data predates the boundary change |

#### Fixes

| Bug / Symptom | Root Cause | Fix |
|---------------|-----------|-----|
| CT: no transport data | Planning region FIPS: LAI has no records for `COUNTY='110'–'190'` | Override CT county list with legacy codes `['001'–'015']` |
| CT: eviction overlay empty | GEOID mismatch: eviction CSV uses legacy county prefix; planning region prefix doesn't match | Index CT eviction tracts by 6-char tract number alone; aggregate via `ctTractToRegion` |
| CT: `ctTractToRegion` empty at toggle | `loadStateData()` built `ctTractToRegion` during LAI processing; the eviction state reset block later in the same function cleared it | Remove `ctTractToRegion` from the eviction reset block — it is LAI/geography data, not eviction data |

### 3.4 JavaScript Timing and Async State

| Bug / Symptom | Root Cause | Fix |
|---------------|-----------|-----|
| County hatching absent on first toggle | **Render-before-data:** `evictionRate` is null when `hatchStyleFor()` called; Leaflet freezes styles at construction time | Call `attachEvictionRates()` before `renderHatchLayer()` in every code path |
| County aggregation uses equal weights prematurely | **Wrong await point:** Toggle didn't await `tractDataPrefetchPromise` | Await prefetch inside toggle handler; add completion re-render |
| Overlay turns off on state change | **Unconditional reset:** `loadStateData()` always cleared `evictionOverlayActive` | `keepOverlayOn = evictionOverlayActive && newStateHasData`; restore after load |
| AZ/FL/RI/TX: null county rates | **Wrong loop source:** Iterated `tractData` (empty at county view) | Loop `evictionTractCache` keys instead; extract county FIPS from GEOID chars 2–4 |

---

## Part IV: Application Architecture

The final application is a single self-contained HTML file with no server-side component and no build step. All data is fetched at runtime from public government APIs and the Eviction Lab S3 bucket.

### 4.1 High-Level Structure

| Layer | Description |
|-------|-------------|
| UI | Leaflet.js map (left, full viewport height) + fixed sidebar (right, 420px). CartoDB light basemap with label overlay pane above the GeoJSON layers. Hover tooltips via a Leaflet custom control. Reset button top-left of map canvas. |
| State machine | Two views: `county` (default) and `tract`. Global variables `currentView`, `currentStateFips`, `currentCountyFips` control which data and layers are active. |
| Data loading | Sequential: Census API → HUD LAI → TIGERweb boundaries. Eviction data is a separate async operation triggered on toggle, not on state load. |
| Rendering | GeoJSON layers styled at construction from pre-computed per-feature data objects. Eviction hatch is a separate GeoJSON layer above the bivariate fill, sharing the same boundary GeoJSON. |
| Caching | `laiTractCache`, `countyData`, `tractData`, `evictionTractCache` / `evictionCountyCache`, `evictionNationalTractText` / `CountyText` (raw CSV strings cached globally so re-parsing on state change avoids a re-fetch). |

### 4.2 Data Sources and API Details

| Source | Endpoint | Notes |
|--------|----------|-------|
| Census ACS 2022 (5-year) | `api.census.gov/data/2022/acs/acs5` | Requires user-supplied API key. Fetched per state at county level, then per county at tract level on drill-down. Key tables: B25070, B25091, B01003, B25001. |
| HUD LAI v3 | `services.arcgis.com/.../Location_Affordability_Index_v3/FeatureServer/0/query` | No API key required. Queried per county with offset pagination. Returns eight household-profile transport cost estimates per tract. CT uses legacy county FIPS (001–015). |
| TIGERweb (Census) | `tigerweb.geo.census.gov/.../MapServer/1` (county) and `MapServer/0` (tract) | No API key required. Returns GeoJSON boundaries. County boundaries fetched once on state load; tract boundaries fetched on drill-down. |
| Eviction Lab | `eviction-lab-data-downloads.s3.amazonaws.com` | Two national CSVs: tract file (~20 MB) and county file (~3 MB). Fetched once on first toggle; cached as raw text strings for the session. CT tract GEOIDs indexed by 6-char tract number only. |
| CartoDB basemap | `{s}.basemaps.cartocdn.com/light_nolabels` + `light_only_labels` | CC BY 3.0 / OpenStreetMap ODbL. Label layer rendered above data layer. |

### 4.3 Key Global State Variables

| Variable | Description |
|----------|-------------|
| `currentStateFips` | 2-char FIPS of the currently loaded state |
| `currentView` | `'county'` \| `'tract'` — controls which layer and data object is active |
| `countyData` | Object keyed by 3-char county FIPS; holds housing burden, transport pct, bivar color, eviction rate per county |
| `tractData` | Object keyed by 9-char `county+tract`; populated on drill-down; same fields plus `isSparse` flag |
| `laiTractCache` | Object keyed by 9-char `county+tract`; holds processed LAI transport data per tract |
| `evictionTractCache` | Object keyed by 11-char GEOID (or 6-char for CT); holds filing rate per tract |
| `evictionCountyCache` | Object keyed by 5-char GEOID; holds filing rate per county |
| `ctTractToRegion` | Object keyed by 6-char tract number; maps to 3-char planning region FIPS; CT only |
| `tractDataPrefetchPromise` | Promise resolving when background Census tract fetch completes; awaited by toggle handler |
| `evictionOverlayActive` | Boolean; persists across state changes if new state has coverage |
| `H_MIN` / `H_MAX` | Dynamic housing burden display range (10th–90th percentile of loaded data, rounded to nearest 5%). Recomputed by `computeRanges()` after each load. |
| `T_MIN` / `T_MAX` | Dynamic transport cost display range. Same computation as `H_MIN`/`H_MAX`. |

### 4.4 Limitations and Future Directions

**Temporal mismatch.** Housing burden data is 2022 ACS; eviction data is 2016 (the most complete pre-pandemic year). The 6-year gap is significant — COVID-era eviction moratoria, the 2021 rental assistance programs, and post-pandemic rent increases all affect both series differently.

**Transport cost modeled, not observed.** HUD LAI estimates are model outputs, not survey-reported figures. They reflect average-household behavior for the matched profile, not the actual spending of households in a specific tract. High-transit urban tracts with below-average car ownership may be systematically overestimated.

**Dynamic ranges suppress cross-geography comparison.** The 10th–90th percentile color scaling makes individual views visually rich, but colors are not comparable across states. Use the numeric values in the sidebar table for cross-state comparisons.

**School catchment boundaries.** The architecture supports future extension to activity-based boundary systems via areal interpolation from census tracts to catchment polygons. The NCES SABS data is the primary source.

**Colorblind accessibility.** The current terracotta × slate-blue palette is not fully colorblind-safe (red-green deficiencies may conflate the housing and transport axes). A future version should include a palette toggle offering a colorblind-friendly alternative.

**Single-file architecture.** The application is deliberately self-contained — no build system, no framework beyond Leaflet. A larger version might separate data fetching, analysis, and rendering into distinct modules.

### 4.5 Development Process

The application was developed collaboratively with Claude (Anthropic), an AI assistant, over approximately 13 sessions. Claude's role covered the full stack: writing and debugging JavaScript, designing the Census API query structure, implementing the OKLAB color interpolation, reasoning through methodological choices (bracket counting vs. medians, adaptive LAI profiles, dynamic range normalization), and producing this documentation.

The collaboration followed a pattern of human analytical direction and AI implementation. Decisions about what to measure, how to frame affordability, which geographic edge cases mattered, and what trade-offs were acceptable were made by the human researcher. Claude translated those decisions into working code, surfaced implementation options and their consequences, and flagged methodological issues that informed subsequent choices.

---

## Part V: Licensing

### License

This software is licensed under the **PolyForm Noncommercial License 1.0.0**.

> Required Notice: Bivariate Choropleth of Household Burdens © the author. Licensed under the PolyForm Noncommercial License 1.0.0 — https://polyformproject.org/licenses/noncommercial/1.0.0

You may use, modify, and distribute this software freely for any **noncommercial purpose**. Permitted uses include personal research, experimentation, education, academic study, public interest work, and use by charitable organizations, educational institutions, public research organizations, public safety or health organizations, environmental protection organizations, and government institutions — regardless of the source of funding.

Commercial use requires a separate agreement with the licensor.

### Full License Text

**PolyForm Noncommercial License 1.0.0**  
https://polyformproject.org/licenses/noncommercial/1.0.0

**Acceptance.** In order to get any license under these terms, you must agree to them as both strict obligations and conditions to all your licenses.

**Copyright License.** The licensor grants you a copyright license for the software to do everything you might do with the software that would otherwise infringe the licensor's copyright in it for any permitted purpose. However, you may only distribute the software according to the Distribution License and make changes or new works based on the software according to the Changes and New Works License.

**Distribution License.** The licensor grants you an additional copyright license to distribute copies of the software. Your license to distribute covers distributing the software with changes and new works permitted by the Changes and New Works License. You must ensure that anyone who gets a copy of any part of the software from you also gets a copy of these terms or the URL for them above, as well as copies of any plain-text lines beginning with `Required Notice:` that the licensor provided with the software.

**Changes and New Works License.** The licensor grants you an additional copyright license to make changes and new works based on the software for any permitted purpose.

**Patent License.** The licensor grants you a patent license for the software that covers patent claims the licensor can license, or becomes able to license, that you would infringe by using the software.

**Noncommercial Purposes.** Any noncommercial purpose is a permitted purpose. Personal use for research, experiment, and testing for the benefit of public knowledge, personal study, private entertainment, hobby projects, amateur pursuits, or religious observance, without any anticipated commercial application, is use for a permitted purpose. Use by any charitable organization, educational institution, public research organization, public safety or health organization, environmental protection organization, or government institution is use for a permitted purpose regardless of the source of funding or obligations resulting from the funding.

**Fair Use.** You may have "fair use" rights for the software under the law. These terms do not limit them.

**No Other Rights.** These terms do not allow you to sublicense or transfer any of your licenses to anyone else, or prevent the licensor from granting licenses to anyone else. These terms do not imply any other licenses.

**Patent Defense.** If you make any written claim that the software infringes or contributes to infringement of any patent, your patent license for the software granted under these terms ends immediately. If your company makes such a claim, your patent license ends immediately for work on behalf of your company.

**Violations.** The first time you are notified in writing that you have violated any of these terms, or done anything with the software not covered by your licenses, your licenses can nonetheless continue if you come into full compliance with these terms, and take practical steps to correct past violations, within 32 days of receiving notice. Otherwise, all your licenses end immediately.

**No Liability.** As far as the law allows, the software comes as is, without any warranty or condition, and the licensor will not be liable to you for any damages arising out of these terms or the use or nature of the software, under any kind of legal claim.

**Definitions.** The *licensor* is the individual or entity offering these terms, and the *software* is the software the licensor makes available under these terms. *You* refers to the individual or entity agreeing to these terms. *Your company* is any legal entity, sole proprietorship, or other kind of organization that you work for, plus all organizations that have control over, are under the control of, or are under common control with, that organization. *Your licenses* are all the licenses granted to you for the software under these terms. *Use* means anything you do with the software requiring one of your licenses.

### Third-Party Data and Licenses

This application fetches and displays data from the following third-party sources, each subject to their own terms of use:

| Source | License / Terms | Notes |
|--------|----------------|-------|
| U.S. Census Bureau ACS | Public domain (U.S. government work) | `api.census.gov` — free API key registration required |
| HUD Location Affordability Index v3 | Public domain (U.S. government work) | Served via ArcGIS FeatureServer; no API key required |
| TIGERweb geographic boundaries | Public domain (U.S. government work) | `tigerweb.geo.census.gov`; no API key required |
| Eviction Lab (Princeton University) | See [evictionlab.org/data-policy](https://evictionlab.org/data-policy) | Research and noncommercial use; national proprietary CSVs from S3 bucket |
| Leaflet.js | BSD 2-Clause License | Map rendering library; unpkg CDN delivery |
| CartoDB basemap tiles | CC BY 3.0 / OpenStreetMap ODbL | `light_nolabels` and `light_only_labels` tile layers |
