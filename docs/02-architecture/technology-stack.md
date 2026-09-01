# Technology Stack

This document details the technologies selected for the Zavy platform, why each was chosen, and how the core local stack is positioned relative to optional future scaling technologies.

---

## 1. Core Technology Stack

| Layer / Concern | Technology | Version / Tooling | Primary Role in Platform |
| --- | --- | --- | --- |
| **OLTP Database** | PostgreSQL | 16.x | Normalized transactional store (`ecom_oltp`) with ACID compliance, constraints, and operational indexing. |
| **Data Processing Runtime** | Python | 3.13.x | Pipeline orchestration, data validation engine, business rule enforcement, and SCD logic. |
| **Data Manipulation** | Pandas | 2.x / 3.x | In-memory staging transformations, structural cleaning, schema mapping, and dataset reconciliation. |
| **Database Connectivity** | SQLAlchemy / `psycopg` | SQLAlchemy 2.0 / `psycopg` 3.x | High-throughput connection pooling, parameterized queries, and batch data loading. |
| **Data Warehouse** | PostgreSQL | 16.x | Analytical star-schema database (`dw`) with surrogate keys, table partitioning, and indexing. |
| **Semantic & Analytics Engine** | PostgreSQL SQL | SQL (PostgreSQL dialect) | Analytical queries (window functions, CTEs, rollups), standard views, and materialized views. |
| **Business Intelligence** | Power BI | Microsoft Power BI Desktop | Interactive reporting, executive dashboards, drill-down visualizations, and domain analytics. |
| **Version Control** | Git / GitHub | Git CLI / GitHub Repo | Codebase versioning, schema versioning, issue tracking, and documentation hosting. |
| **Containerization** | Docker | Docker / Docker Compose | Optional containerized database provisioning and standardized environment setup. |
| **Documentation** | Markdown | GitHub Flavored Markdown | Architecture specifications, data dictionaries, benchmark reports, and setup runbooks. |

---

## 2. Rationales for Core Choices

### 2.1 PostgreSQL 16 (OLTP and Initial Data Warehouse)
* **ACID Guarantees and Integrity**: PostgreSQL provides industry-standard relational enforcement (PK/FK, CHECK constraints, composite indexes, transaction isolation levels).
* **Analytical SQL Capabilities**: PostgreSQL supports advanced SQL standards including common table expressions (CTEs), multi-level window functions, `ROLLUP`/`CUBE`, declarative table partitioning, and materialized views with concurrent refresh.
* **Unified Tooling**: Using PostgreSQL for both operational and warehouse layers in a local development environment minimizes operational complexity while maintaining strict logical database separation (`ecom_oltp` vs `dw`).
* **Execution Plan Visibility**: `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` allows precise measurement of memory buffers, sequential vs index scans, join algorithms (Hash vs Nested Loop vs Merge), and partition pruning.

### 2.2 Python 3.13 & Pandas
* **Rich Data Processing Ecosystem**: Python is the industry standard for data engineering logic. It provides built-in typing, mature testing frameworks (`pytest`), and standard database client drivers.
* **Vectorized Data Transformations**: Pandas enables efficient in-memory operations for batch deduplication, type coercion, null value resolution, and custom business rule validations.
* **Maintainable Pipeline Logic**: Clear Python modules for extraction, validation, and loading are easier to debug, test, and document than complex multi-nested stored procedures.

### 2.3 SQLAlchemy 2.0 & `psycopg` 3
* **Robust Connection Management**: SQLAlchemy provides engine-level connection pooling and connection life-cycle management.
* **High-Speed Binary Transfer**: `psycopg` 3 supports native binary protocol transfers, server-side cursors for large streaming extractions, and fast `COPY` operations for bulk loading fact tables.

### 2.4 Power BI
* **Industry Standard BI**: Native support for PostgreSQL connections, flexible data modeling, DAX measures, and rich interactive visual components.
* **Role-Specific Dashboards**: Easily structures multi-page reporting tailored for executives, sales managers, product category leads, and inventory controllers.

### 2.5 Git and Markdown
* **Reproducibility**: All schema DDLs, ETL scripts, benchmark queries, and test assertions are versioned in Git.
* **Living Documentation**: Markdown alongside code ensures that data dictionaries and architecture records stay updated through code reviews.

---

## 3. Optional Future Extensions

The core platform is designed to be fully functional on a local machine using only PostgreSQL and Python. Advanced distributed or streaming technologies are explicitly treated as **optional extensions**:

- Kafka & Debezium (real-time CDC)
- Apache Spark / PySpark (distributed ETL)
- DuckDB (embedded local OLAP)

### 3.1 Apache Kafka & Debezium (Optional Streaming CDC)
* **Role**: Real-time event capture directly from PostgreSQL WAL logs into message topics.
* **When Justified**: When business requirements transition from scheduled batch processing (e.g., hourly/daily) to sub-minute real-time warehouse synchronization.
* **Local Trade-off**: Adds substantial operational overhead (Zookeeper/KRaft, Kafka brokers, Kafka Connect workers) not required for the core batch-oriented portfolio project.

### 3.2 Apache Spark / PySpark (Optional Distributed Processing)
* **Role**: Distributed compute engine for massive dataset transformations.
* **When Justified**: When data volumes exceed single-node memory capacity (>10–50 million orders).
* **Local Trade-off**: Requires JVM overhead, cluster management, and complex local configuration. Single-node Pandas / SQL processing is far more efficient for datasets under 10 million rows.

### 3.3 DuckDB (Optional Local Analytical Engine)
* **Role**: High-performance vectorized columnar OLAP engine that runs embedded inside the Python process.
* **When Justified**: If direct Parquet file querying or sub-second local OLAP benchmarks are added as a comparative study against PostgreSQL DW performance.
* **Local Trade-off**: PostgreSQL DW is already part of the core project scope; introducing DuckDB adds a second analytical query engine that is optional.
