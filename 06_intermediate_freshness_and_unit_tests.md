# 06 · Intermediate: Source Freshness, Unit Tests and Test Strategy

## 1. Source freshness

Data tests check **content**. Freshness checks whether data is **recent**. If your loader silently stops, models keep running on stale data and nobody notices.

Declare freshness on a source:

```yaml
version: 2

sources:
  - name: raw
    schema: main
    loaded_at_field: _loaded_at        # a timestamp column set by your loader
    freshness:
      warn_after:  {count: 12, period: hour}
      error_after: {count: 24, period: hour}
    tables:
      - name: orders
      - name: customers
        freshness:                      # override per table
          warn_after: {count: 3, period: day}
```

Check it:

```bash
dbt source freshness
```

Run this **before** `dbt build` in production, so you do not transform stale data without knowing it. Freshness writes results to `target/sources.json`, which monitoring tools can read.

You can also select only fresh-enough models in newer workflows with source-status selection. Check current docs for `source_status:` selectors.

---

## 2. Unit tests (dbt 1.8+)

**Data tests** run on real data after the model is built. **Unit tests** check your **logic** on small, fixed, mock inputs **before** relying on real data.

Use them for tricky transformations: regex, case logic, date maths, edge cases.

```yaml
unit_tests:
  - name: test_order_size_buckets
    description: Orders are bucketed into small, medium and large.
    model: fct_orders_bucketed
    given:
      - input: ref('stg_orders')
        rows:
          - {order_id: 1, amount: 5}
          - {order_id: 2, amount: 50}
          - {order_id: 3, amount: 500}
    expect:
      rows:
        - {order_id: 1, size_bucket: small}
        - {order_id: 2, size_bucket: medium}
        - {order_id: 3, size_bucket: large}
```

Run them:

```bash
dbt test --select "test_type:unit"
dbt build                      # runs unit tests before building the model
```

Tips:
- You only need to list the columns your logic uses
- Mock every `ref()` and `source()` the model reads
- For incremental models you can test both modes using `overrides: macros: is_incremental: true`
- Unit tests are best run in **development and CI**, not production, since they do not depend on production data
- Warehouse features that vary by adapter may limit some cases. Check the docs.

---

## 3. Storing failures

By default a failing test just reports a count. Store the failing rows to investigate:

```yaml
data_tests:
  - not_null:
      config:
        store_failures: true
```

Or for the whole project in `dbt_project.yml`:

```yaml
data_tests:
  +store_failures: true
```

Failed rows land in a schema like `<your_schema>_dbt_test__audit`. Query them to find exactly which records failed.

---

## 4. Tuning tests

| Config | Purpose |
|--------|---------|
| `severity: warn` | Report but do not fail the build |
| `error_if` / `warn_if` | Thresholds for failure counts |
| `where:` | Limit the test to a subset of rows (for example only recent data) |
| `limit:` | Cap the number of failing rows reported |
| `tags:` | Group tests so you can run them separately |

```yaml
data_tests:
  - unique:
      config:
        where: "order_date >= current_date - interval 7 day"
        tags: ['critical']
```

Run by tag:

```bash
dbt test --select tag:critical
```

---

## 5. More data tests from packages

`dbt_utils` and `dbt_expectations` (file 04) give you tests such as:

```yaml
data_tests:
  - dbt_utils.accepted_range:
      min_value: 0
      max_value: 100000
  - dbt_expectations.expect_column_values_to_match_regex:
      regex: "^[^@]+@[^@]+\\.[^@]+$"
  - dbt_utils.equal_rowcount:
      compare_model: ref('stg_orders')
```

Use them for ranges, formats, row-count parity and cross-model reconciliation.

---

## 6. A sensible test strategy

Think in layers:

| Layer | What to test | Why |
|-------|--------------|-----|
| **Sources** | Not null on keys, freshness | Catch bad loads at the door |
| **Staging** | Primary keys unique and not null, accepted values | Clean contracts for downstream |
| **Intermediate** | Row-count sanity, relationships | Joins did not fan out or drop rows |
| **Marts** | Keys, business rules, reconciliation | Numbers stakeholders trust |
| **Logic** | Unit tests for complex expressions | Catch regressions before data arrives |

**Principles**
- Test **where problems enter** (sources, staging) so they do not spread
- Test **assumptions you rely on** (uniqueness of a join key)
- Prefer **few, meaningful** tests over blanket coverage
- Decide **who acts** on a failure. A warning nobody reads is worthless.
- Use `severity: warn` for known, tolerated issues and track them to zero

---

## 7. Detecting join fan-out

A classic bug: joining a one-to-many table silently multiplies rows. Guard against it with a row-count test:

```sql
-- tests/assert_fct_orders_rowcount_matches_staging.sql
select 1
from (
    select count(*) as n from {{ ref('fct_orders') }}
) f
cross join (
    select count(*) as n from {{ ref('stg_orders') }}
) s
where f.n <> s.n
```

---

## 8. Exercises

1. Add `loaded_at_field` and freshness to your sources. Run `dbt source freshness` and read `target/sources.json`.
2. Write a unit test for a model with a `case when` expression, including an edge case (null amount).
3. Turn on `store_failures` for one test. Make it fail on purpose and query the audit table.
4. Add a `where:` to a test so it only checks the last 7 days, and explain when that is a good idea.
5. Add `dbt_utils.accepted_range` to `amount`.
6. Write the row-count fan-out test above for one of your marts.

**Next:** `07_advanced_contracts_versions_mesh.md`
