# 07 · Advanced: Contracts, Constraints, Versions, Access, Exposures and the Semantic Layer

As projects and teams grow, the problem shifts from "does my SQL work?" to "can other people depend on my models without breaking?" This file covers dbt's tools for **governing** a large project.

---

## 1. Model contracts

A **contract** enforces a model's shape: column names and data types. If the query does not match, the build fails **before** anything is created.

```yaml
models:
  - name: dim_customers
    config:
      contract:
        enforced: true
    columns:
      - name: customer_id
        data_type: integer
        constraints:
          - type: not_null
          - type: primary_key
      - name: full_name
        data_type: varchar
      - name: lifetime_value
        data_type: decimal(10,2)
```

What you get:
- A guaranteed schema for downstream users and BI tools
- Early failure on accidental column renames or type changes
- Warehouse-level constraints where the platform supports them

---

## 2. Constraints

Constraints are declared in YAML and (where supported) applied in the warehouse DDL:

| Constraint | Notes |
|-----------|-------|
| `not_null` | Enforced by most warehouses |
| `unique` | Often **not enforced** by warehouses, only informational |
| `primary_key` | Same, frequently informational |
| `foreign_key` | Same, frequently informational |
| `check` | Support varies |
| `custom` | Warehouse-specific expression |

Important: enforcement differs by platform. Snowflake, for example, treats some constraints as metadata only. **Constraints do not replace data tests.** Keep your `unique` and `relationships` tests.

---

## 3. Model versions

When you must make a **breaking change** to a public model, create a new **version** and give consumers time to migrate.

```yaml
models:
  - name: dim_customers
    latest_version: 2
    config:
      contract: {enforced: true}
    columns:
      - name: customer_id
        data_type: integer
      - name: country_name
        data_type: varchar
    versions:
      - v: 1                          # old shape, file dim_customers_v1.sql
      - v: 2                          # new shape, file dim_customers_v2.sql
        columns:
          - include: all
            exclude: [country_name]   # v2 drops this column
```

Consumers choose their version:

```sql
select * from {{ ref('dim_customers', v=1) }}   -- pinned to old
select * from {{ ref('dim_customers') }}        -- latest version
```

Add a **deprecation date** to v1 so users get warnings:

```yaml
      - v: 1
        deprecation_date: 2026-12-31
```

Workflow: release v2, announce, let teams migrate, remove v1 after the deprecation date.

---

## 4. Access and groups

Control which models others may `ref()`.

```yaml
# models/groups.yml
groups:
  - name: finance
    owner:
      name: Finance Analytics
      email: finance-data@example.com
```

```yaml
models:
  - name: fct_revenue
    group: finance
    access: public          # anyone, including other projects
  - name: int_revenue_adjustments
    group: finance
    access: private         # only models in the same group
  - name: stg_invoices
    access: protected       # default: same project only
```

| Access | Who can `ref()` it |
|--------|--------------------|
| `private` | Only models in the same group |
| `protected` (default) | Models in the same project |
| `public` | Any model, including other projects |

Treat `public` models as **products** with contracts, tests, descriptions and versions.

---

## 5. Multi-project architecture ("dbt Mesh")

Large organisations split one giant project into several owned by different teams. Teams reference each other's **public** models through **cross-project `ref`**:

```sql
select * from {{ ref('finance_project', 'fct_revenue') }}
```

Why split?
- Faster parse and build times
- Clear ownership and permissions
- Teams release on their own schedule

The cross-project machinery (project dependencies, metadata exchange) is a feature of the dbt platform. With dbt Core alone, you can approximate it by publishing packages or by treating another team's output tables as `source()`s. Check the current docs for what your setup supports. Do not split early. Multiple projects add coordination cost.

---

## 6. Exposures

An **exposure** documents a downstream use of your models (dashboards, apps, ML jobs) so lineage extends to the people who depend on your data.

```yaml
exposures:
  - name: weekly_revenue_dashboard
    type: dashboard
    maturity: high
    url: https://bi.example.com/dashboards/weekly-revenue
    description: Executive revenue summary reviewed every Monday.
    owner:
      name: Finance Analytics
      email: finance-data@example.com
    depends_on:
      - ref('fct_orders')
      - ref('dim_customers')
```

Benefits:
- Show up in the lineage graph
- Let you run everything a dashboard depends on: `dbt build --select +exposure:weekly_revenue_dashboard`
- Answer "what breaks if I change this?"

---

## 7. Meta, tags and organising a large project

```yaml
models:
  - name: fct_orders
    config:
      tags: ['finance', 'daily']
      meta:
        owner: '@finance-analytics'
        contains_pii: false
```

Use tags to select (`--select tag:daily`) and `meta` for custom metadata consumed by tools or documentation. Combine with `selectors.yml` (file 09) for named, reusable selections.

---

## 8. The Semantic Layer and metrics (overview)

Different dashboards often compute "revenue" differently. The **dbt Semantic Layer**, powered by **MetricFlow**, lets you define metrics **once** in YAML and query them consistently from many tools.

Building blocks:
- **Semantic models**: map a model to entities, dimensions and measures
- **Metrics**: business definitions built on measures

```yaml
semantic_models:
  - name: orders
    model: ref('fct_orders')
    defaults:
      agg_time_dimension: order_date
    entities:
      - name: order_id
        type: primary
      - name: customer_id
        type: foreign
    dimensions:
      - name: order_date
        type: time
        type_params:
          time_granularity: day
      - name: status
        type: categorical
    measures:
      - name: order_total
        agg: sum
        expr: amount

metrics:
  - name: revenue
    label: Revenue
    type: simple
    type_params:
      measure: order_total
```

Querying and serving metrics to BI tools is part of the dbt platform's Semantic Layer. MetricFlow itself is open source. The YAML specification has evolved between releases, so read the current docs before building on it.

---

## 9. Governance checklist for a mature project

- [ ] Public models have contracts, tests and descriptions
- [ ] Breaking changes go through versioning with deprecation dates
- [ ] Every model has an owner (group or meta)
- [ ] Private models are marked `private` or `protected`
- [ ] Exposures exist for key dashboards
- [ ] PII columns are tagged and handled per policy
- [ ] Naming conventions and layering are documented and enforced

---

## 10. Exercises

1. Add an enforced contract to `dim_customers`. Change a data type in the SQL and watch the build fail.
2. Create a `groups.yml`, assign marts to a group and mark an intermediate model `private`. Try to `ref()` it from outside the group.
3. Create `dim_customers` v2 that drops a column, keep v1, and `ref()` both.
4. Add an exposure for an imaginary dashboard and run `dbt build --select +exposure:<name>`.
5. Read the MetricFlow docs and define one metric (`revenue`) on `fct_orders`.
6. Write a one-page "data product" description for your best mart: owner, consumers, SLA, tests, versioning policy.

**Next:** `08_advanced_ci_cd_performance_ops.md`
