# Scope

This document defines what is included in the Zavy platform project and what is explicitly excluded.

## In Scope

The project delivers a complete, portfolio-grade data and analytics platform:

**OLTP database**
- A normalized transactional database (PostgreSQL) for customers, products, sellers, orders, order items, payments, shipments, warehouses, inventory, stock movements, returns, return items, and reviews.
- Enforced constraints, primary/foreign keys, indexes, and seed-data generation at the target volumes in `data-requirements.md`.

**Data processing**
- An ETL/ELT pipeline (Python) that extracts, stages, validates, transforms, and loads data into the warehouse.
- Incremental loading with watermark tracking, upserts, and deduplication.
- Slowly Changing Dimension (SCD) Type 2 handling to preserve history.

**Data warehouse**
- A star-schema warehouse (PostgreSQL) with dimension and fact tables, documented grain, and analytical views/materialized views.

**Analytics**
- Analytical SQL queries answering the business questions in `business-questions.md`.
- KPI calculations following `kpi-definitions.md`.

**Optimization**
- Query optimization with indexing, `EXPLAIN ANALYZE` analysis, and before/after benchmarking.
- Partitioning of large tables and analysis of partition pruning.

**Business intelligence**
- Power BI dashboards covering executive overview, sales, customer, product/inventory, and seller performance.

**Testing and documentation**
- Tests for database constraints, ETL correctness, and reconciliation.
- Performance benchmark reports.
- Professional documentation (architecture, database design, data dictionary, ETL, SQL, data quality, dashboard guide, setup guide).

## Out of Scope

The following are explicitly **not** part of this project:

- Real customer transactions — all data is synthetic and generated.
- Real payment processing or payment gateway integration — payments are simulated records only.
- Real shipping carrier integration — shipment data is simulated.
- A production e-commerce frontend or storefront — the project is a data and analytics platform, not a shopping application.
- Production cloud deployment — the system runs locally (PostgreSQL and Python); cloud infrastructure is not required.
- Real customer personal data — no real personal information is used; all identities are fabricated.
- Machine-learning features — predictive or ML models are out of scope unless added as an optional extension.
- Advanced streaming infrastructure — Kafka, Debezium, Spark, and DuckDB are optional extensions and are not part of the core delivery.
- Microservices architecture — the platform is built as a cohesive local system, not a distributed production service.
- Zavy platform operating costs and platform profit — product COGS belongs to sellers and no Zavy operating-cost data is modelled, so a platform profit figure is not computed.