# Chinook Data Engineering Pipeline
### End-to-end Medallion Architecture on Databricks | Azure SQL → Raw → Bronze → Silver → Gold

![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=flat&logo=databricks&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta%20Lake-003366?style=flat&logo=delta&logoColor=white)
![Azure SQL](https://img.shields.io/badge/Azure%20SQL-0078D4?style=flat&logo=microsoftazure&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=flat&logo=python&logoColor=white)
![Apache Spark](https://img.shields.io/badge/Apache%20Spark-E25A1C?style=flat&logo=apachespark&logoColor=white)

---

## Overview

A metadata-driven data engineering pipeline built on **Databricks** using the **Medallion Architecture** (Raw → Bronze → Silver → Gold). The pipeline ingests data from the [Chinook music store dataset](https://github.com/lerocha/chinook-database) hosted in Azure SQL, processes it through four layers of increasing quality and structure, and delivers a fully normalised dimensional model ready for analytics.

---

## Architecture

```
Azure SQL (Chinook)
                        │
                        │  Databricks Connection Manager (Lakehouse Federation)
                        ▼
┌─────────────────────────────────────────────────────┐
│  RAW ZONE  ·  Databricks Volume  ·  Parquet         │
│  /Volumes/{catalog}/raw_zone/chinook/{table}/       │
│  {yyyy}/{MM}/{dd}/{table}_{timestamp}.parquet       │
│  Immutable snapshots — never overwritten            │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│  BRONZE  ·  Unity Catalog Delta                     │
│  Current day's snapshot — overwrite mode            │
│  Exact copy of Raw in Delta format                  │
└───────────────────────┬─────────────────────────────┘
                        │
                        │  DQX profiling + quality rules
                        │  Failed records → quarantine table
                        ▼
┌─────────────────────────────────────────────────────┐
│  SILVER  ·  Unity Catalog Delta                     │
│  Cleaned and standardised                           │
│  TRIM · COALESCE · lowercase email · date cast      │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│  GOLD  ·  Unity Catalog Delta                       │
│  Dimensional model — 7 dims + 2 facts               │
│  dim_customer: SCD Type 2                           │
└─────────────────────────────────────────────────────┘
```

---

## Gold Layer — Dimensional Model

| Table | Type | Rows | Description |
|---|---|---|---|
| `dim_date` | Dimension | 354 | Derived from invoice dates — year, quarter, month, day, weekend flag |
| `dim_artist` | Dimension | 275 | Artist with surrogate key |
| `dim_album` | Dimension | 347 | Album with FK → dim_artist |
| `dim_genre` | Dimension | 25 | Genre lookup |
| `dim_media_type` | Dimension | 5 | Media type lookup |
| `dim_employee` | Dimension | 8 | Employee with self-referencing manager FK |
| `dim_customer` | SCD Type 2 | 59+ | Customer history — tracks all attribute changes across pipeline runs |
| `dim_track` | Dimension | 3503 | Track with FKs to album, genre, media type; derived duration |
| `fact_sales` | Fact | 2240 | Invoice line grain — one row per line item |
| `fact_sales_customer_agg` | Aggregate Fact | 59 | Customer-level aggregation built from fact_sales |

---

## Key Features

**Metadata-driven design**
The pipeline reads a parent metadata table to determine which tables to extract each run. To add a new source table, insert one row — zero code changes needed.

**Immutable Raw zone**
Every pipeline run writes a new timestamped Parquet file to the Volume. Existing snapshots are never overwritten. Recovery is always possible by reprocessing from Raw.

**DQX data quality validation**
Before writing to Silver, every Bronze table is profiled (nulls, duplicates, type consistency, value ranges) and validated against quality rules. Failed records go to a quarantine table with the violation reason.

**SCD Type 2 on dim_customer**
The customer dimension tracks historical changes using Delta MERGE. When customer attributes change between pipeline runs, the old record is expired (`is_current = false`, `effective_end_date` set) and a new current record is inserted. Fact tables always join to the current record via `is_current = true`.

**Fully parameterised**
No hardcoded values anywhere. All catalog names, schema names, and paths are passed as widget parameters through the Databricks Job configuration panel.

**End-to-end Job orchestration**
A single Databricks Job with 4 sequential tasks, task dependencies, job-level parameters, email notifications on success and failure, and concurrency settings.

---

## Repository Structure

```
chinook-pipeline/
│
├── notebooks/
│   ├── 00_environment_setup.py          # Creates schemas, Volume, metadata tables
│   ├── 00b_connection_manager_setup.py  # Connection Manager guide
│   ├── 01_extract_from_source.py        # Azure SQL → Raw Volume (Parquet)
│   ├── 02_raw_to_bronze.py              # Raw Volume → Bronze Delta
│   ├── 03_bronze_to_silver.py           # DQX validation + cleaning → Silver Delta
│   └── 04_silver_to_gold.py            # Dimensional model + SCD2 → Gold Delta
│
├── sql/
│   ├── 00_metadata_tables.sql           # Metadata + control table DDL
│   ├── version2_azure_sql_updates.sql   # Version 2 UPDATE statements for SCD2 demo
│   └── version2_scd2_verification.sql   # SCD2 verification queries
│
└── README.md
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| Databricks workspace | Unity Catalog enabled |
| Azure SQL | Chinook dataset loaded ([script](https://github.com/lerocha/chinook-database)) |
| Databricks Runtime | 13.3 LTS or higher |
| Permissions | CREATE SCHEMA, CREATE VOLUME on your catalog |

---

## Setup & Deployment

### Step 1 — Environment setup

Run `00_environment_setup.py` in Databricks with your catalog name. This creates:
- `raw_zone` schema + `chinook` Volume
- `bronze`, `silver`, `gold` schemas
- `pipeline_control` schema with all metadata and control tables

### Step 2 — Connection Manager

In Databricks Catalog Explorer:
```
Catalog → gear icon → Connections → Create connection
  Type: SQL Server
  Host: <your-server>.database.windows.net
  Port: 1433

Then: + Add → Foreign catalog
  Name: chinook_source
  Connection: (the one you just created)
```

### Step 3 — Verify connection

```sql
SELECT * FROM chinook_source.chinook.Album LIMIT 5;
```

### Step 4 — Import notebooks

Import all notebooks from the `notebooks/` folder into a Databricks Git folder.

### Step 5 — Configure and run the Job

Create a Databricks Job with 4 tasks in sequence:

```
01_extract_from_source → 02_raw_to_bronze → 03_bronze_to_silver → 04_silver_to_gold
```

Add these Job-level parameters:

| Parameter | Value |
|---|---|
| `catalog_name` | your catalog (e.g. `workspace`) |
| `schema_name` | `pipeline_control` |
| `control_schema` | `pipeline_control` |
| `bronze_schema` | `bronze` |
| `silver_schema` | `silver` |
| `gold_schema` | `gold` |
| `base_path` | `/Volumes/{catalog}/raw_zone/chinook` |
| `source_catalog` | `chinook_source` |
| `source_schema` | `chinook` |

---

## Running the SCD Type 2 Demo (Version 2)

1. Run the UPDATE statements in `sql/version2_azure_sql_updates.sql` against your Azure SQL database — this changes attributes for 5 customers
2. Re-run the full Databricks Job
3. Run `sql/version2_scd2_verification.sql` in Databricks SQL Editor to confirm:
   - `dim_customer` has 64 rows (59 current + 5 expired)
   - Changed customers have 2 rows each — old expired, new current
   - `fact_sales` still joins correctly via `is_current = true`

---

## Control Tables

| Table | Purpose |
|---|---|
| `pipeline_control.pipeline_metadata_parent` | What to extract — one row per source table, `active_flag = Y/N` |
| `pipeline_control.pipeline_metadata_child` | Audit trail — one row per table per run, `source_row_count` must equal `target_row_count` |
| `pipeline_control.dqx_execution_log` | DQX summary — total, passed, and failed record counts per table per run |
| `pipeline_control.dqx_quarantine` | Failed records — stored with violation reason for review |

---

## Data Quality Rules

| Table | Column | Rule |
|---|---|---|
| customer | CustomerId, Email, Country | Not null |
| invoice | Total | > 0 |
| invoice | InvoiceDate | Not null |
| invoiceline | UnitPrice, Quantity | > 0, not null |
| track | Milliseconds, UnitPrice | > 0 |
| All tables | Primary keys | Not null |

---

## Known Considerations

**Databricks cluster outbound IP** — Azure SQL firewall must allow the cluster's outbound IP. Databricks on Azure can route through AWS IPs (`3.145.x.x`) which are not covered by the "Allow Azure services" toggle. Add an explicit firewall rule for your cluster's IP if connections fail.

**Temp view cross-session isolation** — `dbutils.notebook.run()` creates an isolated Spark session. Temp views registered in a parent notebook are not visible in child notebooks. The pipeline avoids this by writing Parquet directly in `01_extract_from_source` rather than passing DataFrames via temp views.

---

## License

MIT License — free to use, adapt, and build on.
