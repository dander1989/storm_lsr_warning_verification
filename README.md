# StormVerify-Py: Automated NWS Warning Verification & Spatial Audit

A spatial data pipeline that evaluates the accuracy of NWS convective warnings by joining them against ground-truth Local Storm Reports (LSRs) using DuckDB's spatial engine and Python.

---

## Overview

The National Weather Service issues polygon-based warnings to protect life and property. A warning is only considered "verified" if a qualifying ground-truth storm report occurs **inside the polygon** while the warning is **still active**. This project automates that verification process for a single high-impact event day and produces an interactive map showing which warnings were hits and which were false alarms.

The target event is **April 27, 2026**, a Moderate Risk day across the Midwestern US with 380 total storm reports logged by the SPC (21 tornado, 234 wind, 125 hail, plus high wind and large hail reports).

> **Note on tornado reports:** Storm surveys for this event are still ongoing. The tornado report count may increase as surveys conclude and previously unconfirmed tornadoes are verified. This is expected and reflects how ground-truth data works operationally.

---

## Warning Types and Verification Criteria

This project covers two warning types with type-specific verification criteria:

| Warning Type | Verified By |
|---|---|
| Tornado Warning (TO) | Tornado reports only |
| Severe Thunderstorm Warning (SV) | Hail reports OR damaging wind reports |

Flash Flood Warnings are out of scope for this project.

A Tornado Warning is not considered verified by hail or wind reports alone, even though those events can co-occur with tornadoes. The NWS operationally requires a tornado report to verify a Tornado Warning. It's also worth noting that some tornado reports occurred inside Severe Thunderstorm Warning polygons where a Tornado Warning was subsequently issued — in those cases the tornado report verifies the TOR warning, and the SVR warning is classified separately.

---

## Verification Status Labels

Each warning polygon in the final output carries one of these labels:

| Status | Description |
|---|---|
| Verified | Met the type-specific verification criteria |
| Unverified - No Reports | No qualifying reports inside the polygon during the valid time |
| Unverified - Tornado Warning w/ Hail | TOR warning with hail reports but no tornado report |
| Unverified - Tornado Warning w/ Wind | TOR warning with wind reports but no tornado report |
| Unverified - Tornado Warning w/ Hail + Wind | TOR warning with both hail and wind reports but no tornado report |
| Unverified - SVR w/ Tornado Report | SVR warning with a tornado report inside but no hail or wind reports |

---

## Results: April 27, 2026

A total of **421 warnings** (TO and SV combined) were analyzed across the event period (1200 UTC April 27 to 1159 UTC April 28).

### Overall Verification Summary

| Status | Count |
|---|---|
| Verified | 225 |
| Unverified - No Reports | 160 |
| Unverified - Tornado Warning w/ Wind | 21 |
| Unverified - Tornado Warning w/ Hail | 10 |
| Unverified - Tornado Warning w/ Hail + Wind | 3 |
| Unverified - SVR w/ Tornado Report | 2 |

### By Warning Type

| Warning Type | Total | Verified | Verification Rate |
|---|---|---|---|
| Tornado (TO) | 89 | 13 | ~15% |
| Severe Thunderstorm (SV) | 332 | 212 | ~64% |

The contrast between warning types tells an interesting story. SVR warnings verified at a much higher rate, which is expected — hail and wind reports are common with organized convection. Tornado warnings are significantly harder to verify because many are issued on radar-detected rotation signatures that never produce a confirmed tornado touchdown. 34 tornado warnings had some kind of report inside them (hail, wind, or both) but still did not meet the verification threshold.

### Notable WFO Findings

On the SVR side, LMK and IND performed well with verification rates of 85.71% and 83.3% respectively. LZK stood out at the other end with only 1 verified warning out of 12 issued (8.33%).

On the tornado side, overall verification rates were low across the board, consistent with the day's storm mode. ILX, LSX and IND issued the most tornado warnings (17, 26, and 9 respectively) with relatively few verifications.

---

## A Note on Overlapping Warnings

It is intentional that a single storm report can verify more than one warning polygon if it spatially and temporally satisfies both. This reflects real NWS operational practice — warnings sometimes overlap because a new polygon is issued before the previous one expires to maintain continuous coverage as a storm moves.

---

## Data Sources

All data comes from the [Iowa Environmental Mesonet (IEM)](https://mesonet.agron.iastate.edu/):

- **LSRs:** [IEM LSR Archive](https://mesonet.agron.iastate.edu/request/gis/lsrs.phtml) — filtered to the SPC convective day (1200 UTC April 27 to 1159 UTC April 28)
- **Storm-Based Warnings:** [IEM Warning Archive](https://mesonet.agron.iastate.edu/request/gis/watchwarn.phtml) — storm-based polygons, not county-wide alerts

### Data Quality Notes

- One LSR record with an erroneous latitude of 8.9 degrees (a TSTM WND GST report from a private weather sensor in Jackson, MO) was identified and removed prior to analysis.
- Near-duplicate tornado reports exist in the IEM data — two records separated by approximately 0.01 decimal degrees (~1km) were identified. These were retained since they likely represent the same tornado at slightly different survey points, and duplicate reports do not affect the binary verification logic.
- TSTM WND GST and TSTM WND DMG report types are combined as wind verification criteria for SVR warnings.
- The SPC convective day window (1200-1159 UTC) was used rather than a calendar day to match standard severe weather reporting conventions and ensure all significant convection from the event is captured.

---

## Stack

- **Python** — data fetching, preprocessing, and export
- **DuckDB + spatial extension** — all spatial joins, aggregations, and classification
- **GeoPandas / PyArrow** — GeoJSON and GeoParquet I/O
- **Felt** — interactive map visualization

---

## Project Structure

```
storm_lsr_warning_verification/
│
├── notebook.ipynb # Main pipeline: ingest, join, classify, export
│        
│
├── data/                     # Git-ignored: raw data
│   ├── raw/
│   └── processed/
│       ├── verified_warnings.geojson   # Warning geometries with per report summaries
│       ├── warn.parquet    # Raw warnings
│       └── lsr.parquet     # Raw Local Storm Reports (LSRs)
│     
│
└── README.md
```

---

## How the Pipeline Works

1. **Ingest** — pull LSRs and storm-based warnings from the IEM API for the SPC convective day window
2. **Preprocess** — filter LSRs to qualifying report types; filter warnings to TO and SV only; convert timestamps to datetime; construct Point geometries for LSRs
3. **Convert** — write filtered GeoDataFrames to GeoParquet for efficient DuckDB loading
4. **Spatial Join** — point-in-polygon LEFT JOIN with a temporal constraint (report time must fall within warning issue and expiry times)
5. **Aggregate** — count reports by type (tornado, hail, wind) per warning, producing one row per warning
6. **Classify** — apply CASE WHEN logic to assign a verification status label per warning
7. **WFO Stats** — roll up to WFO level for verification rate summary
8. **Export** — write final classified GeoJSON of warning polygons
9. **Visualize** — interactive map in Felt with traffic-light styling

---

## Key SQL Logic

The aggregation query counts qualifying reports by type per warning:

```sql
SELECT
    w.wfo,
    w.etn,
    w.warn_type,
    w.warn_geom,
    COUNT_IF(typetext = 'TORNADO')                                    AS tornado_reports,
    COUNT_IF(typetext = 'HAIL')                                       AS hail_reports,
    COUNT_IF(typetext = 'TSTM WND DMG' OR typetext = 'TSTM WND GST') AS wind_reports,
    COUNT(typetext)                                                    AS total_reports
FROM lsr_warn_join
GROUP BY wfo, etn, warn_type, warn_geom
```

Classification is then applied as a follow-up CASE WHEN query on those counts, with logic branching per warning type.

---

## Limitations

This pipeline identifies false alarms (warned, no qualifying report) well but does not fully quantify missed events (qualifying report with no corresponding warning). Some tornado reports occurred outside all active Tornado Warning polygons — identifying those systematically would require an additional spatial join and is noted as a stretch goal.

---

## Stretch Goals

- **Missed Event Analysis** — identify tornado reports that fell outside all active TOR warning polygons; check whether those reports fell inside SVR polygons instead (spin-up tornadoes in line convection)
- **Lead Time Calculation** — time between warning issuance and the first qualifying report inside the polygon
- **Date Range Script** — a command line script that prompts for a date range and runs the full pipeline automatically for any event period

---

## Skills Demonstrated

- Spatial SQL with DuckDB and the spatial extension
- Point-in-polygon joins with compound spatial and temporal constraints
- GeoParquet as a performant intermediate format
- Data quality auditing and documented decision-making
- Interactive geospatial visualization with Felt
- Reproducible geospatial pipeline in a Jupyter notebook
