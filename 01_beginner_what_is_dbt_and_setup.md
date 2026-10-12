# 01 · Beginner: What dbt Is and Getting Set Up

## 1. What is dbt?

**dbt** is a tool that lets you transform data **inside your warehouse** using plain SQL `SELECT` statements. You write the query. dbt wraps it in the right `CREATE TABLE` or `CREATE VIEW` statement, runs everything in dependency order, tests the results and documents the project.

dbt is the **T** in **ELT**:

```
Sources  ->  Extract + Load  ->  Warehouse (raw)  ->  dbt (Transform)  ->  Analytics tables  ->  BI / ML
            (Fivetran, Airbyte,                                          (dim_, fct_ models)
             custom scripts)
```

**What dbt does**
- Turns `SELECT` statements into tables and views
- Manages dependencies between models (builds them in the right order)
- Tests data quality
- Generates documentation and a lineage graph
- Brings software engineering practice (git, code review, CI) to SQL

**What dbt does not do**
- Extract or load data (you need another tool for that)
- Store data (your warehouse does that)
- Schedule itself (use an orchestrator, dbt Cloud, Airflow, GitHub Actions and so on)

---

## 2. Key concepts at a glance

| Concept | Meaning |
|---------|---------|
| **Project** | A folder with `dbt_project.yml` and your code |
| **Model** | A `.sql` file containing one `SELECT`. Each becomes a table or view. |
| **Source** | A raw table loaded by another tool, declared in YAML |
| **Seed** | A small CSV file in your repo that dbt loads as a table |
| **Test** | An assertion about your data (unique, not null and so on) |
| **Materialization** | How a model is built: view, table, incremental, ephemeral |
| **Macro** | Reusable Jinja code, like a SQL function |
| **Adapter** | The plugin for your warehouse (`dbt-duckdb`, `dbt-snowflake`, ...) |
| **Profile** | Connection settings, stored in `profiles.yml` |
| **Target** | A named environment inside a profile (dev, prod) |

**dbt Core vs dbt Cloud (the dbt platform)**
- **dbt Core:** open source command line tool. Free. You run it yourself.
- **dbt Cloud:** hosted platform with an IDE, scheduler, CI and more.
Everything in this guide works with dbt Core.

---

## 3. Install (DuckDB, zero cloud setup)

```bash
# 1. Create an isolated environment
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

# 2. Install dbt Core plus the DuckDB adapter
pip install dbt-core dbt-duckdb

# 3. Check
dbt --version
```

### Create a project

```bash
dbt init shop_analytics
cd shop_analytics
```

`dbt init` asks which adapter to use (choose `duckdb`) and creates a sample project.

### Configure the connection

Create or edit `profiles.yml`. dbt looks in the current project folder first and then in `~/.dbt/`.

```yaml
shop_analytics:          # must match 'profile:' in dbt_project.yml
  target: dev
  outputs:
    dev:
      type: duckdb
      path: dev.duckdb   # the database file DuckDB will create
      threads: 4
```

### Test the connection

```bash
dbt debug
```

You want to see `All checks passed!`.

---

## 4. Project layout

```
shop_analytics/
├── dbt_project.yml        # project-level settings
├── profiles.yml           # connection (may live in ~/.dbt instead)
├── models/                # your SQL models
│   ├── staging/
│   ├── intermediate/
│   └── marts/
├── seeds/                 # CSV files
├── macros/                # reusable Jinja
├── snapshots/             # SCD2 history
├── tests/                 # singular tests
├── analyses/              # SQL you want compiled but not built
└── target/                # compiled SQL and artifacts (generated, git-ignore)
```

Add a `.gitignore`:

```
target/
dbt_packages/
logs/
*.duckdb
```

---

## 5. Your first data and model

### Seed data

Create `seeds/raw_customers.csv`:

```csv
customer_id,first_name,last_name,email,signup_date
1,Ava,Singh,ava@example.com,2025-01-05
2,Noah,Dubois,noah@example.com,2025-01-12
3,Mia,Ramdin,mia@example.com,2025-02-03
4,Liam,Chen,liam@example.com,2025-02-20
```

Create `seeds/raw_orders.csv`:

```csv
order_id,customer_id,order_date,status,amount
101,1,2025-02-01,completed,45.00
102,1,2025-02-14,completed,30.50
103,2,2025-02-15,returned,60.00
104,3,2025-03-02,completed,25.00
105,3,2025-03-10,pending,80.00
106,5,2025-03-11,completed,15.00
```

(Order 106 points to customer 5, who does not exist. We will catch that with a test in file 03.)

Load them:

```bash
dbt seed
```

### First model

Delete the sample models in `models/example/`, then create `models/staging/stg_customers.sql`:

```sql
select
    customer_id,
    first_name,
    last_name,
    first_name || ' ' || last_name as full_name,
    lower(email)                   as email,
    cast(signup_date as date)      as signup_date
from {{ ref('raw_customers') }}
```

And `models/staging/stg_orders.sql`:

```sql
select
    order_id,
    customer_id,
    cast(order_date as date) as order_date,
    status,
    cast(amount as decimal(10, 2)) as amount
from {{ ref('raw_orders') }}
```

`{{ ref('raw_customers') }}` is the heart of dbt. It resolves to the real table name and tells dbt that `stg_customers` depends on the seed.

### Run it

```bash
dbt run
```

Inspect the result without leaving the terminal:

```bash
dbt show --select stg_orders
```

---

## 6. What happens when you run `dbt run`

1. dbt **parses** every file and builds a dependency graph (a DAG).
2. It **compiles** each model, replacing Jinja like `ref()` with real names. Look in `target/compiled/`.
3. It **wraps** each query in `CREATE VIEW AS ...` or `CREATE TABLE AS ...` (see `target/run/`).
4. It **executes** them in dependency order, in parallel up to `threads`.

---

## 7. Exercises

1. Add a model `stg_payments` from a new seed `raw_payments.csv` (`payment_id, order_id, method, amount`).
2. Run only one model: `dbt run --select stg_customers`.
3. Open `target/compiled/` and `target/run/` and compare the two files for the same model.
4. Rename a column in `stg_orders` and run again. What happens in the downstream models (none yet)? Why does `ref()` matter once you have downstream models?
5. Run `dbt run --select stg_orders+` and explain what the `+` does (hint: file 09).

**Next:** `02_beginner_models_ref_sources_seeds.md`
