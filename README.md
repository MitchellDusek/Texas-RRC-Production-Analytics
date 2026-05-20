# Texas RRC Production Database
### A MySQL relational database built from the Texas Railroad Commission PDQ Dump

![MySQL](https://img.shields.io/badge/Tool-MySQL%209.x-4479A1?logo=mysql&logoColor=white)
![SQL](https://img.shields.io/badge/Focus-Advanced%20SQL-darkgreen)
![Data Modeling](https://img.shields.io/badge/Focus-Schema%20Design-blue)
![Scale](https://img.shields.io/badge/Scale-77M%2B%20Row%20Fact%20Table-orange)
![AI Assisted](https://img.shields.io/badge/AI-Assisted%20Development-8A2BE2)

> Built to demonstrate SQL competency through real-world energy data — not a tutorial dataset. Every design decision was driven by the data itself.

---

## Overview

This project loads, cleans, and structures the Texas Railroad Commission (RRC) Production Data Query (PDQ) Dump into a queryable MySQL relational database. The PDQ Dump is a free public export of Texas oil and gas production data spanning January 1993 to the current month — approximately 33 GB across 16 source files.

The goal was to build a database that supports meaningful analytical questions at the lease, field, operator, and district level, and to demonstrate the SQL techniques required to answer them. The richness of the project comes from the depth of the queries, not the breadth of the schema.

---

## Portfolio Context

This database is the bottom-up complement to the [Energy Intelligence Dashboard](https://github.com/MitchellDusek/Energy-Analytics-Dashboard), a Power BI project analyzing U.S. national energy production, consumption, trade, and energy mix.

The two projects form a coherent analytical stack:

| Layer | Project | Scope |
|---|---|---|
| National | Energy Intelligence Dashboard (Power BI) | U.S. production, consumption, trade, energy mix |
| State / Operator | **This project (MySQL)** | Texas lease, field, operator, and district production detail |

Texas produces approximately 40% of U.S. crude oil. The granular production data in this database directly underlies the national trends visible in the Power BI dashboard, giving the two projects a clear relationship rather than simply coexisting as separate portfolio pieces.

---

## Data Source

**Texas Railroad Commission — Production Data Query (PDQ) Dump**

The PDQ Dump is a complete export of the RRC's Production Data and Historical Ledger databases. It is publicly available as a free monthly download and covers all reported Texas oil and gas production from January 1993 forward. Source files use `}` as the field delimiter with Unix line endings. Total uncompressed size is approximately 33 GB across 16 DSV files.

---

## Database Scale

| Table | Rows | Category |
|---|---|---|
| `og_lease_cycle` | 77,432,187 | Fact |
| `og_lease_cycle_disp` | 47,787,789 | Fact |
| `og_well_completion` | 817,027 | Dimension |
| `og_regulatory_lease_dw` | 545,975 | Dimension |
| `og_operator_dw` | 77,895 | Dimension |
| `og_field_dw` | 65,825 | Dimension |
| `og_summary_master_large` | 1,114,733 | Standalone |
| `og_summary_onshore_lease` | 1,106,000 | Standalone |
| `og_county_cycle` | 240,894 | Standalone |
| `gp_county` | 277 | Reference |
| `gp_district` | 14 | Reference |
| `gp_date_range_cycle` | 1 | Reference |

Three additional derived views aggregate from `og_lease_cycle` at the district, field, and operator level. See [`/schema`](./schema/README.md) for full detail.

---

## SQL Techniques Demonstrated

The query library is built around the analytical questions this data is designed to answer. Techniques used across the query set include:

- Window functions with `PARTITION BY` and `ORDER BY` for rolling per-lease aggregation
- `ROWS BETWEEN` for bounded rolling window calculation
- `STDDEV()` for statistical anomaly detection against a rolling baseline
- `LAG()` for month-over-month comparison
- CTEs to layer multi-step analysis in readable, maintainable steps
- `CASE WHEN` for multi-column conditional interpretation logic
- Multi-table joins across fact, dimension, and reference tables
- Time series aggregation across a 30+ year production history

The featured query — lease inventory signal analysis using `LEASE_OIL_ENDING_BAL` as a rolling supply signal — is documented in full in [`/queries`](./queries/README.md).

---

## Repository Structure

```
/
├── README.md               <- You are here — project overview
├── schema/
│   └── README.md           <- Schema design, table definitions, design decisions
├── load/
│   └── README.md           <- Load process, data cleaning, integrity checks
└── queries/
    └── README.md           <- Analytical questions and SQL query library
```

Each folder contains its own README covering that phase of the project in full.

---

## Future Additions

The following expansions have been evaluated and are feasible without disrupting the existing schema:

- **Lat/long coordinates** — Static reference table joining on `FIELD_NO`. Enables spatial queries.
- **Formation data** — Adds formation name, well type, and depth for reservoir-level analysis.
- **Basin/play lookup table** — Maps fields to basin names (Permian, Eagle Ford, etc.) for basin-level filtering.
- **Macro financial layer** — A potential third portfolio project layering commodity prices and energy equity data above the national and state production tiers.
