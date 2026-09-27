# TPC-H Analytics Engineering Project

A dimensional data warehouse built on **Snowflake** and **dbt**, using the industry-standard TPC-H
benchmark dataset that ships with every Snowflake trial account. CI runs on **GitHub Actions**,
building and testing every model on each pull request.

## Tech Stack
- **Warehouse:** Snowflake
- **Transformation:** dbt (dbt-snowflake)
- **Testing:** dbt schema tests (uniqueness, not-null, referential integrity) + dbt_utils
- **Historical tracking:** dbt snapshots (Type 2 Slowly Changing Dimensions)
- **CI/CD:** GitHub Actions (runs `dbt run`, `dbt test`, and `dbt snapshot` on every PR)
- **Version control:** Git / GitHub

## Architecture

```
Snowflake (SNOWFLAKE_SAMPLE_DATA.TPCH_SF1)
        │
        ▼
  staging models (1:1 with source, renamed/typed columns)
        │
        ▼
  marts (dimensional model — star schema)
   ├── dim_customers   (customer + nation + region)
   └── fct_orders      (order grain, line-item revenue aggregated up)
        │
        ▼
  snapshots (Type 2 SCD on customer attributes)
```

## Project layout

```
models/
  staging/tpch/       -- 1:1 staging models + source definitions + tests
  marts/core/          -- dimensional model (dim_customers, fct_orders) + tests
snapshots/
  customers_snapshot.sql   -- Type 2 SCD snapshot on the customer dimension
.github/workflows/
  dbt_ci.yml            -- CI: dbt run + dbt test + dbt snapshot on every PR
profiles.yml.example    -- template for local/CI Snowflake connection
```

## Setup

1. Create a free Snowflake trial account at https://trial.snowflake.com — this automatically
   gives you read access to `SNOWFLAKE_SAMPLE_DATA.TPCH_SF1`, no data loading required.
2. `pip install dbt-snowflake`
3. Copy `profiles.yml.example` to `~/.dbt/profiles.yml` and fill in your Snowflake credentials
   (or export them as environment variables — the template reads from env vars either way).
4. `dbt deps` to install packages.
5. `dbt run` to build the staging and marts models.
6. `dbt test` to run all schema tests.
7. `dbt snapshot` to take the first snapshot of the customer dimension.

## CI/CD

Every pull request and push to `main` triggers `.github/workflows/dbt_ci.yml`, which installs
dbt, runs all models, runs all tests, and takes a snapshot — using Snowflake credentials stored
as GitHub repository secrets (`SNOWFLAKE_ACCOUNT`, `SNOWFLAKE_USER`, `SNOWFLAKE_PASSWORD`, etc.).
This mirrors a real analytics-engineering CI pattern: nothing merges to `main` without a clean
build and passing tests.

## A note on the SCD snapshot

TPC-H is a static benchmark dataset — it doesn't change between runs, so running `dbt snapshot`
repeatedly against it won't show any historical versions by itself. The snapshot logic itself is
fully real and production-correct (`strategy='check'` on `market_segment`, `account_balance`,
`address`, and `phone`), demonstrating the exact Type 2 SCD mechanism used in production dbt
projects.

To demonstrate it actually capturing a change (useful for a portfolio walkthrough or interview),
manually update a row in your own copy of the customer table between two snapshot runs, e.g.:

```sql
update your_database.your_schema.stg_tpch_customers_copy
set market_segment = 'AUTOMOBILE'
where customer_id = 1;
```

then run `dbt snapshot` again — the snapshot table will now show two rows for that customer,
with `dbt_valid_from` / `dbt_valid_to` columns marking the change over time, which is exactly
how Type 2 SCD history is tracked in a real warehouse.

## Possible extensions
- Add a `mart_finance` layer that reconciles order totals against a synthetic "payments" source,
  mirroring FX/transaction reconciliation work.
- Swap in a dataset with real historical changes (e.g. Kaggle's Olist e-commerce dataset) to see
  the snapshot mechanism capture genuine history without the manual-update workaround above.
