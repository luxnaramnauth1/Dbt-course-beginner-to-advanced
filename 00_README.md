# dbt Learning Path: Beginner to Advanced

A self-paced course on **dbt (data build tool)**, split into separate files so you can work through them one at a time.

This guide targets dbt Core 1.9+ concepts. dbt evolves quickly (new releases, the newer Fusion engine, renamed config keys), so when something behaves differently on your version, check the official docs at https://docs.getdbt.com and the release notes.

---

## Who this is for

- Analysts and aspiring analytics engineers who know basic SQL
- Data engineers moving transformation logic out of ad hoc scripts
- Anyone preparing for analytics engineering or data engineering roles

**Prerequisites:** comfortable with `SELECT`, `JOIN`, `GROUP BY`, CTEs; basic command line use; a recent Python 3 installation.

---

## The path

| # | File | Level | What you will learn |
|---|------|-------|---------------------|
| 01 | `01_beginner_what_is_dbt_and_setup.md` | Beginner | What dbt is, ELT, installing with DuckDB, first model |
| 02 | `02_beginner_models_ref_sources_seeds.md` | Beginner | Models, `ref()`, `source()`, seeds, materializations, layering |
| 03 | `03_beginner_tests_and_docs.md` | Beginner | Generic and singular tests, `dbt build`, documentation site |
| 04 | `04_intermediate_jinja_macros_packages.md` | Intermediate | Jinja, macros, variables, hooks, packages |
| 05 | `05_intermediate_incremental_and_snapshots.md` | Intermediate | Incremental models, microbatch, snapshots (SCD2) |
| 06 | `06_intermediate_freshness_and_unit_tests.md` | Intermediate | Source freshness, unit tests, test strategy, store failures |
| 07 | `07_advanced_contracts_versions_mesh.md` | Advanced | Contracts, constraints, model versions, access, exposures, semantic layer |
| 08 | `08_advanced_ci_cd_performance_ops.md` | Advanced | Slim CI, state and defer, artifacts, orchestration, performance, Python models |
| 09 | `09_cheatsheet_and_capstone.md` | All | Command cheatsheet, selector syntax, capstone project, interview questions |

---

## The running example

Every file uses the same tiny fictional shop with **customers**, **orders** and **payments**. You will build:

```
seeds (raw data)
   -> staging models   (clean and rename)
   -> intermediate     (join and reshape)
   -> marts            (dim_customers, fct_orders)
```

Everything runs locally on **DuckDB**, so you need no cloud warehouse account. The SQL ports to Snowflake, BigQuery, Postgres, Databricks and others with small changes.

---

## Suggested schedule

| Week | Focus |
|------|-------|
| 1 | Files 01 to 03. Build the staging and marts layers and add tests. |
| 2 | Files 04 to 06. Add macros, an incremental model and a snapshot. |
| 3 | Files 07 and 08. Add contracts, CI and state-based builds. |
| 4 | File 09. Finish the capstone and review the interview questions. |

---

## Tips for learning well

1. **Type the code yourself.** Copy and paste teaches far less.
2. **Break things on purpose.** Remove a column, fail a test, read the error.
3. **Read compiled SQL.** Look in `target/compiled/` often. It demystifies Jinja.
4. **Commit as you go.** Use git so you can compare versions of your project.
5. **Put it on your portfolio.** A public repo with the capstone is a strong signal to recruiters.
