# 09 · Cheatsheet, Capstone Project and Interview Questions

## Part A: Command cheatsheet

### Everyday commands

| Command | What it does |
|---------|--------------|
| `dbt init <name>` | Create a new project |
| `dbt debug` | Check connection and configuration |
| `dbt deps` | Install packages from `packages.yml` |
| `dbt seed` | Load CSV seeds |
| `dbt run` | Build models |
| `dbt test` | Run data and unit tests |
| `dbt build` | Seeds, snapshots, models and tests in DAG order |
| `dbt snapshot` | Run snapshots |
| `dbt compile` | Compile SQL without running it |
| `dbt parse` | Parse the project (fast validation) |
| `dbt show` | Preview a model's or inline query's results |
| `dbt ls` | List resources that match a selector |
| `dbt docs generate` / `dbt docs serve` | Build and view documentation |
| `dbt source freshness` | Check source freshness |
| `dbt run-operation <macro>` | Run a macro |
| `dbt clean` | Remove folders listed in `clean-targets` |
| `dbt retry` | Rerun from the last failure point |
| `dbt clone` | Clone nodes from another state |

### Useful flags

| Flag | Meaning |
|------|---------|
| `--select` / `-s` | Choose resources |
| `--exclude` | Remove resources from the selection |
| `--target` / `-t` | Pick the profile target |
| `--full-refresh` | Rebuild incrementals and seeds from scratch |
| `--vars '{k: v}'` | Pass variables |
| `--threads N` | Override thread count |
| `--fail-fast` / `-x` | Stop at the first failure |
| `--empty` | Build with zero-row inputs |
| `--state <path>` | Folder with a previous manifest |
| `--defer` | Use the state's relations for unbuilt upstream nodes |
| `--debug` | Verbose logging |

---

## Part B: Selector syntax

| Selector | Meaning |
|----------|---------|
| `my_model` | One model |
| `+my_model` | The model and all **upstream** parents |
| `my_model+` | The model and all **downstream** children |
| `+my_model+` | Both directions |
| `2+my_model` | Up to 2 levels upstream |
| `@my_model` | The model, its children, and the parents of those children |
| `staging` or `path:models/staging` | A folder |
| `tag:daily` | By tag |
| `config.materialized:incremental` | By config |
| `source:raw.orders+` | A source and everything downstream |
| `resource_type:model` | By resource type |
| `test_type:unit` / `test_type:generic` / `test_type:singular` | By test type |
| `state:modified+` | Changed since the compared state, plus descendants |
| `result:error+` | Failed in the compared run, plus descendants |
| `exposure:my_dashboard` | An exposure |

**Set operations**

```bash
dbt build --select "tag:daily,config.materialized:table"   # comma = intersection (AND)
dbt build --select "tag:daily tag:hourly"                  # space = union (OR)
dbt build --select staging --exclude stg_legacy
```

**Named selectors** in `selectors.yml`:

```yaml
selectors:
  - name: nightly
    description: Everything tagged nightly, except experimental models
    definition:
      union:
        - method: tag
          value: nightly
        - exclude:
            - method: tag
              value: experimental
```

```bash
dbt build --selector nightly
```

---

## Part C: Config quick reference

```yaml
# dbt_project.yml (excerpt)
name: shop_analytics
version: '1.0.0'
profile: shop_analytics

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets: ["target", "dbt_packages"]

vars:
  start_date: '2025-01-01'

models:
  shop_analytics:
    staging:
      +materialized: view
      +tags: ['staging']
    intermediate:
      +materialized: ephemeral
    marts:
      +materialized: table
      +tags: ['marts']

seeds:
  shop_analytics:
    +schema: raw

data_tests:
  +store_failures: false
```

---

## Part D: Capstone project

Build a portfolio-ready analytics project end to end.

### Brief

You are the analytics engineer for an online shop. Leadership wants a trustworthy view of **customers, orders, payments and revenue**, refreshed daily, with history of customer plan changes and clear documentation.

### Requirements

**Data and layering**
- [ ] Seeds (or a loader script) for at least 4 raw tables: customers, orders, order_items, payments
- [ ] Staging models for each, with consistent naming and casting
- [ ] At least one intermediate model (for example an orders and payments rollup)
- [ ] Marts: `dim_customers`, `fct_orders`, and a daily `fct_revenue`

**Quality**
- [ ] Unique and not-null tests on every primary key
- [ ] `relationships` tests on every foreign key
- [ ] `accepted_values` on status columns
- [ ] At least 2 singular tests for business rules
- [ ] At least 1 custom generic test
- [ ] At least 2 unit tests for tricky logic
- [ ] Source freshness configured

**Intermediate features**
- [ ] A reusable macro (and use of one package such as `dbt_utils`)
- [ ] One incremental model with a lookback window
- [ ] One snapshot (for example customer plan history) and a point-in-time join

**Advanced features**
- [ ] Enforced contract on `dim_customers`, `fct_orders` and `fct_revenue`
- [ ] Groups and access levels, with `private` intermediate models
- [ ] One model with a v1 and v2 and a deprecation date
- [ ] At least one exposure
- [ ] `dev`, `ci` and `prod` targets
- [ ] GitHub Actions CI using `state:modified+` and `--defer` (can be simulated locally with saved artifacts)

**Documentation**
- [ ] Descriptions on every public model and column
- [ ] A doc block for at least one complex field
- [ ] Generated docs site (screenshot of the lineage graph in your README)
- [ ] A README: purpose, architecture diagram, how to run, decisions and trade-offs

### Stretch goals

- Define a `revenue` metric with MetricFlow
- Add SQLFluff and pre-commit hooks
- Add `dbt_project_evaluator` and fix what it finds
- Deploy on a free-tier cloud warehouse or run on DuckDB with a hosted docs site
- Write a short blog post explaining your design decisions

### How to present it (for recruiters)

1. A public GitHub repo with a clean commit history
2. A README with an architecture diagram and a lineage screenshot
3. A short "decisions" section: why incremental here, why snapshot there, what you would change at scale
4. A green CI badge
5. A 2-minute walkthrough on LinkedIn linking to the repo

---

## Part E: Interview questions

### Beginner

1. What is dbt and where does it sit in an ELT pipeline?
2. What is the difference between `ref()` and `source()`?
3. Name the four built-in generic tests.
4. What are the main materializations, and when would you use each?
5. What does `dbt build` do that `dbt run` does not?
6. What is a seed, and when would you not use one?

### Intermediate

7. Explain how an incremental model works. What is `is_incremental()` and `{{ this }}`?
8. How do you handle late-arriving data in an incremental model?
9. What is a snapshot, and what do `dbt_valid_from` and `dbt_valid_to` mean?
10. When would you use a macro rather than a CTE, and when is Jinja overkill?
11. What is the difference between a data test and a unit test?
12. How do you manage secrets and environments in dbt?
13. What does `--full-refresh` do?

### Advanced

14. What is a model contract, and how is it different from a test?
15. How would you make a breaking change to a model used by five other teams?
16. Explain slim CI. What do `state:modified+` and `--defer` do?
17. A model takes 40 minutes. How do you investigate and speed it up?
18. How do you decide between one large dbt project and several?
19. How would you organise testing across sources, staging and marts?
20. What are exposures and why use them?
21. How would you detect and prevent join fan-out?
22. How do you monitor a production dbt deployment?

### Scenario prompts

- "Revenue differs between two dashboards. How do dbt features help prevent that?" (semantic layer, single mart, contracts, tests)
- "A source system started sending duplicate rows. Where in the project do you catch it, and how do you alert?" (source and staging tests, freshness, alerts)
- "You need yesterday's customer plan on every historical order." (snapshot and point-in-time join)

---

## Part F: Learning resources

- Official docs: https://docs.getdbt.com
- Package hub: https://hub.getdbt.com
- Community Slack and forum (linked from the docs)
- Official free courses (dbt Fundamentals and later courses) on the dbt Learn site

**Finished the path? Ship the capstone, write about it, and share it.**
