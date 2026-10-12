# 02 · Beginner: Models, ref(), source(), Seeds and Materializations

## 1. Models

A model is a `.sql` file with **one `SELECT`** and nothing else. The file name becomes the table or view name.

```sql
-- models/marts/dim_customers.sql
with customers as (
    select * from {{ ref('stg_customers') }}
),

orders as (
    select * from {{ ref('stg_orders') }}
),

customer_orders as (
    select
        customer_id,
        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders,
        sum(case when status = 'completed' then amount else 0 end) as lifetime_value
    from orders
    group by customer_id
)

select
    c.customer_id,
    c.full_name,
    c.email,
    c.signup_date,
    co.first_order_date,
    co.most_recent_order_date,
    coalesce(co.number_of_orders, 0) as number_of_orders,
    coalesce(co.lifetime_value, 0)   as lifetime_value
from customers c
left join customer_orders co using (customer_id)
```

**Rules of thumb**
- No `CREATE`, `INSERT` or semicolons. dbt writes the DDL for you.
- Use **CTEs** at the top for each input, then a final `select`.
- One model, one purpose.

---

## 2. `ref()`: connect models together

```sql
select * from {{ ref('stg_orders') }}
```

`ref()` does three things:
1. Builds the **dependency graph**, so dbt runs models in the right order.
2. Resolves the **correct schema and name** for the environment (dev schema versus prod schema).
3. Powers **lineage** in the docs.

Never hard-code `analytics.stg_orders`. Always use `ref()`.

---

## 3. `source()`: declare raw tables

Raw data loaded by other tools should be declared as **sources**, not hard-coded.

`models/staging/_sources.yml`:

```yaml
version: 2

sources:
  - name: raw
    schema: main            # where the raw tables live
    tables:
      - name: customers
      - name: orders
```

Use it in a model:

```sql
select * from {{ source('raw', 'orders') }}
```

**Why bother?**
- A single place to rename or relocate raw tables
- Lineage starts at the true origin
- You can test and freshness-check raw data (file 06)

In this guide we load raw data as seeds and use `ref()`. In real projects, raw data comes from your loader and you use `source()`.

---

## 4. Seeds

Seeds are **CSV files in your repo** that dbt loads into the warehouse with `dbt seed`.

Good for: country codes, status mappings, small lookup tables.
Not good for: large or frequently changing data.

Control column types in `dbt_project.yml`:

```yaml
seeds:
  shop_analytics:
    raw_orders:
      +column_types:
        order_id: integer
        amount: decimal(10,2)
```

---

## 5. Materializations

A **materialization** decides how dbt builds the model.

| Type | Builds | Pros | Cons | Use when |
|------|--------|------|------|----------|
| `view` (default) | A view | Always fresh, no storage | Slow if stacked or heavy | Light transforms, staging |
| `table` | A table (rebuilt each run) | Fast to query | Rebuild cost each run | Marts, heavy joins |
| `incremental` | Table, only new rows added | Cheap on big data | More logic, more risk | Large event tables (file 05) |
| `ephemeral` | Nothing (inlined as a CTE) | No clutter in warehouse | Cannot query directly | Tiny helper logic |
| `materialized_view` | Warehouse materialized view | Warehouse keeps it fresh | Adapter-specific | Where supported |

Set it **per model**:

```sql
{{ config(materialized='table') }}

select ...
```

Or **per folder** in `dbt_project.yml`:

```yaml
models:
  shop_analytics:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

A model-level `config()` overrides the folder setting.

---

## 6. Layering your project

A common, battle-tested structure:

```
models/
├── staging/          # 1:1 with source tables. Rename, cast, clean. Nothing else.
│   ├── _sources.yml
│   ├── stg_customers.sql
│   └── stg_orders.sql
├── intermediate/     # Joins and reshaping needed by several marts
│   └── int_orders_enriched.sql
└── marts/            # Business-facing tables
    ├── dim_customers.sql
    └── fct_orders.sql
```

**Naming conventions**
- `stg_<source>__<entity>` (for example `stg_shop__orders`), or simply `stg_orders` in small projects
- `int_<description>` for intermediate
- `dim_` for dimensions, `fct_` for facts
- `snake_case` everywhere; plural entity names are a common convention

**Staging rules**
- Only select from `source()` or seeds
- Rename columns to a consistent style
- Cast types, standardize booleans and timestamps
- No joins, no aggregation
- Materialize as views

---

## 7. A fact model

`models/marts/fct_orders.sql`:

```sql
select
    o.order_id,
    o.customer_id,
    o.order_date,
    o.status,
    o.amount,
    c.full_name as customer_name
from {{ ref('stg_orders') }} o
left join {{ ref('stg_customers') }} c using (customer_id)
```

Build the whole project:

```bash
dbt run
```

---

## 8. Running only part of the project

```bash
dbt run --select stg_orders              # one model
dbt run --select stg_orders+             # the model and everything downstream
dbt run --select +fct_orders             # the model and everything upstream
dbt run --select staging                 # a folder
dbt run --select tag:daily               # by tag
dbt run --exclude fct_orders             # everything except one
```

---

## 9. Exercises

1. Create `models/staging/_sources.yml` for the raw tables and switch one staging model from `ref()` to `source()`.
2. Make `marts` build as tables and `staging` as views through `dbt_project.yml`. Confirm with `dbt run` and check the log output.
3. Create `int_orders_enriched.sql` that joins orders to customers, then rebuild `fct_orders` on top of it.
4. Use `dbt run --select +fct_orders` and list which models run.
5. Add `{{ config(materialized='ephemeral') }}` to a small model. Find it in `target/compiled/` inside its downstream model.

**Next:** `03_beginner_tests_and_docs.md`
