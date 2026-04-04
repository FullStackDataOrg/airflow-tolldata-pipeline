# Teardown — Airflow ETL: Two Pipelines

Two Apache Airflow DAGs in a single repository: an ETL pipeline for multi-format toll data loaded into PostgreSQL, and a server access log processing pipeline that downloads, extracts, transforms, and saves web server logs.

---

## Stack Choices & Rationale

| Component | Decision Rationale |
|---|---|
| **Apache Airflow** | The dominant open-source workflow orchestration platform. DAG-based dependency management, retry policies, task-level monitoring, and a web UI make it the standard for ETL pipelines in the modern data stack. |
| **BashOperator + PythonOperator** | The toll pipeline uses both: Bash for file extraction and preprocessing (natural for shell-based format handling), Python for data consolidation and database loading (where pandas and SQLAlchemy are appropriate). |
| **pandas + SQLAlchemy** | Standard combination for Python-based ETL. pandas handles format normalisation and column consolidation; SQLAlchemy abstracts the PostgreSQL connection and handles the bulk insert. |
| **PostgreSQL** | Structured sink for the processed toll data. The `car_details` table schema is defined by the pipeline, demonstrating schema management alongside data loading. |
| **PythonOperator only (log pipeline)** | Access log processing is purely Python — no shell commands needed for download, parse, transform, and save. Keeps the DAG consistent and testable. |

---

## Key Design Decisions

- **Each extraction task is scoped to a single file format** (CSV, TSV, fixed-width). This enables independent retries — if the TSV extraction fails, CSV extraction does not re-run.
- **`consolidate_data` is a separate task**, not merged into the extraction tasks. The consolidation logic is decoupled from format-specific parsing, making both independently testable.
- **The log pipeline uses a `download` task** that fetches from a public URL — making the pipeline self-contained with no manual file placement required to run it.
- **Task naming follows `verb_object`** (`extract_data_from_csv`, `consolidate_data`, `postgresload`) — a consistent convention that makes DAG graphs readable at a glance.

---

## Trade-offs

| Decision | Benefit | Cost |
|---|---|---|
| BashOperator for extraction | Direct use of standard Unix tools (`cut`, `awk`), no Python overhead | Harder to test and debug than Python; errors surface as non-zero exit codes without stack traces |
| pandas for consolidation | Readable, flexible, handles column alignment across formats | Not suitable for large-scale data; for files > 1 GB, Spark or DuckDB would be more appropriate |
| Single PostgreSQL load step | Simple, atomic load after all extraction succeeds | No incremental loading — full reload on every run; adds idempotency complexity for production schedules |
| Airflow (full installation) | Production-grade orchestration, full feature set | Heavyweight for two simple pipelines; Prefect or Dagster would have lower setup overhead at this scale |

---

## Extensions & Real-World Use Cases

- Add an **idempotent load pattern** to the PostgreSQL sink: use `INSERT ... ON CONFLICT DO UPDATE` (upsert) with a unique key on `vehicle_number + timestamp` to make the pipeline safe to re-run.
- Replace the BashOperator extraction with **DuckDB** — DuckDB can read CSV, TSV, and fixed-width files directly in SQL, eliminating the shell scripting layer entirely.
- Extend the toll pipeline with a **dbt transformation layer** downstream: Airflow loads to Bronze, dbt transforms to Silver/Gold — directly composing the DbtEngineer pattern.
- Package the log pipeline as an **Airflow sensor** that polls for new log files at a source URL, triggering the pipeline on new data rather than on a fixed schedule.
- Add **Airflow data quality operators** (`SQLCheckOperator`, `SQLValueCheckOperator`) to assert row counts and null rates after loading.

---

## Portfolio Signal

The combination of BashOperator and PythonOperator in the same DAG reflects a realistic production pattern, where legacy ETL scripts are wrapped in Airflow tasks alongside newer Python-based logic. It demonstrates practical, not idealistic, orchestration.
