# Architecture Decision Records (ADRs)

This document records the foundational architecture decisions for the Zavy platform, including rationale, expected benefits, and trade-offs.

---

## ADR-001: Separation of OLTP and Analytical Workloads

* **Status**: Accepted
* **Context**: Zavy operates an e-commerce marketplace requiring reliable transaction processing alongside multi-year reporting and KPI analytics across millions of orders.
* **Decision**: Strictly separate the transactional database (`ecom_oltp`) from the analytical data warehouse (`dw`). Operational workflows never query the warehouse, and analytical/BI queries never hit the operational database directly.
* **Reason**: Running analytical aggregations, table scans, and multi-join queries against an operational transactional store causes CPU spikes, lock contention, buffer pool churn, and latency degradation for active customer checkouts.
* **Benefits**:
  * Complete resource and lock isolation between transactional writes and heavy analytical queries.
  * Schemas can be independently optimized: 3NF normalization for transactional integrity vs. Star Schema for analytical query performance.
  * Enables independent scaling, indexing, and backup strategies for each layer.
* **Trade-offs**:
  * Introduces data synchronization latency (pipeline batch interval).
  * Requires building, maintaining, and monitoring an ETL pipeline.
  * Increases storage requirements due to data duplication across OLTP and DW.

---

## ADR-002: PostgreSQL as the Operational Database and Initial Data Warehouse

* **Status**: Accepted
* **Context**: The platform needs a solid relational transactional database and an analytical store supporting complex SQL, partitioning, and indexing in a local development environment.
* **Decision**: Use PostgreSQL 16 as the database engine for both the operational store (`ecom_oltp`) and the initial data warehouse (`dw`), separated as distinct database instances/catalogs.
* **Reason**: PostgreSQL provides enterprise-grade ACID transactions, rich indexing methods (B-tree, Hash, GIN), declarative table partitioning, window functions, CTEs, and materialized views. Using the same engine family locally eliminates multi-database driver complexity while demonstrating advanced relational DBMS optimization techniques.
* **Benefits**:
  * Single database technology to install, configure, and maintain locally.
  * Full support for `EXPLAIN (ANALYZE, BUFFERS)` to benchmark query execution plans and index impact.
  * Native connector support across Python (`psycopg`, SQLAlchemy) and Power BI.
* **Trade-offs**:
  * PostgreSQL is a row-oriented relational engine, not a specialized MPP columnar data warehouse (e.g., Snowflake, BigQuery, ClickHouse). For extreme datasets (>50M rows), a dedicated columnar engine would scan faster.
  * Requires deliberate indexing, partitioning, and materialized views to achieve high performance on large analytical queries.

---

## ADR-003: Python and Pandas for Data Processing and ETL

* **Status**: Accepted
* **Context**: Data extracted from the OLTP layer must be validated, cleaned, transformed, reconciled, and loaded into the dimensional model.
* **Decision**: Implement the ETL pipeline in Python 3 using Pandas, SQLAlchemy, and `psycopg`.
* **Reason**: Python provides a clean, maintainable, and testable environment for business rule enforcement, schema validation, and SCD Type 2 logic. Writing ETL in Python allows unit testing with `pytest` and keeps transformation logic out of brittle database triggers or nested stored procedures.
* **Benefits**:
  * High developer productivity, readable code, and rich library ecosystem.
  * Vectorized batch transformations with Pandas for in-memory operations.
  * Clean separation of concerns between extraction, validation, and loading.
* **Trade-offs**:
  * Single-node processing is bounded by machine RAM and CPU cores.
  * Requires batch chunking if processing extremely large extraction deltas.

---

## ADR-004: Dedicated Staging Layer Between Extraction and Warehouse

* **Status**: Accepted
* **Context**: Directly transforming and loading extracted data into production warehouse tables during long-running extractions risks locking destination tables and complicates error recovery.
* **Decision**: Introduce an explicit staging layer (in-memory buffers / transient staging structures) between source extraction and warehouse loading.
* **Reason**: Decouples extraction from transformation. If validation or loading fails midway, the staged extraction batch can be inspected, quarantined, or retried without re-querying the operational database.
* **Benefits**:
  * Minimal read lock time on operational source tables.
  * Clear quarantine and audit boundary: source rows = processed rows + rejected rows.
  * Simplifies data quality logging and error diagnosis.
* **Trade-offs**:
  * Adds an intermediate processing step and transient memory overhead.

---

## ADR-005: Watermark-Based Incremental Data Ingestion

* **Status**: Accepted
* **Context**: Reloading the entire historical dataset on every ETL execution is computationally prohibitive as data volumes scale to millions of orders.
* **Decision**: Implement incremental extraction using a high-watermark approach driven by operational `updated_at` timestamps and a persistent pipeline metadata tracking table.
* **Reason**: Pulling only rows modified since the last successful extraction checkpoint keeps pipeline execution fast, scalable, and independent of total historical data size.
* **Benefits**:
  * Pipeline execution time scales with the volume of *new/changed* records, not total dataset size.
  * Idempotent re-runs: if a pipeline batch fails, the watermark is not advanced, allowing safe re-execution.
  * Dramatically reduces I/O load on the operational database.
* **Trade-offs**:
  * Requires source tables to maintain accurate, indexed `updated_at` timestamps.
  * Hard deletes in the operational database cannot be detected via simple timestamp watermarks without soft-delete flags or CDC logs.

---

## ADR-006: Slowly Changing Dimensions (SCD Type 2) Implemented in Warehouse

* **Status**: Accepted
* **Context**: Operational entities (e.g., customer addresses, seller locations) change over time. The business requires historical reporting to reflect the dimensional state active at the time an order occurred.
* **Decision**: Implement SCD Type 2 tracking exclusively in the Data Warehouse layer (`dw`) on selected dimensions (`dim_customer`, etc.). The OLTP database retains only the current operational state.
* **Reason**: Operational databases are optimized for fast writes of the current reality; forcing SCD Type 2 into OLTP complicates application CRUD queries and violates standard operational normalization. The data warehouse is the correct architectural home for historical dimension tracking.
* **Benefits**:
  * Keeps the OLTP schema clean, simple, and standard (3NF current state).
  * Fact records in the warehouse link to the exact surrogate key representing the dimensional state when the order was placed.
  * Fully supports historical point-in-time and cohort reporting.
* **Trade-offs**:
  * The transformation pipeline must maintain surrogate key mapping tables, check incoming natural keys for attribute changes, and expire old dimension records during every load.

---

## ADR-007: Treating Streaming CDC, Spark, and DuckDB as Optional Future Extensions

* **Status**: Accepted
* **Context**: Modern data architectures often incorporate distributed engines (Spark), streaming brokers (Kafka/Debezium), or embedded columnar engines (DuckDB).
* **Decision**: Explicitly classify Kafka, Debezium, Spark, and DuckDB as optional future extensions. Keep the baseline production-style pipeline built entirely on PostgreSQL and Python.
* **Reason**: The core requirements of the platform (1M orders, 3M line items, Star Schema, SCD Type 2, indexing/partitioning benchmarks, Power BI reporting) are fully achievable, faster to develop, and easier to reproduce on a local machine using PostgreSQL 16 and Python 3.13. Introducing distributed streaming infrastructure prematurely adds high operational overhead without changing the fundamental analytical outcomes.
* **Benefits**:
  * Clean, reproducible local environment that runs without heavy cluster dependencies.
  * Clear architecture boundaries make it straightforward to swap components later (e.g., replacing the timestamp extractor with a Debezium WAL reader, or swapping Pandas with PySpark).
* **Trade-offs**:
  * Batch synchronization interval (e.g., minutes/hours) rather than sub-second streaming replication.
