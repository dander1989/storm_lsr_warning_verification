# StormVerify-Py: Automated NWS Warning Verification & Spatial Audit

A spatial data pipeline that evaluates the accuracy of NWS convective warnings by joining them against ground-truth Local Storm Reports (LSRs) using DuckDB's spatial engine and Python.

---

## What This Project Does

The National Weather Service issues polygon-based warnings to protect life and property. A warning is only considered "verified" if a qualifying ground-truth storm report occurs **inside the polygon** while the warning is **still active**. This project automates that verification process for a single high-impact event day and produces an interactive map showing which warnings were hits and which were false alarms.

The target event is **April 27, 2026**, a Moderate Risk day across the Midwestern US with 380 total storm reports logged by the SPC (21 tornado, 234 wind, 125 hail, plus high wind and large hail reports).

> **Note on tornado reports:** Storm surveys for this event are still ongoing. The tornado report count may increase as surveys conclude and previously unconfirmed tornadoes are verified. This is expected and reflects how ground-truth data works operationally.

---

## Warning Types Covered

This project covers two warning types with type-specific verification criteria:

| Warning Type | Verified By |
|---|---|
| Tornado Warning (TOR) | Tornado reports only |
| Severe Thunderstorm Warning (SVR) | Hail reports OR damaging wind reports |

Flash Flood Warnings are out of scope for this project.

A Tornado Warning is **not** considered verified by hail or wind reports alone, even though those events can co-occur with tornadoes. The reasoning: large hail and strong winds can happen independently of a tornado touching down, and the NWS operationally requires a tornado report to verify a Tornado Warning.

---

## Verification Status Labels

Each warning polygon in the final output gets one of these labels:

- **Verified** — met the type-specific criteria above
- **Unverified - No Reports** — no qualifying reports inside the polygon during the warning's valid time
- **Unverified - Tornado w/ Hail** — tornado warning with hail reports inside but no tornado report
- **Unverified - Tornado w/ Wind** — tornado warning with wind reports inside but no tornado report

---

## Data Sources

All data comes from the [Iowa Environmental Mesonet (IEM)](https://mesonet.agron.iastate.edu/):

- **LSRs (Local Storm Reports):** [IEM LSR Archive](https://mesonet.agron.iastate.edu/request/gis/lsrs.phtml) — filtered to April 27, 2026
- **Warnings (VTEC Storm-Based):** [IEM Warning Archive](https://mesonet.agron.iastate.edu/request/gis/watchwarn.phtml) — use "Storm Based Warnings" to get actual polygons rather than county-wide alerts

Data is downloaded as GeoJSON and converted to GeoParquet for efficient loading into DuckDB.

---

## Stack

- **Python** — data fetching, preprocessing, and visualization
- **DuckDB + spatial extension** — all spatial joins and aggregations
- **GeoPandas / PyArrow** — GeoJSON to GeoParquet conversion
- **Folium** — interactive map output

---

## Project Structure

```
stormverify-py/
│
├── notebooks/
│   └── analysis.ipynb        # Main pipeline: ingest, join, classify, visualize
│
├── data/                     # Git-ignored: raw and processed data
│   ├── raw/
│   └── processed/
│
└── README.md
```

---

## How the Pipeline Works

The core logic follows these steps:

1. **Ingest** — pull LSRs and storm-based warnings from the IEM API for the 24-hour window
2. **Preprocess** — filter LSRs to qualifying report types; filter warnings to TOR and SVR only
3. **Convert** — write GeoJSON to GeoParquet for DuckDB
4. **Spatial Join** — point-in-polygon join with a temporal constraint (report time must fall within warning issue and expiry times)
5. **Aggregate** — count reports by type per warning, producing one row per warning
6. **Classify** — apply CASE WHEN logic to assign a verification status label
7. **Export** — write final GeoJSON of classified warning polygons
8. **Visualize** — render an interactive Folium map with a traffic-light color scheme

### A note on overlapping warnings

It is intentional that a single storm report can verify more than one warning polygon if it spatially and temporally satisfies both. This reflects real NWS operational practice: warnings sometimes overlap because a new polygon is issued before the previous one expires to maintain continuous coverage as a storm moves.

---

## Key SQL Logic (DuckDB)

The aggregation query groups by warning and counts qualifying reports by type:

```sql
SELECT
    w.event_id,
    w.wfo,
    w.warning_type,
    COUNT(CASE WHEN l.type = 'TORNADO' THEN 1 END)                        AS tornado_report_count,
    COUNT(CASE WHEN l.type IN ('HAIL', 'LARGE HAIL') THEN 1 END)          AS hail_report_count,
    COUNT(CASE WHEN l.type IN ('TSTM WIND GST', 'HIGH WIND') THEN 1 END)  AS wind_report_count
FROM warnings w
LEFT JOIN lsrs l
    ON ST_Intersects(ST_GeomFromWKB(l.geom), ST_GeomFromWKB(w.geom))
    AND l.valid_time BETWEEN w.issue_time AND w.expire_time
GROUP BY w.event_id, w.wfo, w.warning_type
```

Classification is then applied as a follow-up query using CASE WHEN on the counts above.

---

## Output Metrics

Per-WFO verification stats are calculated as:

```
verification_rate = verified_warnings / total_warnings_issued
```

Results are summarized in a table and visualized on the interactive map.

---

## Map Styling

The output map uses a traffic-light color scheme on warning polygons:

- 🟢 **Green** — Verified
- 🔴 **Red** — Unverified (No Reports)
- 🟡 **Yellow** — Unverified with partial reports (hail or wind inside a tornado warning)

Clicking a polygon shows a popup with the WFO, event ID, warning type, report counts, and verification status.

---

## Stretch Goals

- **Lead Time Calculation** — time between warning issuance and the first qualifying report (how much heads-up the public had)
- **Warning-to-Report Distance** — for unverified warnings, use `ST_Distance` to find how close the nearest qualifying report was

---

## Skills Demonstrated

- Spatial SQL with DuckDB and the spatial extension
- Point-in-polygon joins with compound spatial + temporal constraints
- GeoParquet as a performant intermediate format
- Interactive geospatial visualization with Folium
- Reproducible geospatial pipeline in a Jupyter notebook
