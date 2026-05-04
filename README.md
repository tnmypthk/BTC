# Bitcoin Whale Alert — Snowflake + dbt Pipeline

An ELT pipeline built with **dbt Core** and **Snowflake** that ingests raw Bitcoin blockchain transaction data, parses semi-structured JSON outputs, and identifies **whale wallets** — addresses responsible for large-volume BTC transfers. Includes incremental loading, a custom audit trail, source freshness monitoring, and CI/CD via GitHub Actions.

---

## What Is a Whale Alert?

In crypto markets, a "whale" is a wallet address that moves large volumes of Bitcoin. This pipeline detects wallets that have sent **more than 10 BTC** in total, ranks them by volume, and converts their totals to USD — giving a view of where significant on-chain value is flowing.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Cloud Data Warehouse | Snowflake |
| Transformation | dbt Core |
| Package Management | dbt-utils |
| CI/CD | GitHub Actions (slim CI with state) |
| Data Format | Semi-structured JSON (Bitcoin transaction outputs) |
| Language | SQL, Jinja |

---

## Architecture

```
Snowflake Raw Source (btc.btc_schema.btc)
            │
            ▼
    stg_btc             ← Incremental merge on HASH_KEY, tracks BLOCK_TIMESTAMP
            │
            ▼
    stg_btc_outputs     ← LATERAL FLATTEN to unnest JSON outputs array
            │               Extracts output_address + output_value per transaction
            ▼
    whale_alert_v1      ← Wallets sending >10 BTC, ranked by total volume + USD value
    whale_alert_v2      ← Same logic, BTC only (no USD conversion)
```

---

## Key dbt Features Used

**Incremental models — two strategies**
- `stg_btc` uses `merge` on `HASH_KEY` to deduplicate and upsert new blocks
- `stg_btc_outputs` uses `append` to add new flattened output rows as blocks arrive

**Semi-structured data parsing**
- Bitcoin transaction `outputs` is a JSON array stored as a Snowflake VARIANT column
- `LATERAL FLATTEN` unpacks each output into its own row, extracting `address` and `value`

**Custom macros**
- `convert_to_usd(amount)` — converts BTC values to USD inline in the mart query
- `log_dbt_run()` — audit macro that inserts a row into `BTC.AUDIT.dbt_audit` on every run, capturing invocation ID, run timestamp, dbt command, target profile, user, and dbt version

**Source freshness monitoring**
- Warns if Bitcoin source data is more than 1 hour stale
- Errors if data is more than 3 hours stale
- Enforces data reliability without manual checks

**Invocation ID tracking**
- Every model embeds `{{ invocation_id }}` as a column, linking every output row back to the exact dbt run that produced it — full lineage at the row level

**SCD Type 2 snapshot**
- `customer_snapshots` tracks historical changes to `CUSTOMER_STATUS` using a timestamp strategy on `LAST_UPDATED`, with `hard_deletes: 'new_record'` to capture deletions as new rows

**Slim CI**
- GitHub Actions workflow uses `dbt state:modified` to only build and test models that changed in a pull request — keeps CI fast on a large project

---

## Model Reference

| Model | Type | Materialization | Description |
|---|---|---|---|
| `stg_btc` | Staging | Incremental (merge) | Raw BTC transactions, deduplicated on HASH_KEY |
| `stg_btc_outputs` | Staging | Incremental (append) | Flattened transaction outputs — one row per recipient address |
| `whale_alert_v1` | Mart | Table | Whale wallets with BTC volume + USD equivalent |
| `whale_alert_v2` | Mart | Table | Whale wallets with BTC volume only |
| `customer_snapshots` | Snapshot | SCD Type 2 | Historical customer status with hard delete tracking |

---

## Audit Trail

Every dbt run is logged to `BTC.AUDIT.dbt_audit`:

```sql
select * from BTC.AUDIT.dbt_audit order by run_started_at desc;
```

| column | description |
|---|---|
| `invocation_id` | Unique ID for the dbt run — matches the column in every model |
| `run_started_at` | Timestamp the run began |
| `dbt_command` | The exact command executed (e.g. `dbt build`) |
| `target_profile` | Which dbt profile was used |
| `target_name` | Environment (dev / prod) |
| `target_user` | Snowflake user that ran the job |
| `dbt_version` | dbt version at time of run |

---

## Getting Started

### Prerequisites

- Snowflake account with access to Bitcoin blockchain data
- Python 3.8+
- dbt Core: `pip install dbt-snowflake`

### Setup

```bash
git clone https://github.com/tnmypthk/BTC.git
cd BTC

# Install dbt packages
dbt deps

# Add your Snowflake connection to ~/.dbt/profiles.yml (see below)

# Run all models
dbt build

# Check source freshness
dbt source freshness
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
      database: btc
      warehouse: <your_warehouse>
      schema: dev
      threads: 4
```

---

## CI/CD Pipeline

Pull requests trigger a GitHub Actions workflow that:

1. Runs `dbt deps` to install packages
2. Compiles the project
3. Uses `dbt state:modified+` to build and test only changed models
4. Fails the PR if any test breaks

---

## What I Learned

- Parsing semi-structured JSON in Snowflake using `LATERAL FLATTEN` on VARIANT columns
- Building incremental models with both `merge` and `append` strategies
- Writing reusable Jinja macros for currency conversion and audit logging
- Implementing row-level lineage tracking via `invocation_id`
- Configuring source freshness thresholds to enforce data SLAs
- Setting up SCD Type 2 snapshots with hard delete tracking
- Running slim CI with dbt state comparison to keep GitHub Actions fast

---

## About

A hands-on learning project applying Snowflake and dbt to real Bitcoin blockchain data. Built to develop practical analytics engineering skills across incremental modelling, semi-structured data, macros, and CI/CD.

**Author:** Tanmay Pathak · [@tnmypthk](https://github.com/tnmypthk)
