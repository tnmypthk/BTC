# Bitcoin Price Analytics — Snowflake + dbt Pipeline

A modular ELT pipeline built with **dbt Core** and **Snowflake** that ingests raw Bitcoin price and market data, transforms it through a layered data model, and produces analytics-ready mart tables. Includes CI/CD via GitHub Actions with dbt slim CI using state comparison.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Cloud Data Warehouse | Snowflake |
| Transformation | dbt Core |
| Package Management | dbt-utils |
| CI/CD | GitHub Actions |
| Data Source | Bitcoin market data (CSV / AWS S3) |
| Version Control | Git |

---

## Project Architecture

Raw data is ingested into Snowflake and transformed through a standard dbt layered architecture:

```
Raw Source (Snowflake Stage)
        │
        ▼
  Staging Models          ← Rename columns, cast types, light cleaning
        │
        ▼
   Mart Models            ← Business logic, aggregations, analytics-ready tables
   (materialised as tables)
```

### Folder Structure

```
BTC/
├── .github/
│   └── workflows/          # GitHub Actions CI/CD — dbt slim CI with state
├── models/
│   ├── staging/            # stg_* models: source cleaning and typing
│   └── marts/              # Aggregated, analytics-ready Bitcoin tables
├── macros/                 # Reusable Jinja macros (e.g. date utilities)
├── seeds/                  # Static reference data loaded via dbt seed
├── snapshots/              # SCD Type 2 snapshots for slowly changing data
├── tests/                  # Custom singular tests
├── analyses/               # Ad-hoc analytical SQL (not materialised)
├── state/                  # dbt state artefacts for slim CI diffing
├── packages.yml            # dbt package dependencies (dbt-utils)
├── package-lock.yml        # Locked package versions
└── dbt_project.yml         # Project configuration
```

---

## Key dbt Features Used

- **Sources & freshness** — `sources.yml` defines Snowflake source tables with lineage tracking
- **Staging / Mart layering** — clean separation between raw cleaning and business logic
- **dbt macros** — reusable Jinja templating for date logic and transformations
- **dbt-utils package** — surrogate keys, date spine, and test utilities
- **Generic + singular tests** — `not_null`, `unique`, and custom data quality assertions
- **Snapshots** — SCD Type 2 history tracking for slowly changing records
- **Slim CI** — GitHub Actions workflow uses `dbt state:modified` to only run changed models on pull requests, reducing CI run time

---

## CI/CD Pipeline

Pull requests trigger a GitHub Actions workflow that:

1. Installs dbt and dependencies (`dbt deps`)
2. Compiles the project (`dbt compile`)
3. Runs only modified models using dbt state comparison (`dbt build --select state:modified+`)
4. Reports test results

This ensures fast feedback on PRs without rebuilding the entire project.

---

## Getting Started

### Prerequisites

- Snowflake account (free trial works)
- Python 3.8+
- dbt Core installed: `pip install dbt-snowflake`

### Setup

```bash
# Clone the repo
git clone https://github.com/tnmypthk/BTC.git
cd BTC

# Install dbt packages
dbt deps

# Configure your Snowflake connection
# Add a profiles.yml to ~/.dbt/ with your Snowflake credentials
# See: https://docs.getdbt.com/docs/core/connect-data-platform/snowflake-setup

# Run all models
dbt run

# Run tests
dbt test

# Build everything (run + test)
dbt build
```

### profiles.yml example

```yaml
my_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: <your_account>
      user: <your_user>
      password: <your_password>
      role: <your_role>
      database: <your_database>
      warehouse: <your_warehouse>
      schema: dev
      threads: 4
```

---

## What I Learned

- Designing a layered dbt architecture (staging → marts) for financial time-series data
- Writing reusable Jinja macros to avoid repetition across models
- Configuring GitHub Actions for dbt CI/CD with slim state-based diffing
- Using dbt packages (`dbt-utils`) for surrogate keys and testing utilities
- Managing Snowflake stages for bulk data ingestion from S3 and local files
- Implementing SCD Type 2 snapshots for historical tracking

---

## About

Built as a hands-on learning project to apply Snowflake and dbt skills on real Bitcoin market data. Part of an ongoing journey into analytics engineering and modern data stack tooling.

**Author:** Tanmay Pathak · [@tnmypthk](https://github.com/tnmypthk)
