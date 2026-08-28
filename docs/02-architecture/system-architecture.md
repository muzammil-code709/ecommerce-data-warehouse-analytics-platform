# System Architecture

This document describes the end-to-end architecture of the Zavy E-Commerce Data Warehouse & Analytics Platform. It outlines the logical layers, workload boundaries, operational responsibilities, and how data travels from transactional generation to analytical reporting.

---

## 1. Architectural Overview

Zavy is a multi-seller e-commerce marketplace. The platform decouples operational transaction processing from analytical workloads to ensure that heavy reporting queries do not degrade transactional performance, and that business metrics remain historically accurate and reproducible.

```text
       ZAVY OPERATIONS
 (Customers | Sellers | Orders)
               │
               ▼
     POSTGRESQL OLTP LAYER
   (ecom_oltp - Normalized 3NF)
               │
               ▼
    INCREMENTAL EXTRACTION
   (Watermarks & Checkpoints)
               │
               ▼
         STAGING LAYER
     (Transient Ingestion)
               │
               ▼
   VALIDATION & TRANSFORMATION
     (Python / Pandas / Rules)
               │
               ▼
   POSTGRESQL DATA WAREHOUSE
      (dw - Star Schema)
               │
               ▼
    ANALYTICAL SQL LAYER
  (Views / Materialized Views)
               │
               ▼
       POWER BI DASHBOARDS
    (Executive & Domain BI)
               │
               ▼
        BUSINESS INSIGHTS
```

---

## 2. Layer Specifications

### 2.1 Operational Source Layer (Zavy Operations)
* **Purpose**: Simulates marketplace user interactions across buyers, sellers, and platform services.
* **Responsibilities**: Emits operational events: customer registrations, catalog listings, order placement, payments, warehouse inventory changes, shipments, returns, and reviews.
* **Input**: Operational events from synthetic event generators and business process workflows.
* **Output**: Atomic database writes into the transactional database.
* **Technology**: Python synthetic data generation workflows.
* **Interactions**: Direct transactional interface with the PostgreSQL OLTP layer.

### 2.2 Transactional Database Layer (PostgreSQL OLTP)
* **Purpose**: Serves as the single operational source of truth (`ecom_oltp`), optimized for fast ACID transactions and data integrity.
* **Responsibilities**:
  * Enforces relational constraints (PK, FK, UNIQUE, NOT NULL, CHECK).
  * Stores normalized data (3NF) across customers, addresses, products, categories, brands, sellers, orders, order items, payments, shipments, shipment items, warehouses, inventory, stock movements, returns, return items, and reviews.
  * Captures order-time pricing snapshots (`unit_price`, `unit_cost`, `commission_rate`) on order line items.
  * Tracks operational record timestamps (`created_at`, `updated_at`).
* **Input**: Transactional CRUD operations from marketplace workflows.
* **Output**: Structured operational tables with transactional guarantees.
* **Technology**: PostgreSQL 16 (`ecom_oltp`).
* **Interactions**: Read by the Incremental Extraction pipeline; written to by operational processes.

### 2.3 Incremental Extraction Layer
* **Purpose**: Pulls new and modified records from the OLTP database without performing full table dumps on each pipeline execution.
* **Responsibilities**:
  * Tracks pipeline watermarks (high-watermark timestamps and sequential checkpoints).
  * Queries source tables for records where `updated_at > last_watermark`.
  * Guarantees reproducible extraction batches and records extraction metadata (start time, end time, row counts).
* **Input**: High-watermark metadata and source OLTP tables.
* **Output**: Raw incremental record batches.
* **Technology**: Python 3.13, SQLAlchemy, `psycopg`.
* **Interactions**: Reads from `ecom_oltp` using indexed timestamp columns; passes record batches to the Staging Layer.

### 2.4 Staging Layer
* **Purpose**: Provides a decoupled, transient landing zone for extracted data prior to validation and transformation.
* **Responsibilities**:
  * Isolates extraction from transformation, preventing long-running transactions on source tables.
  * Holds raw records in memory and transient staging structures without enforcing warehouse constraints.
  * Enables pipeline replay and auditing if transformation fails midway.
* **Input**: Raw incremental extraction batches.
* **Output**: Staged batch datasets ready for validation rules.
* **Technology**: Python data structures / Pandas in-memory staging DataFrames.
* **Interactions**: Receives extracted records from Extraction; feeds into the Validation & Transformation engine.

### 2.5 Validation & Transformation Layer
* **Purpose**: Cleans, standardizes, validates, and models incoming staged records into analytical dimensions and facts.
* **Responsibilities**:
  * Applies business rules (e.g., non-negative stock, valid pricing, valid status transitions).
  * Performs data quality checks: schema enforcement, null checks, range validations, and foreign key integrity verification.
  * Separates conforming records from rejected records and logs quarantine counts.
  * Implements Slowly Changing Dimension (SCD) Type 2 logic for tracking historical changes (e.g., customer address changes).
  * Reconstructs daily warehouse-level historical inventory snapshots from stock movements.
  * Computes derived analytical fields (e.g., line-item GMV, commission amounts, profit margins).
* **Input**: Staged raw datasets and previous dimension states.
* **Output**: Validated, conforming dimension records (current + historical versions) and fact records.
* **Technology**: Python 3.13, Pandas, custom validation rules engine.
* **Interactions**: Reads staged data; queries existing warehouse dimensions for surrogate key lookups and SCD comparison; outputs clean datasets to the Warehouse Loading engine.

### 2.6 Data Warehouse Layer (PostgreSQL DW)
* **Purpose**: Stores historical, clean, and modeled analytical data (`dw`) organized into a Star Schema optimized for high-performance aggregations and multi-dimensional analysis.
* **Responsibilities**:
  * Houses dimension tables (`dim_customer`, `dim_product`, `dim_seller`, `dim_date`, `dim_location`, `dim_payment`) with surrogate keys.
  * Preserves historical dimension attributes via SCD Type 2 fields (`effective_date`, `end_date`, `is_current`).
  * Houses fact tables (`fact_sales`, `fact_returns`, `fact_inventory`) with documented grain definitions.
  * Maintains periodic inventory snapshot tables for point-in-time stock calculations.
  * Applies physical optimizations: B-tree indexes, composite indexes, covering indexes, and table partitioning on large transactional facts (e.g., by order date).
* **Input**: Validated dimension and fact datasets from the transformation layer.
* **Output**: Star-schema analytical tables.
* **Technology**: PostgreSQL 16 (`dw`).
* **Interactions**: Loaded by the Python ETL pipeline; queried by the Analytical SQL and BI layers.

### 2.7 Semantic & Analytical SQL Layer
* **Purpose**: Exposes clean analytical interfaces, pre-computed KPI aggregations, and business metrics for reporting.
* **Responsibilities**:
  * Implements advanced SQL queries answering core business questions (window functions, CTEs, rollups, YoY growth, cohort retention).
  * Houses standard analytical views and materialized views for computationally expensive aggregations (e.g., monthly GMV, seller performance summaries).
  * Encapsulates business logic and KPI formulas defined in `kpi-definitions.md`.
* **Input**: Raw fact and dimension tables from `dw`.
* **Output**: Business-ready analytical datasets, KPI views, and query result sets.
* **Technology**: PostgreSQL SQL, Views, Materialized Views.
* **Interactions**: Sits directly on top of the Data Warehouse; serves as the primary data source for Power BI dashboards.

### 2.8 Business Intelligence Layer (Power BI)
* **Purpose**: Delivers interactive visualization, executive dashboards, drill-down analytics, and trend exploration for business stakeholders.
* **Responsibilities**:
  * Consumes semantic views and analytical tables from the data warehouse.
  * Provides specialized dashboards: Executive Overview, Sales Analysis, Customer Analytics, Product & Inventory, and Seller Performance.
  * Provides interactive filtering by date, region, category, brand, and seller.
* **Input**: SQL queries, views, and materialized views from `dw`.
* **Output**: Interactive visual reports, executive summaries, and KPI cards.
* **Technology**: Microsoft Power BI Desktop / Service (PostgreSQL database connector).
* **Interactions**: Connects via PostgreSQL driver to `dw` semantic views.

---

## 3. Workload Separation Analysis

To guarantee system stability, Zavy strictly separates four distinct computational workloads:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                             WORKLOAD TAXONOMY                               │
├───────────────────┬─────────────────────────────────────────────────────────┤
│ Workload          │ Primary Characteristics & Resource Profile              │
├───────────────────┼─────────────────────────────────────────────────────────┤
│ 1. OLTP           │ • High-frequency, low-latency row-level writes & reads  │
│    (ecom_oltp)    │ • Strict ACID guarantees, row-level locks, 3NF schema   │
│                   │ • Small memory footprint per transaction                │
├───────────────────┼─────────────────────────────────────────────────────────┤
│ 2. Data Processing│ • Batch CPU & memory usage for cleaning and transforms  │
│    (Python ETL)   │ • Out-of-database processing (Pandas / Python runtime)  │
│                   │ • Controlled extraction rate via timestamp watermarks   │
├───────────────────┼─────────────────────────────────────────────────────────┤
│ 3. OLAP           │ • Large sequential scans, hash joins, big aggregations  │
│    (dw)           │ • Read-heavy star schema, surrogate keys, partitioning  │
│                   │ • Materialized views and indexing tuned for query plans │
├───────────────────┼─────────────────────────────────────────────────────────┤
│ 4. BI Reporting   │ • Interactive dashboards, filter queries, slice-and-dice│
│    (Power BI)     │ • Reads semantic views / materialized aggregations      │
│                   │ • Decoupled from operational database entirely          │
└───────────────────┴─────────────────────────────────────────────────────────┘
```

### Why Separation is Mandatory
1. **Concurrency and Lock Contention**: Operational inserts, updates, and lock acquisitions on orders and inventory never compete with analytical table scans.
2. **Schema Optimization**: OLTP tables eliminate data redundancy (3NF); OLAP tables eliminate query-time joins across transactional normalization boundaries (Star Schema).
3. **Historical State Preservation**: The operational database only retains current state (e.g., customer's current address). The data warehouse manages dimensional history (SCD Type 2) and point-in-time facts.
4. **Independent Resource Scaling**: Transactional disk I/O and analytical compute resources can be tuned, indexed, partitioned, or migrated independently.

---

## 4. Architectural Boundaries and Extension Points

The core architecture uses **PostgreSQL 16** and **Python 3.13**. All components are designed with explicit architectural boundaries that allow optional future extensions without re-architecting the system:

* **Optional Change Data Capture (CDC)**: The incremental extraction layer currently uses indexed timestamp watermarks. If real-time streaming is introduced later, a Debezium/Kafka CDC connector can plug into PostgreSQL's write-ahead log (WAL) without changing the downstream warehouse schema.
* **Optional Distributed Processing**: For larger data volumes (>10M rows), the Python/Pandas processing modules can be replaced with PySpark jobs using the same staging and transformation logic.
* **Optional Embedded OLAP Engine**: DuckDB can be introduced as an in-process analytical query accelerator between the staging files and BI layer if local analytical execution speed needs to be benchmarked against PostgreSQL.
