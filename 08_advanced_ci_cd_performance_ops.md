# 08 · Advanced: CI/CD, State, Performance, Python Models and Operations

## 1. Environments and targets

You want at least **dev** and **prod**, and ideally a **CI** environment.

```yaml
shop_analytics:
  target: dev
  outputs:
    dev:
      type: snowflake              # example adapter
      schema: dbt_{{ env_var('USER') }}
      threads: 4
      # account, user, auth ... via env vars
    ci:
      type: snowflake
      schema: "dbt_ci_{{ env_var('PR_NUMBER', 'local') }}"
      threads: 8
    prod:
      type: snowflake
      schema: analytics
      threads: 8
```

Run against a target:

```bash
dbt build --target prod
```

Rules:
- Developers **never** write to prod schemas from their laptops
- Secrets come from environment variables or a secrets manager, never from git
- Each developer and each CI run gets an isolated schema

### Controlling schema names

By default dbt builds `<target_schema>_<custom_schema>`. Many teams want prod to use clean schema names (`marts`, not `analytics_marts`). Override the macro:

```sql
-- macros/generate_schema_name.sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if target.name == 'prod' and custom_schema_name is not none -%}
        {{ custom_schema_name | trim }}
    {%- elif custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ default_schema }}_{{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

---

## 2. Artifacts

Every run writes files into `target/`:

| File | Contents |
|------|----------|
| `manifest.json` | The full project graph: nodes, config, dependencies, docs |
| `run_results.json` | Status, timing and rows affected for every node in the last run |
| `catalog.json` | Column metadata from `dbt docs generate` |
| `sources.json` | Freshness results |

Artifacts power state comparison, docs, and observability. Persist `manifest.json` from your production runs. You will need it for slim CI.

---

## 3. State-based selection and Slim CI

Rebuilding the whole project on every pull request is slow and costly. **Slim CI** builds only what changed, and **defers** to production for everything else.

Concepts:
- `--state <path>`: folder holding a previous `manifest.json` (usually from the last production run)
- `state:modified`: nodes whose definition changed compared to that state
- `state:modified+`: those nodes plus everything downstream
- `--defer`: for unchanged upstream models, read the **production** version instead of building a copy

```bash
# prod-artifacts/ contains manifest.json from the last production run
dbt build \
  --select state:modified+ \
  --defer \
  --state prod-artifacts \
  --target ci
```

Variants:

```bash
dbt build --select state:modified+ --exclude state:modified+,config.materialized:incremental
dbt build --select result:error+ --state prod-artifacts      # rerun last failures and downstream
dbt retry                                                     # rerun from the point of failure
```

### Preview a build without processing data

```bash
dbt build --empty
```

`--empty` limits `ref()` and `source()` inputs to zero rows. The models still build, so you validate schema and SQL quickly and cheaply.

### Clone

`dbt clone` creates zero-copy or lightweight copies of prod models into your environment on warehouses that support it, which is an alternative to deferral.

---

## 4. A GitHub Actions example

`.github/workflows/dbt_ci.yml`:

```yaml
name: dbt CI

on:
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      DBT_PROFILES_DIR: ./ci
      WAREHOUSE_USER: ${{ secrets.WAREHOUSE_USER }}
      WAREHOUSE_PASSWORD: ${{ secrets.WAREHOUSE_PASSWORD }}
      PR_NUMBER: ${{ github.event.number }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - run: pip install -r requirements.txt

      - run: dbt deps

      # Get the latest production manifest (store it in cloud storage after prod runs)
      - name: Download prod artifacts
        run: ./scripts/download_prod_manifest.sh prod-artifacts

      - name: Lint SQL
        run: sqlfluff lint models --dialect snowflake

      - name: Build changed models
        run: |
          dbt build \
            --select state:modified+ \
            --defer --state prod-artifacts \
            --target ci

      - name: Cleanup CI schema
        if: always()
        run: dbt run-operation drop_ci_schema --target ci
```

Adapt the script names, adapter and secrets to your setup. Typical **CD** job: on merge to `main`, run `dbt build --target prod`, then upload the new `manifest.json` as the production artifact.

---

## 5. Orchestration

dbt does not schedule itself. Options:

| Option | Notes |
|--------|-------|
| dbt platform (cloud) jobs | Built-in scheduler, CI, docs hosting |
| Airflow, Dagster, Prefect | Run `dbt build` as tasks, or model each dbt node as an asset or task |
| GitHub Actions, GitLab CI, cron | Fine for smaller teams |
| Warehouse-native schedulers | Possible on some platforms |

Production job pattern:

1. `dbt deps`
2. `dbt source freshness`
3. `dbt build` (seeds, snapshots, models, tests)
4. Upload artifacts, send alerts on failure

Split heavy or critical models into **separate jobs** using tags or selectors (hourly versus daily), so one slow model does not hold up everything.

---

## 6. Performance tuning

**Start by measuring.** Look at `run_results.json` and your warehouse's query history to find slow models.

Practical levers:

| Lever | Detail |
|-------|--------|
| `threads` | More parallel models, limited by the DAG shape and warehouse concurrency |
| Materialization | Heavy joins to `table`, huge append-only data to `incremental` |
| Incremental or microbatch | Process only new data |
| Avoid `select *` | Pull only the columns you need |
| Filter early | Push `where` clauses upstream, before joins |
| Partitioning and clustering | Configure per adapter, for example `partition_by` and `cluster_by` on BigQuery |
| Reduce fan-out | Fewer, wider intermediate models can beat many tiny ones, or the reverse. Measure. |
| Selective runs | State selection in CI, tags and selectors in prod |
| Warehouse sizing | Sometimes the right fix is a bigger warehouse for one heavy model, or scheduling it off-peak |

Cost tip: use `--empty` and small dev data (`{% if target.name == 'dev' %} limit ... {% endif %}` or a date filter on dev builds) to keep developer iteration cheap.

---

## 7. Custom generic tests

If you copy the same singular test twice, turn it into a **generic test** in `tests/generic/` or `macros/`:

```sql
-- tests/generic/is_positive.sql
{% test is_positive(model, column_name) %}

select {{ column_name }}
from {{ model }}
where {{ column_name }} <= 0

{% endtest %}
```

Use it:

```yaml
columns:
  - name: amount
    data_tests:
      - is_positive
```

---

## 8. Custom materializations (rarely needed)

You can write your own materialization in Jinja when built-in ones do not fit (for example a special load pattern). Most teams never need this. Read `materialization` in the docs, and prefer configuring an existing incremental strategy first.

---

## 9. Python models

On adapters that support them (such as Snowflake, Databricks and BigQuery), a model can be a Python file that returns a dataframe:

```python
# models/marts/customer_segments.py
def model(dbt, session):
    dbt.config(materialized="table")

    orders = dbt.ref("fct_orders")           # dataframe in the platform's engine
    # ... pandas / PySpark / Snowpark operations here ...
    return orders
```

Use Python models for things SQL does badly (ML scoring, complex statistical work, calling a library). Keep core business logic in SQL, and note that the dataframe type depends on the platform (Snowpark, PySpark or pandas-style). Check your adapter's docs.

---

## 10. Debugging toolbox

| Situation | Try |
|-----------|-----|
| Which SQL actually ran? | `target/compiled/` and `target/run/`, and `logs/dbt.log` |
| Jinja confusion | `dbt compile`, `dbt show --inline "..."` |
| Slow model | `run_results.json`, warehouse query profile |
| Test failure | `store_failures: true`, or `dbt test --select <name>` and run the printed compiled test SQL |
| Wrong dependency | `dbt ls --select +model --output json`, lineage graph |
| Parse errors | `dbt parse` for fast feedback |
| "Where did my column go?" | `dbt docs generate` then check the catalog and lineage |

Useful debug flags: `dbt --debug run` and `dbt run --log-level debug`.

---

## 11. Observability and reliability

- **Alert** on failed jobs, failed critical tests and stale sources
- **Track** run duration trends. Gradual slowness is an early warning.
- **Record** test results over time (`store_failures`, or observability packages and tools)
- **Define SLAs** for key tables (for example "ready by 7am") and monitor against them
- **Runbooks**: for each critical failure, write down who investigates, how, and how to backfill
- **Backfills**: know your incremental model reprocessing options (`--full-refresh`, microbatch ranges)

---

## 12. Code quality

- **SQL linting:** SQLFluff with a dbt templater, run in CI
- **Project evaluators:** packages and tools that check naming, documentation coverage, DAG structure and more (for example `dbt_project_evaluator`)
- **Pre-commit hooks:** run lint and parse before every commit
- **Code review:** require review on pull requests, with CI green before merge
- **Documentation coverage:** make descriptions on public models mandatory

---

## 13. The engine landscape

dbt has been evolving beyond the original Python implementation, including a newer engine (Fusion) with faster parsing and richer static analysis. Features, supported adapters and licensing change over time. Treat this guide as concept-focused, and check the official dbt docs and release notes for what your version and platform support.

---

## 14. Exercises

1. Create `dev`, `ci` and `prod` targets in `profiles.yml` using environment variables.
2. Run a full `dbt build`, save `target/manifest.json` into `prod-artifacts/`, edit one model and run `dbt build --select state:modified+ --defer --state prod-artifacts`. Which models ran?
3. Override `generate_schema_name` so prod uses clean schema names, and verify with `dbt compile --target prod`.
4. Write the GitHub Actions workflow for your repo (use DuckDB with a file database for a free demo).
5. Turn one singular test into a custom generic test.
6. Read `run_results.json` and list your three slowest models. Propose one optimisation for each.
7. Add SQLFluff to the project and fix the lint errors.

**Next:** `09_cheatsheet_and_capstone.md`
