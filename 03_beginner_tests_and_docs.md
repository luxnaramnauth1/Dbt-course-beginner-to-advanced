# 03 · Beginner: Tests and Documentation

## 1. Why test?

Transformations fail quietly: duplicate IDs, missing values, orphaned records. dbt tests turn assumptions into checks that run on every build.

A dbt test is a **query that returns failing rows**. If it returns zero rows, the test passes.

---

## 2. Generic tests (built in)

Four ship with dbt:

| Test | Checks |
|------|--------|
| `unique` | No duplicates in the column |
| `not_null` | No nulls in the column |
| `accepted_values` | Values come from an allowed list |
| `relationships` | Every value exists in another model's column (foreign key check) |

Declare them in YAML next to your models, for example `models/staging/_staging.yml`:

```yaml
version: 2

models:
  - name: stg_customers
    description: One row per customer.
    columns:
      - name: customer_id
        description: Primary key.
        data_tests:
          - unique
          - not_null
      - name: email
        data_tests:
          - not_null

  - name: stg_orders
    description: One row per order.
    columns:
      - name: order_id
        data_tests:
          - unique
          - not_null
      - name: status
        data_tests:
          - accepted_values:
              values: ['completed', 'pending', 'returned']
      - name: customer_id
        data_tests:
          - relationships:
              to: ref('stg_customers')
              field: customer_id
```

> **Version note:** since dbt 1.8 the key is `data_tests:`. The older `tests:` key still works. Very recent versions may also prefer nesting test arguments under an `arguments:` key. Check the docs for your version if you see a deprecation warning.

Run them:

```bash
dbt test
dbt test --select stg_orders
```

With our seed data, the `relationships` test **fails**: order 106 belongs to customer 5, who does not exist. That is exactly what tests are for. Decide whether to fix the source, filter the row, or downgrade the test to a warning.

---

## 3. Test severity and thresholds

```yaml
- name: customer_id
  data_tests:
    - relationships:
        to: ref('stg_customers')
        field: customer_id
        config:
          severity: warn          # default is error
```

Warn only after a threshold:

```yaml
        config:
          severity: error
          error_if: ">10"
          warn_if: ">0"
```

---

## 4. Singular tests

A singular test is a `.sql` file in `tests/` that returns failing rows.

`tests/assert_no_negative_amounts.sql`:

```sql
select
    order_id,
    amount
from {{ ref('stg_orders') }}
where amount < 0
```

If the query returns any rows, the test fails. Use singular tests for one-off business rules. If you reuse the pattern, turn it into a custom generic test (file 08).

---

## 5. `dbt build`: run and test together

`dbt run` builds models. `dbt test` tests them. **`dbt build`** does both, in dependency order, and also loads seeds and runs snapshots.

```bash
dbt build
```

If an upstream test fails, dbt **skips downstream models**, so bad data does not flow forward. This is why `dbt build` is the standard command for production.

---

## 6. Documentation

### Descriptions

Add `description:` to models and columns in YAML (as above). Use Markdown if you like.

### Doc blocks for long text

`models/docs.md`:

```
{% docs order_status %}
The lifecycle state of an order.

| status    | meaning                         |
|-----------|---------------------------------|
| completed | Paid and delivered              |
| pending   | Created but not yet paid        |
| returned  | Delivered, then sent back       |
{% enddocs %}
```

Reference it:

```yaml
      - name: status
        description: "{{ doc('order_status') }}"
```

### Generate and view the site

```bash
dbt docs generate
dbt docs serve
```

You get a searchable website with:
- Every model, column, test and source
- The **lineage graph** (click any node to see upstream and downstream)
- Compiled SQL for every model

Show the project in an interview with the lineage graph open. It is a strong demo.

---

## 7. What to test: a practical starter list

- Every **primary key**: `unique` and `not_null`
- Every **foreign key**: `relationships`
- Every **status or category** column: `accepted_values`
- Every **source**: at least not-null on keys
- A few **business rules** as singular tests (revenue never negative, dates in range)

Do not over-test. Hundreds of noisy, low-value tests train people to ignore failures. Test what would hurt if it were wrong.

---

## 8. Exercises

1. Add the YAML above, run `dbt test` and confirm the `relationships` failure for order 106.
2. Fix it two ways: (a) set `severity: warn`, (b) filter the orphan in `stg_orders`. Discuss the trade-offs.
3. Add `unique` and `not_null` to every primary key in your marts.
4. Write a singular test that fails if any `completed` order has `amount = 0`.
5. Add a doc block, generate the docs and find your model's lineage graph.
6. Run `dbt build`. Break a test on purpose and see which downstream models get skipped.

**Next:** `04_intermediate_jinja_macros_packages.md`
