# Tel Aviv Municipality Job Summary

## Overview
Automated data pipeline that processes Tel Aviv business registry and street closure data to calculate daily compensation for businesses affected by street closures. Implements a medallion architecture (Bronze → Silver → Gold → KPI) with 8 orchestrated tasks running as a Databricks job.

## Schedule
* **Frequency**: Daily at 9:00:00 AM
* **Timezone**: Asia/Jerusalem (Israel Standard Time)

* **Status**: ⏸️ Currently PAUSED
* **Concurrency**: Max 1 run at a time

## Architecture Pattern
**Medallion Architecture**: Raw data ingestion → Data cleansing → Dimensional modeling → Business metrics

```
Bronze (Raw)  →  Silver (Cleansed)  →  Gold (Dimensional)  →  KPI (Aggregations)
```

## Pipeline Tasks (8 Total)

### 🔵 Bronze Layer - Data Ingestion
Ingest raw data from external sources with full audit trail.

1. **api_source** (`api_src` notebook)
   * Ingests business registry data from Tel Aviv ArcGIS REST API
   * Merge-based ingestion with event_id deduplication
   * Output: `workspace.tel_aviv_municipality_raw.business`

2. **storage_src** (`google_storage_src` notebook)
   * Ingests street closure data from CSV in Google Cloud Storage
   * Reads from: `/Volumes/workspace/default/external/rechov_sagur.csv`
   * Output: `workspace.tel_aviv_municipality_raw.street_closures`

### 🟢 Silver Layer - Data Cleansing
Parse JSON, cleanse data, handle nulls, deduplicate.

3. **business_stg** (`business_stg` notebook)
   * **Depends on**: api_source
   * Parses JSON payloads, cleanses street names, deduplicates businesses
   * Removes symbols from street names for consistent joins
   * Output: `workspace.tel_aviv_municipality_stg.business`

4. **street_stg** (`street_closures_stg` notebook)
   * **Depends on**: storage_src
   * Parses JSON, cleanses street names, extracts time ranges
   * Normalizes from/to street pairs into separate rows
   * Output: `workspace.tel_aviv_municipality_stg.street_closures`

### 🟡 Gold Layer - Dimensional Model
Build star schema with dimensions and fact tables.

5. **dim_street** (`dim_street` notebook)
   * **Depends on**: street_stg
   * Master street reference dimension
   * Merge-based updates preserve surrogate key stability
   * Output: `workspace.tel_aviv_municipality.dim_street`

6. **dim_business** (`dim_business` notebook)
   * **Depends on**: business_stg, street_stg
   * Business dimension with SCD Type 2 (historical tracking)
   * Tracks changes to holder names, addresses, areas over time
   * Output: `workspace.tel_aviv_municipality.dim_business`

7. **compensation_fact** (`facts` notebook)
   * **Depends on**: dim_street, dim_business, business_stg
   * Daily business compensation fact table (star schema grain)
   * Joins businesses with closures on street name
   * Calculates daily compensation (10% of property area in ILS)
   * Output: `workspace.tel_aviv_municipality.fact_daily_business_compensation`

### 🟣 KPI Layer - Business Aggregations
Aggregate metrics for reporting and dashboards.

8. **kpi_2023_agg_compensation_payments** (`kpi_2023_agg_compensation_payments` notebook)
   * **Depends on**: compensation_fact
   * Pre-aggregated compensation metrics for 2023
   * Optimized for dashboard queries and reporting
   * Output: `workspace.tel_aviv_municipality.kpi_2023_agg_compensation_payments`

## Task Dependencies (DAG)

```
         ┌─────────────┐              ┌─────────────┐
         │ api_source  │              │ storage_src │
         │  (Bronze)   │              │  (Bronze)   │
         └──────┬──────┘              └──────┬──────┘
                │                            │
                ▼                            ▼
         ┌─────────────┐              ┌─────────────┐
         │business_stg │              │ street_stg  │
         │  (Silver)   │              │  (Silver)   │
         └──────┬──────┘              └──────┬──────┘
                │                            │
                │        ┌───────────────────┴─────────┐
                │        │                             │
                │        ▼                             ▼
                │  ┌─────────────┐            ┌─────────────┐
                └─▶│dim_business │            │ dim_street  │
                   │   (Gold)    │            │   (Gold)    │
                   └──────┬──────┘            └──────┬──────┘
                          │                          │
                          └────────┬─────────────────┘
                                   │
                                   ▼
                          ┌─────────────────┐
                          │compensation_fact│
                          │     (Gold)      │
                          └────────┬────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │kpi_2023_agg_compensation_    │
                    │         payments (KPI)       │
                    └──────────────────────────────┘
```

**Execution Order**:
1. **Parallel**: api_source + storage_src (independent ingestion)
2. **Parallel**: business_stg + street_stg (depends on respective sources)
3. **Parallel**: dim_business + dim_street (depends on silver layer)
4. **Sequential**: compensation_fact (depends on both dimensions)
5. **Sequential**: kpi_2023_agg_compensation_payments (depends on fact)


## Output Tables

### Bronze Layer
* `workspace.tel_aviv_municipality_raw.business` - Raw business registry (JSON payloads)
* `workspace.tel_aviv_municipality_raw.street_closures` - Raw closure data (JSON payloads)

### Silver Layer
* `workspace.tel_aviv_municipality_stg.business` - Cleansed business data (structured)
* `workspace.tel_aviv_municipality_stg.street_closures` - Cleansed closure data (structured)

### Gold Layer (Dimensional Model)
* `workspace.tel_aviv_municipality.dim_street` - Street dimension (master reference)
* `workspace.tel_aviv_municipality.dim_business` - Business dimension (SCD Type 2)
* `workspace.tel_aviv_municipality.fact_daily_business_compensation` - Daily compensation fact

### KPI Layer
* `workspace.tel_aviv_municipality.kpi_2023_agg_compensation_payments` - 2023 compensation aggregations

