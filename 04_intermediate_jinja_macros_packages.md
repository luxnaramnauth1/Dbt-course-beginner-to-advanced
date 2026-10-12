# 04 · Intermediate: Jinja, Macros, Variables, Hooks and Packages

## 1. Jinja in one page

dbt uses **Jinja**, a templating language, on top of SQL. Jinja runs **before** the SQL is sent to the warehouse. The warehouse never sees Jinja, only the compiled result.

Three delimiters:

| Syntax | Purpose | Example |
|--------|---------|---------|
| `{{ ... }}` | Output an expression | `{{ ref('stg_orders') }}` |
| `{% ... %}` | Statements (logic) | `{% if ... %}`, `{% for ... %}` |
| `{# ... #}` | Comments (removed) | `{# note to self #}` |

### Variables

```sql
{% set payment_methods = ['credit_card', 'bank_transfer', 'gift_card'] %}
```

### Loops

Generate repetitive SQL:

```sql
select
    order_id,
    {% for method in payment_methods %}
    sum(case when method = '{{ method }}' then amount else 0 end) as {{ method }}_amount
    {% if not loop.last %},{% endif %}
    {% endfor %}
from {{ ref('stg_payments') }}
group by order_id
```

Compiled output (check `target/compiled/`):

```sql
select
    order_id,
    sum(case when method = 'credit_card' then amount else 0 end) as credit_card_amount,
    sum(case when method = 'bank_transfer' then amount else 0 end) as bank_transfer_amount,
    sum(case when method = 'gift_card' then amount else 0 end) as gift_card_amount
from main.stg_payments
group by order_id
```

### Conditionals

```sql
{% if target.name == 'dev' %}
    limit 1000
{% endif %}
```

### Whitespace control

A dash trims whitespace: `{%- ... -%}`. It only affects readability of compiled SQL, not correctness.

---

## 2. Useful built-in context

| Object | What it gives you |
|--------|-------------------|
| `target.name` | Current target (dev, prod) |
| `target.schema` | Schema for this run |
| `target.type` | Adapter type (duckdb, snowflake ...) |
| `this` | The current model's relation (used in incremental logic) |
| `var('x')` | A project variable |
| `env_var('X')` | An environment variable |
| `run_started_at` | Timestamp the run began |
| `invocation_id` | Unique ID of this dbt invocation |

---

## 3. Macros

A **macro** is a reusable block of Jinja, like a function. Put macros in `macros/`.

`macros/cents_to_dollars.sql`:

```sql
{% macro cents_to_dollars(column_name, scale=2) %}
    round({{ column_name }} / 100.0, {{ scale }})
{% endmacro %}
```

Use it:

```sql
select
    order_id,
    {{ cents_to_dollars('amount_cents') }} as amount_dollars
from {{ ref('stg_payments') }}
```

### A more useful macro: clean text

```sql
{% macro clean_string(column_name) %}
    nullif(trim(lower({{ column_name }})), '')
{% endmacro %}
```

### Macros that generate or run statements

```sql
{% macro grant_select(schema, role) %}
    {% set sql %}
        grant usage on schema {{ schema }} to {{ role }};
        grant select on all tables in schema {{ schema }} to {{ role }};
    {% endset %}
    {% do run_query(sql) %}
{% endmacro %}
```

Run a macro from the command line:

```bash
dbt run-operation grant_select --args '{schema: analytics, role: reporter}'
```

> Syntax for grants differs by warehouse. Many adapters also support a built-in `grants:` config, which is often better than hand-written grant macros.

### Overriding behaviour

dbt lets you override built-in macros. The most common is `generate_schema_name` (file 08).

---

## 4. Variables

Define defaults in `dbt_project.yml`:

```yaml
vars:
  start_date: '2025-01-01'
  is_test_run: false
```

Use them in SQL:

```sql
where order_date >= '{{ var("start_date") }}'
```

Override at run time:

```bash
dbt run --vars '{start_date: "2025-03-01"}'
```

Give `var()` a default for safety: `{{ var('start_date', '2025-01-01') }}`.

### Environment variables for secrets

Never commit passwords. In `profiles.yml`:

```yaml
password: "{{ env_var('DBT_PASSWORD') }}"
```

---

## 5. Hooks

Hooks run SQL **before or after** models or runs.

In `dbt_project.yml`:

```yaml
on-run-start:
  - "create schema if not exists audit"
on-run-end:
  - "insert into audit.run_log values ('{{ invocation_id }}', current_timestamp)"

models:
  shop_analytics:
    marts:
      +post-hook:
        - "analyze {{ this }}"
```

Use hooks sparingly. They hide behaviour from the DAG. Prefer first-class config such as `grants:` where it exists.

---

## 6. Packages

Packages are shared dbt projects. Add them in `packages.yml` at the project root:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.1.0", "<2.0.0"]
  - package: calogica/dbt_expectations
    version: [">=0.10.0", "<1.0.0"]
```

Install:

```bash
dbt deps
```

Packages land in `dbt_packages/` (git-ignore it). Browse options at https://hub.getdbt.com. Always pin version ranges and check the package page for the current recommended version.

### `dbt_utils` favourites

```sql
-- Surrogate key from several columns
{{ dbt_utils.generate_surrogate_key(['order_id', 'line_number']) }} as order_line_key

-- Date spine
{{ dbt_utils.date_spine(
    datepart="day",
    start_date="cast('2025-01-01' as date)",
    end_date="cast('2026-01-01' as date)"
) }}

-- Pivot
{{ dbt_utils.pivot('method', dbt_utils.get_column_values(ref('stg_payments'), 'method')) }}
```

Extra tests from `dbt_utils`:

```yaml
data_tests:
  - dbt_utils.unique_combination_of_columns:
      combination_of_columns: [order_id, line_number]
  - dbt_utils.expression_is_true:
      expression: "amount >= 0"
```

(As noted in file 03, very recent versions may prefer an `arguments:` wrapper for test inputs.)

---

## 7. Cross-warehouse macros

dbt provides `adapter.dispatch` and built-in cross-database macros so one macro can work across warehouses. Many `dbt_utils` macros use this. Examples of built-ins:

```sql
{{ dbt.date_trunc('month', 'order_date') }}
{{ dbt.datediff('first_order_date', 'most_recent_order_date', 'day') }}
{{ dbt.concat(['first_name', "' '", 'last_name']) }}
```

Use these instead of warehouse-specific functions when you want portability.

---

## 8. Debugging Jinja

- `dbt compile --select my_model` then open `target/compiled/...`
- `{{ log('value is ' ~ my_var, info=True) }}` prints to the console
- `dbt show --inline "select {{ cents_to_dollars('1999') }}"` evaluates SQL quickly
- If a loop or if is confusing, simplify until the compiled SQL looks right, then add back

---

## 9. When *not* to use Jinja

Jinja can turn a readable query into puzzle code. Use it to remove genuine repetition. Do not use it to be clever. A little repeated SQL that people can read beats a macro nobody understands.

---

## 10. Exercises

1. Write a `cents_to_dollars` macro and use it in a staging model.
2. Use a Jinja `for` loop to pivot payment methods into columns on `fct_orders`.
3. Add `dbt_utils` through `packages.yml`, run `dbt deps` and use `generate_surrogate_key`.
4. Add a `var('start_date')` filter and override it from the command line.
5. Add an `on-run-end` hook that writes a row to an audit table. Then remove it and discuss why you might avoid it.
6. Use `dbt show --inline` to test a macro without building a model.

**Next:** `05_intermediate_incremental_and_snapshots.md`
