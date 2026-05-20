# Schema Design
### Table definitions, design decisions, and data model overview

---

## Overview

The `rrc_production` database is built around a central fact table — `og_lease_cycle` — with dimension and reference tables supporting joins at the lease, operator, field, and district level. Three derived views provide pre-aggregated rollups without adding physical table overhead.

12 physical tables were loaded. 3 rollup tables from the source were replaced by derived views. 1 source table was excluded entirely due to size.

---

## Table Categories

### Reference Tables

Small, stable lookup tables used for joining and display. These do not change with monthly data refreshes.

| Table | Rows | Description |
|---|---|---|
| `gp_district` | 14 | RRC district definitions. `DISTRICT_NO` is an internal sequential ID (01-14); `DISTRICT_NAME` is the public designation (e.g., internal `07` = public `6E`, internal `10` = public `08`). Always join on `DISTRICT_NO`, display `DISTRICT_NAME`. |
| `gp_county` | 277 | Texas county definitions with district assignments and geographic codes. |
| `gp_date_range_cycle` | 1 | Single-row table defining the current PDQ export date range. |

### Dimension Tables

Descriptive tables for the core entities in the production data. These define the who, what, and where.

| Table | Rows | Description |
|---|---|---|
| `og_field_dw` | 65,825 | Field master — field name, district, county, field type, and status. |
| `og_operator_dw` | 77,895 | Operator master — operator name, P-5 organization report data, and regulatory status. |
| `og_regulatory_lease_dw` | 545,975 | Lease master — lease name, type, district, county, field assignment, operator assignment, and regulatory attributes including allowable and severance flag. |
| `og_well_completion` | 817,027 | Well-level detail — well number, API number, completion type, status codes, and shut-in dates. |

### Fact Tables

The core production records. These are updated monthly with the PDQ refresh.

| Table | Rows | Description |
|---|---|---|
| `og_lease_cycle` | 77,432,187 | Monthly production volumes per lease — oil, gas, condensate, casinghead gas, allowables, ending inventory balance, and disposition totals. The primary analytical table. |
| `og_lease_cycle_disp` | 47,787,789 | Monthly disposition detail per lease — breaks total disposed volume into 40 individual disposition code columns (sold, gas lift injection, repressure, vented, etc.). |

### Standalone Tables

Tables from the PDQ Dump that are not joined into the main schema. These provide independent summary views from a different RRC reporting system and are queried independently.

| Table | Rows | Description |
|---|---|---|
| `og_county_cycle` | 240,894 | Estimated county-level production volumes. Only 4 of 16 volume columns are populated by the RRC at this level. |
| `og_summary_master_large` | 1,114,733 | Summary production records from the RRC's separate summary reporting system. |
| `og_summary_onshore_lease` | 1,106,000 | Onshore lease-level summary records from the same system. |

### Derived Views

Three rollup tables present in the PDQ Dump (`OG_DISTRICT_CYCLE`, `OG_FIELD_CYCLE`, `OG_OPERATOR_CYCLE`) were not loaded as physical tables. Instead, equivalent derived views were created that aggregate directly from `og_lease_cycle`, keeping rollup numbers always consistent with the underlying lease-level data.

| View | Aggregation |
|---|---|
| `v_district_cycle` | Production by district, cycle year/month, and oil_gas_code |
| `v_field_cycle` | Production by field, district, cycle, and oil_gas_code |
| `v_operator_cycle` | Production by operator, cycle, and oil_gas_code |

**Performance note:** These views GROUP BY across 77 million rows on every execution. For repeated analytical use, materializing into indexed summary tables is worth considering.

---

## Key Design Decisions

### Oracle DATE Fields Stored as VARCHAR

All date columns from the source files are stored as `VARCHAR` rather than MySQL `DATE`. The source is an Oracle database export using `DD-MON-YY` format (e.g., `14-FEB-26`). MySQL does not natively parse this format on ingestion. Storing as VARCHAR preserves the data exactly as received and avoids silent conversion errors.

For date arithmetic in queries, use `STR_TO_DATE(col, '%d-%b-%y')` to convert at query time.

### Composite Primary Keys

Several tables use composite primary keys rather than surrogate keys. This mirrors how the RRC source data is structured and keeps the schema honest — a lease is uniquely identified by its lease number and district number together, not by an auto-incremented ID that would have no meaning outside this database.

### Foreign Key Orphan Handling

Production history spans back to 1993. Dimension tables reflect the current state of the RRC's records. As a result, some historical production records reference operators, fields, or leases that no longer exist in the current dimension exports. These are legitimate historical records, not data errors. FK checks were disabled during load for affected tables and re-enabled immediately after. Orphan counts are documented in [`/load`](../load/README.md).

### OG_COUNTY_LEASE_CYCLE Excluded

The PDQ Dump includes an `OG_COUNTY_LEASE_CYCLE` file containing county-level breakdowns at the lease level. At 11.63 GB it is the largest single file in the dump and adds significant load time and storage cost for analytical value that is largely available through direct lease-level queries on `og_lease_cycle`. It was excluded from this build.

---

## Verified Integrity — December 2025

A sample analytical query joining `og_lease_cycle`, `og_field_dw`, `og_operator_dw`, and `gp_district` for December 2025 confirmed that joins, filters, and aggregations return accurate, coherent results across the schema.

| District | Operator | Field | Oil (BBL) |
|---|---|---|---|
| 08 | DIAMONDBACK E&P LLC | SPRABERRY (TREND AREA) | 14,841,152 |
| 08 | PIONEER NATURAL RES. USA, INC. | SPRABERRY (TREND AREA) | 14,755,616 |
| 7C | PIONEER NATURAL RES. USA, INC. | SPRABERRY (TREND AREA) | 3,541,803 |
| 08 | XTO ENERGY INC. | SPRABERRY (TREND AREA) | 3,399,853 |
| 01 | EOG RESOURCES, INC. | EAGLEVILLE (EAGLE FORD-1) | 3,301,800 |

All three derived views were smoke-tested against the same period and returned results consistent with direct `og_lease_cycle` queries.
