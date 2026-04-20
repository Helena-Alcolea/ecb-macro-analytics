# ecb-macro-analytics
# ECB Exchange Rates ETL Pipeline

End-to-end data pipeline for extracting, transforming, and storing daily exchange rate data from the European Central Bank (ECB), with a reporting layer connected via Power BI.

---

## Overview

This project implements a two-stage ETL pipeline that retrieves official exchange rate time series from the ECB Data API (SDMX), processes the XML responses into structured tabular data, and loads them into a PostgreSQL database. A Power BI dashboard connects directly to the database to support exploratory analysis and reporting.

The pipeline covers 16 currencies against the euro (USD, GBP, JPY, CHF, AUD, CAD, NZD, SEK, NOK, DKK, PLN, CZK, HUF, RON, BGN, ISK), with daily granularity from the full historical series available via the ECB API.

---

## Architecture

```
ECB Data API (SDMX)
        │
        ▼
  HTTP request (requests)
        │
        ▼
  XML parsing (ElementTree)
        │
        ▼
  In-memory CSV buffer (StringIO)
        │
        ▼
  PostgreSQL (psycopg)
  ┌─────────────────────┐
  │  exchange_rates     │  ← fact table
  │  dim_currency       │  ← dimension table
  └─────────────────────┘
        │
        ▼
  Power BI (DirectQuery)
```

---

## Database Schema

### `exchange_rates` (fact table)

| Column     | Type          | Description                              |
|------------|---------------|------------------------------------------|
| `id`       | SERIAL PK     | Surrogate key, auto-generated            |
| `date`     | DATE NOT NULL | Observation date                         |
| `currency` | VARCHAR(3)    | ISO 4217 currency code                   |
| `rate`     | NUMERIC       | Exchange rate against EUR                |

Unique constraint on `(date, currency)` — prevents duplicate observations and enables safe upserts.

### `dim_currency` (dimension table)

| Column          | Type         | Description                     |
|-----------------|--------------|---------------------------------|
| `id`            | VARCHAR(3) PK | ISO 4217 currency code          |
| `currency_name` | VARCHAR(50)  | Full currency name               |
| `country`       | VARCHAR(50)  | Issuing country or economic area |

---

## ETL Workflow

### Stage 1 — Initial load (`1-ecb_exchange_rates_initial_load.ipynb`)

Runs once to set up the database from scratch:

1. Creates the `exchange_rates` fact table if it does not exist.
2. Extracts the full historical exchange rate series from the ECB API.
3. Parses the SDMX XML response and writes records to an in-memory buffer.
4. Loads all records into PostgreSQL using an upsert strategy.
5. Creates and populates the `dim_currency` dimension table from the distinct currency codes in the fact table.

### Stage 2 — Incremental update (`2-ecb_exchange_rates_incremental_update.ipynb`)

Runs periodically to keep the database current:

1. Queries the latest date stored in `exchange_rates`.
2. Requests only new observations from the ECB API (from the next missing date onward).
3. Parses, transforms, and loads new records using the same upsert logic.

The update function checks whether the ECB has published new data since the last stored observation. If not (i.e., the gap is ≤ 1 day), it returns a warning and does not attempt an empty load.

---

## Key Design Decisions

**Upsert over insert.** Both load functions use `ON CONFLICT (date, currency) DO UPDATE` instead of plain `INSERT`. This makes the pipeline idempotent: re-running it on already-loaded data produces no duplicates and no errors.

**In-memory buffer.** The XML response is parsed directly into a `StringIO` buffer rather than writing intermediate files to disk. This reduces I/O overhead and keeps the pipeline stateless between runs.

**Incremental extraction by date.** The update notebook queries the database for the latest stored date before calling the API, and restricts the request to `startPeriod={next_date}`. This avoids re-downloading the full historical series on every run.

**Credentials excluded from repository.** Database connection parameters are intentionally left empty in the notebooks. Screenshots of the Power BI dashboard are included to demonstrate the reporting layer without exposing connection details.

---

## Tech Stack

| Layer         | Technology                        |
|---------------|-----------------------------------|
| Extraction    | Python · requests · ECB SDMX API  |
| Parsing       | xml.etree.ElementTree · StringIO  |
| Transformation| Pandas (EDA notebooks)            |
| Storage       | PostgreSQL · psycopg              |
| Reporting     | Power BI (DirectQuery)            |
| Environment   | Jupyter Notebooks · Git           |

---

## Repository Structure

```
ecb-macro-analytics/
│
├── 1-ecb_exchange_rates_initial_load.ipynb    # Full historical load
├── 2-ecb_exchange_rates_incremental_update.ipynb  # Periodic update
├── screenshots/                               # Power BI dashboard previews
└── README.md
```

---

## Data Source

[ECB Data Portal](https://data.ecb.europa.eu/) — European Central Bank Statistical Data Warehouse, accessed via the [ECB Data API (SDMX 2.1)](https://data-api.ecb.europa.eu/service/).

Dataset used: `EXR` — Daily exchange rates of selected currencies against the euro.


