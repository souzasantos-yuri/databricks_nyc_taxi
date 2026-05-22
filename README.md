# NYC Taxi Data Pipeline — Databricks

End-to-end data engineering project built entirely in **Databricks**, using the NYC Yellow Taxi dataset (February 2016) to demonstrate a Medallion Architecture (Bronze → Silver → Gold) with Delta Lake and Unity Catalog.

---

## Overview

The project ingests raw NYC Taxi trip data, cleans and enriches it, and produces an aggregated analytical layer ready for dashboarding and ad-hoc querying — all within a single Databricks workspace using managed Delta tables and Databricks SQL.

---

## Architecture

```
CSV (Volume: bronze)
        │
        ▼
  ┌──────────────┐
  │    BRONZE    │  Raw ingestion — schema normalized, saved as Delta table
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │    SILVER    │  Cleaned & enriched — derived columns, quality filters, partitioned by year/month
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │     GOLD     │  Aggregated daily metrics — ready for BI/dashboards
  └──────────────┘
```

All layers are stored as **managed Delta tables** in **Unity Catalog** under `main.nyc_taxi`.

---

## Repository Structure

```
databricks_nyc_taxi/
├── extract_load_transform.ipynb   # Main ELT notebook (Bronze → Silver → Gold)
├── Queries/                       # Databricks SQL queries for analysis
└── Dashboard/                     # Databricks Dashboard exports
```

---

## Notebook: `extract_load_transform.ipynb`

The notebook is structured in clearly titled cells, each corresponding to a pipeline stage:

### 1. SETUP
Creates the Unity Catalog structure idempotently:
- Catalog: `main`
- Schema: `nyc_taxi`
- Volumes: `bronze`, `silver`, `gold`

### 2. SCHEMAS
Defines the target table names and the input CSV path:
- Input: `/Volumes/main/nyc_taxi/bronze/yellow_tripdata_2016-02.csv`
- Tables: `yellow_tripdata_2016_02_bronze`, `yellow_tripdata_2016_02_silver`, `yellow_trip_2016_02_gold_daily`

> The CSV file must be uploaded manually via the Databricks UI: **Catalog → main → nyc_taxi → Volumes → bronze → Upload**.

### 3. DATA CHECK
Uses `dbutils.fs.ls()` to verify the CSV file is present in the bronze Volume before proceeding.

### 4. BRONZE — Ingestion
- Reads the CSV with `inferSchema` and `header=True`
- Normalizes all column names to lowercase
- Saves the raw data as a **managed Delta table** (`overwrite` mode)

### 5. SILVER — Cleaning & Enrichment
- Casts columns to correct types (timestamps, doubles, integers)
- Handles column name variations across dataset versions using a helper function
- Derives new columns:
  - `pickup_date`, `year`, `month`, `day`, `hour`
  - `trip_minutes`, `trip_hours`
  - `fare_per_mile`
  - `tip_rate` (tip / fare)
  - `avg_speed_mph`
- Applies quality filters:
  - Non-null pickup and dropoff timestamps
  - Trips within February 2016
  - Positive distance and duration
  - At least 1 passenger (when present)
- Saves as a **partitioned Delta table** (`partitionBy("year", "month")`)

### 6. GOLD — Aggregation
- Groups by `pickup_date`, `vendor_id`, and `payment_type`
- Computes daily KPIs:
  - `n_trips` — total number of trips
  - `avg_miles` — average trip distance
  - `avg_minutes` — average trip duration
  - `avg_fare` — average fare amount
  - `avg_tip_rate` — average tip percentage
  - `avg_speed_mph` — average speed
- Saves as a **partitioned Delta table**, ready for consumption

### 7. VALIDATION
Runs row-count and summary queries across all three layers to confirm the pipeline ran successfully.

---

## Tech Stack

| Tool | Usage |
|---|---|
| **Databricks** | Workspace, notebooks, compute, SQL |
| **Unity Catalog** | Catalog, schema, and volume management |
| **Delta Lake** | Storage format for all managed tables |
| **PySpark** | Data transformation logic |
| **Databricks SQL** | Ad-hoc queries and dashboarding |
| **dbutils** | Volume file verification |

---

## Data Source

**NYC TLC Yellow Taxi Trip Records — February 2016**

Available at the [NYC Taxi & Limousine Commission open data portal](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page).

---

## How to Run

1. **Upload the CSV** to the bronze Volume in Databricks:
   `Catalog → main → nyc_taxi → Volumes → bronze → Upload`

2. **Open** `extract_load_transform.ipynb` in your Databricks workspace.

3. **Run all cells** in order (top to bottom). Each cell is idempotent.

4. **Explore results** using the queries in the `Queries/` folder or the dashboard in `Dashboard/`.
