# Non-Functional Requirements

Quality attributes for a locally-run system on PostgreSQL. The targets are meant to be measurable rather than aspirational.

## Performance

- **NFR-001** — Standard analytical queries against the benchmark dataset must return results in a reasonable time for interactive reporting (target: most dashboard queries within 30 seconds on reference hardware).
- **NFR-002** — Transactional writes on the OLTP database must remain fast and must not be degraded by analytical workloads running concurrently on the warehouse.
- **NFR-003** — The benefit of optimization techniques (indexes, partitioning, materialized views) must be measurable via `EXPLAIN ANALYZE` and documented through before/after benchmarks.

## Scalability

- **NFR-004** — The platform must handle the target data volume defined in `data-requirements.md` (100,000 customers, 50,000 products, 1,000 sellers, 1,000,000 orders, ~3,000,000 order items).
- **NFR-005** — The platform must support larger volumes used for performance testing without redesign (larger datasets must be generated with the same generators).
- **NFR-006** — Incremental loading must scale with the number of new records rather than with total data volume.

## Reliability

- **NFR-007** — The pipeline must be idempotent: re-running it over the same input must not produce duplicates or corrupt the warehouse.
- **NFR-008** — A failed pipeline run must be recoverable by re-running from the last successful checkpoint.
- **NFR-009** — Referential integrity in both the OLTP database and the warehouse must be verifiable by automated checks after every load.

## Data Quality

- **NFR-010** — Row reconciliation must be performed for every load: source rows must equal processed rows plus rejected rows.
- **NFR-011** — Validation rules must be applied consistently to every batch; rejected records must be counted and inspectable.
- **NFR-012** — KPIs produced from the warehouse must be reproducible: the same input data must always produce the same KPI values.

## Maintainability

- **NFR-013** — The codebase must be organized by clear module boundaries (database, ETL, analytics, tests) with consistent naming and documentation.
- **NFR-014** — Configuration (database credentials, connection settings) must live outside source code in environment files.
- **NFR-015** — Schema and pipeline scripts must be version-controlled and re-runnable.

## Security

- **NFR-016** — Database credentials must not be committed to the repository; they are supplied via environment variables.
- **NFR-017** — The system must use synthetic data only; no real personal or payment information is used.
- **NFR-018** — Database access requires authentication (scram-sha-256) on every connection.

## Reproducibility

- **NFR-019** — The data generator must accept a seed so that the same dataset can be regenerated identically.
- **NFR-020** — Installation and setup must be documented so that a new environment can be reproduced from the documentation alone.

## Observability

- **NFR-021** — Pipeline runs must record timestamps, row counts (source/processed/rejected), and status to allow tracing and auditing.
- **NFR-022** — Query performance measurements (execution time, planning time, rows scanned) must be recorded so that optimization work can be compared.