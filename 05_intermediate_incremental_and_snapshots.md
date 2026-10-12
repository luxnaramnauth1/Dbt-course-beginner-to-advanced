# 05 · Intermediate: Incremental Models and Snapshots

## Part A: Incremental models

### 1. The problem

A `table` model rebuilds **everything** on every run. For a 2-billion-row event table, that is slow and expensive. An **incremental** model builds the full table once, then on later runs only processes **new or changed rows**.

### 2. The basic pattern

```sql
{{
    config(
        materialized='incremental',
        unique_key='order_id'
    )
}}

select
    order_id,
    customer_id,
    order_date,
    status,
    amount,
    updated_at
from {{ ref('stg_orders') }}

{% if is_incremental() %}
  -- only runs on incremental runs, not the first build
  where updated_at > (select max(updated_at) from {{ this }})
{% endif %}
```

How it behaves:

| Run | `is_incremental()` | What happens |
|-----|--------------------|--------------|
| First run (table does not exist) | False | Builds the full table |
| Later runs | True | Selects only new rows, then inserts or merges |
| `dbt run --full-refresh` | False | Drops and rebuilds everything |

`{{ this }}` refers to the model's own existing table.

### 3. `unique_key`

- **No `unique_key`:** new rows are just **appended**. Re-processing causes duplicates.
- **With `unique_key`:** existing rows with the same key are **updated** (merge or delete then insert, depending on strategy). Use a composite key if needed.

### 4. Incremental strategies

Available strategies depend on the adapter. Common ones:

| Strategy | Behaviour | Typical use |
|----------|-----------|-------------|
| `append` | Insert new rows only | Immutable event logs |
| `merge` | Upsert by `unique_key` | Rows can change |
| `delete+insert` | Delete matching keys, insert fresh | Adapters without merge |
| `insert_overwrite` | Replace whole partitions | Partitioned big tables (BigQuery, Spark) |
| `microbatch` | Process time-sliced batches independently | Large time-series data (dbt 1.9+) |

Choose with:

```sql
{{ config(materialized='incremental', incremental_strategy='merge', unique_key='order_id') }}
```

Check your adapter's docs for which strategies it supports and its defaults.

### 5. Handling late-arriving data

Filtering on `max(updated_at)` misses rows that arrive late with an older timestamp. Use a **lookback window**:

```sql
{% if is_incremental() %}
  where updated_at > (
      select dateadd('day', -3, max(updated_at)) from {{ this }}
  )
{% endif %}
```

Reprocessing three days is fine with `unique_key` set, since duplicates are merged. (Date function syntax varies by warehouse. Use `dbt.dateadd` for portability.)

### 6. Schema changes

When you add a column to an incremental model, the existing table does not have it:

```sql
{{ config(materialized='incremental', unique_key='order_id', on_schema_change='append_new_columns') }}
```

Options: `ignore` (default), `fail`, `append_new_columns`, `sync_all_columns`. For big structural changes, run `--full-refresh`.

### 7. Microbatch (dbt 1.9+)

Instead of writing the `is_incremental()` filter yourself, microbatch splits work into time batches and handles filtering, retries and backfills for you.

```sql
{{
    config(
        materialized='incremental',
        incremental_strategy='microbatch',
        event_time='order_ts',
        batch_size='day',
        begin='2025-01-01',
        lookback=2
    )
}}

select * from {{ ref('stg_orders') }}
```

The upstream model also needs an `event_time` configured so dbt can filter inputs by batch. You can backfill a range with:

```bash
dbt run --select fct_orders --event-time-start "2025-03-01" --event-time-end "2025-03-15"
```

Microbatch is newer. Read the current docs for supported adapters and options before using it in production.

### 8. Pitfalls

- **Forgetting that logic changes do not rewrite old rows.** Change a calculation and old rows keep old values until `--full-refresh`.
- **No `unique_key` plus reprocessing = duplicates.**
- **Non-deterministic filters** (`current_date`) make backfills hard.
- **Always test** the model: `unique` on the key and a row-count sanity check against the source.
- Schedule a periodic **full refresh** for models where drift is a risk.

### 9. When to go incremental

Use it when full rebuilds are genuinely too slow or costly. Start with `table`. Move to `incremental` only when you have a reason. Simple beats clever.

---

## Part B: Snapshots (tracking history, SCD Type 2)

### 1. The problem

Source systems overwrite data. If a customer changes plan from `basic` to `premium`, the old value is gone. A **snapshot** records every version of a row over time, a **Slowly Changing Dimension Type 2**.

### 2. How snapshots look

Snapshots add four columns:

| Column | Meaning |
|--------|---------|
| `dbt_scd_id` | Unique ID per row version |
| `dbt_updated_at` | Timestamp used to detect the change |
| `dbt_valid_from` | When this version became current |
| `dbt_valid_to` | When it was replaced (`NULL` = current) |

Example result for customer 7 after two changes:

| customer_id | plan | dbt_valid_from | dbt_valid_to |
|-------------|------|----------------|--------------|
| 7 | basic | 2025-01-01 | 2025-03-10 |
| 7 | premium | 2025-03-10 | `NULL` |

### 3. Defining a snapshot

**YAML style (dbt 1.9+):**

`snapshots/customers_snapshot.yml`:

```yaml
snapshots:
  - name: customers_snapshot
    relation: source('raw', 'customers')
    config:
      schema: snapshots
      unique_key: customer_id
      strategy: timestamp
      updated_at: updated_at
```

**Classic SQL style (works on older versions, still supported):**

`snapshots/customers_snapshot.sql`:

```sql
{% snapshot customers_snapshot %}

{{
    config(
      target_schema='snapshots',
      unique_key='customer_id',
      strategy='timestamp',
      updated_at='updated_at'
    )
}}

select * from {{ source('raw', 'customers') }}

{% endsnapshot %}
```

### 4. Strategies

- **`timestamp`** (preferred): uses a reliable `updated_at` column. Cheap and accurate.
- **`check`**: compares chosen columns. Use when there is no `updated_at`.

```yaml
config:
  strategy: check
  check_cols: [plan, email]      # or 'all'
```

### 5. Running snapshots

```bash
dbt snapshot
```

Run snapshots **on a schedule** (for example daily). A snapshot only captures what it sees when it runs. If a value changes twice between runs, you only see the final one.

### 6. Using snapshot data

Point-in-time join:

```sql
select
    o.order_id,
    o.order_date,
    c.plan as plan_at_order_time
from {{ ref('stg_orders') }} o
left join {{ ref('customers_snapshot') }} c
  on  o.customer_id = c.customer_id
  and o.order_date >= c.dbt_valid_from
  and (o.order_date < c.dbt_valid_to or c.dbt_valid_to is null)
```

### 7. Best practices

- Snapshot **sources**, not heavily transformed models, so history reflects raw truth
- Select as few columns as you need, but ensure a stable unique key
- Never delete snapshot tables casually. History cannot be rebuilt from today's data.
- Add tests: `unique` on `dbt_scd_id`, and a custom check that only one current row exists per key

---

## Exercises

1. Convert `fct_orders` to incremental with `unique_key='order_id'`. Run it, insert new rows into the seed, run again and inspect the log for row counts.
2. Add a three-day lookback. Explain why it needs `unique_key`.
3. Run `dbt run --full-refresh --select fct_orders` and compare log output with a normal run.
4. Create a snapshot of customers. Change a customer in the seed (change `email`), run `dbt seed` then `dbt snapshot`, and inspect the history rows.
5. Write the point-in-time join above against your snapshot.
6. (Stretch) Read the current docs for microbatch and rewrite your incremental model with it.

**Next:** `06_intermediate_freshness_and_unit_tests.md`
