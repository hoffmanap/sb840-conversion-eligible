# SB 840 Conversion Eligibility Dataset — El Paso, TX

## What this is

A parcel-level dataset identifying properties in El Paso potentially eligible for
conversion to mixed-use residential or multifamily housing under **Texas SB 840**
(89th Legislature, effective 9/1/2025), which added Chapter 218 to the Local
Government Code. SB 840 requires cities meeting certain population thresholds to
allow conversion of existing office, retail, and warehouse buildings to
multifamily/mixed-use residential occupancy, subject to specific conditions, and
preempts several categories of municipal regulation for such projects (parking
minimums, traffic studies, impact fees, etc. — see the "Bill Requirements &
Benefits" section below).

## Data sources

| File | Source | Role |
|---|---|---|
| `Improvements2027Dump` | EPCAD (El Paso Central Appraisal District) | One row per building/improvement component. Carries the structure class code, Texas Comptroller state class code, year built, and square footage. |
| `Properties2027Dump` | EPCAD | Crosswalk linking each improvement's `Property_dbId` to a `GeoID` (the reliable key into the parcel layer — see Assumptions). |
| `Parcel_2025.shp` | EPCAD | Parcel geometry, land acreage, situs address, and parcel identifiers (`PIDN`, `GEO_ID`). |
| `2025_Codes_table` | EPCAD | Lookup table defining every `imprv_det_class_cd` structure class code (e.g. `MRCA` → "Retail Store; Masonry; Average"). |

## Methodology

### 1. Eligible use classification
SB 840 §218.202 limits the conversion pathway to buildings currently used for
**office, retail, or warehouse**. EPCAD's structure class codes are organized by
prefix:
- `P` = Professional/office
- `M` = Mercantile/retail
- `I` = Industrial (warehouse is a *subset* — heavy manufacturing, engineering/
  research, and similar non-warehouse industrial codes were explicitly excluded)

Codes were classified as eligible using a combination of prefix rules and
keyword matching against each code's description, with explicit exclusions for
subtypes that share a prefix but aren't genuinely qualifying uses (government
buildings, hospitals/nursing homes, restaurants, motels, service garages, etc.).
The reviewed/edited eligible-code list is in `sb840_eligible_codes.csv`.

### 2. Confirmed vs. potential candidates
Every parcel has one or more improvement rows (main building plus additions,
canopies, parking structures, etc.), and many secondary-component rows carry a
generic placeholder code (`*`) rather than a specific structure type. To avoid
under-counting genuinely eligible commercial stock whose main-structure row
happens to be unclassified, each property is tagged:

- **`confirmed_conversion_candidate`** — has at least one improvement row with a
  specific structure class code on the reviewed eligible-codes list.
- **`potential_conversion_candidate`** — no confirmed row, but the property's
  Texas Comptroller state class is `F1` (Commercial Real) or `F2` (Industrial
  Real), and none of its improvement rows carry a code we can confidently rule
  out as non-qualifying (restaurant, motel, daycare, etc.). These need a manual
  look before being treated as truly eligible — the data alone can't confirm
  the specific use.

### 3. Five-year construction-age test
SB 840's conversion pathway (§218.202(3)) requires the building have been
constructed at least 5 years before the conversion date. `YearBuilt` from the
Improvements file is used, preferring the confirmed structure's year when
available. Properties whose building is confirmed under 5 years old are
excluded; properties with no year-built data on file are *kept but flagged*
(`Age_Test_Status = "Unknown - verify manually"`) rather than silently dropped.

### 4. Density and unit-size estimates
SB 840 §218.102(a)(1) caps municipal density restrictions at the greater of the
city's highest allowed residential density or **145 units per acre** — El Paso's
applicable ceiling. Two estimates are provided per parcel:

- **Maximum units allowed**: `Land_Acres × 145`, rounded down.
- **Average unit size at max density**: `Total_Improvement_SqFt ÷ Max_Units_Allowed`
  — i.e., how small units would need to be to hit the legal maximum.
- **Units at a realistic unit size**: `Total_Improvement_SqFt ÷ 650`, using 650
  sq ft as a representative one-bedroom unit size for El Paso (a more realistic,
  non-maximized planning estimate than the max-density calculation).

`Total_Improvement_SqFt` sums the `SquareFootage` field across all improvement
rows counted toward a property's eligibility — it approximates total building
area, not living area specifically.

## Corrections log

- **Eligibility bug (fixed):** the initial prefix-based classifier matched
  any structure code starting with "M" as mercantile/retail. EPCAD's table
  also uses bare codes `M1`–`M5` to mean **multifamily** (residential), and
  `QI` ("Office/Apartments") for a building that already contains housing.
  Both were being incorrectly counted as eligible office/retail candidates.
  Fixed by requiring a letter (not a digit) immediately after "M" for
  mercantile matches, excluding `QI` explicitly, and adding a blanket
  description-based exclusion for any code mentioning multifamily,
  apartment, condo, duplex/triplex/quadplex, or mobile home — regardless of
  which prefix it happens to share. **Existing residential buildings,
  including multifamily, are not eligible conversion candidates under SB
  840 and are excluded from both the confirmed and potential tiers.**
- **Scope corrected to City of El Paso only.** SB 840 Chapter 218 applies
  within municipal jurisdiction, not the extraterritorial jurisdiction or
  other municipalities in the county. The parcel filter now requires the
  city/jurisdiction field to match El Paso before a parcel is included —
  confirm the exact field/value for your parcel export with
  `inspect-city-field`.

## Key assumptions and known limitations

- **EPCAD's documented schema for the Properties crosswalk file did not match
  the actual data.** The published column order (`dbId`, `PropertyId`, ...) was
  empirically found to be swapped (`PropertyId`, `dbId`, ...), confirmed by
  cross-referencing known DBID values from the Improvements file. All joins in
  this pipeline were verified empirically against actual data, not assumed from
  documentation.
- **`PropertyId` (crosswalk) and `PROP_ID` (parcel file) are NOT the same key**,
  despite similar naming. Both are small sequential integers occupying similar
  numeric ranges, which produced a misleadingly plausible-looking ~20% overlap
  that was actually mostly coincidental. The reliable join key between the
  crosswalk and parcel layer is **`GeoID`** (alphanumeric legal/geo identifier),
  verified at a ~99.9% match rate against the parcel file's `GEO_ID` field.
- **The "potential" tier is a genuine judgment call, not a certainty.** It
  exists because a large share of `F1`/`F2` (Commercial/Industrial Real)
  improvement rows carry a generic wildcard structure code rather than a
  specific building type — the data cannot confirm the actual current use for
  these, only that the property is broadly commercial/industrial in nature.
- **This dataset does not evaluate the bill's other conversion conditions**,
  including the 65%-residential-of-building-area test, mixed-use's 65%
  residential / up to 35% nonresidential split, proximity exclusions (airports,
  military bases, heavy industrial uses), or whether a given parcel sits in a
  zoning classification that already permits office/commercial/retail/warehouse/
  mixed-use (a prerequisite under §218.101). Those require additional data
  (zoning layer, proximity analysis) not yet incorporated.
- **Government-owned, hospital/nursing home, and other institutional-care
  buildings were excluded by default** even where their class code shares a
  prefix with genuine office use, since they aren't realistic private
  redevelopment candidates. This is an editorial judgment call and can be
  reversed in `sb840_eligible_codes.csv` / the script's `EXCLUDE_PREFIXES`.
- Reprojection to WGS84 (EPSG:4326) for GeoJSON output assumes the parcel
  shapefile's embedded CRS (Texas State Plane Central, EPSG:2277) is correct;
  this was confirmed from the shapefile's `.prj` sidecar during processing.

## Findings summary

*(Fill in with your final run's numbers before sharing externally — the counts
below are placeholders reflecting the last run in this session.)*

- **Total eligible parcels identified:** ~4,585 (subject to change with the
  5-year age filter and density calculations now applied)
- **Confirmed conversion candidates:** ~3,635
- **Potential conversion candidates (needs manual review):** ~950
- Eligible parcels are concentrated in and around downtown El Paso and the
  city's designated commercial corridors, consistent with expected commercial/
  office/retail/warehouse land use patterns.

## Files in this deliverable

- `epcad_sb840_prefilter.py` — the full processing pipeline (diagnostic and
  filter commands; see the script's docstring for usage).
- `sb840_eligible_codes.csv` — the reviewed/edited list of eligible structure
  class codes, with descriptions.
- `sb840_eligible_parcels.geojson` — the final parcel-level output, ready for
  mapping/dashboarding.
- `README.md` — this file.
