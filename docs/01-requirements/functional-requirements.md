# Functional Requirements

This document defines what the Zavy platform must be able to do. Requirements are uniquely identified and grouped by capability area. They are written at the platform level and do not lock in implementation details (the underlying technologies are PostgreSQL and Python).

## Data Storage (OLTP)

- **FR-001** — The platform must store transactional data for customers, addresses, products, categories, brands, sellers, orders, order items, payments, shipments, warehouses, inventory, stock movements, returns, return items, and reviews.
- **FR-002** — Each transactional entity must have a primary key, and relationships between entities must be enforced with foreign keys.
- **FR-003** — The platform must enforce data integrity constraints: NOT NULL, UNIQUE, and CHECK constraints where defined in the business rules.
- **FR-004** — The platform must record creation and update timestamps so that data changes can be tracked over time.

## Relationships and Integrity

- **FR-005** — Orders must reference the customer who placed them; order items must reference the order and the product; shipment items must reference the shipment and the order item; and payments, returns, and reviews must reference their parent orders/items.
- **FR-006** — The platform must reject or quarantine records that violate referential integrity during data loading.
- **FR-007** — The platform must prevent the deletion of operational records that are referenced by other records (e.g., a product that appears in order items).

## Incremental Extraction

- **FR-008** — The platform must identify and extract only records that are new or changed since the last load.
- **FR-009** — The extraction process must use a watermark (e.g., max timestamp processed) so that progress is tracked across runs.
- **FR-010** — The extraction process must be repeatable: re-running it must not duplicate records already loaded.

## Validation and Transformation

- **FR-011** — The pipeline must validate required fields, data types, date ranges, prices, quantities, and referential integrity before loading into the warehouse.
- **FR-012** — The pipeline must detect and handle duplicate records according to defined rules.
- **FR-013** — The pipeline must standardize values (e.g., category and status naming) to consistent business terms.
- **FR-014** — Records that fail validation must be rejected and counted, and must not silently disappear.

## Warehouse Loading

- **FR-015** — The platform must load data into a star-schema warehouse with dimension and fact tables.
- **FR-016** — The platform must load dimension tables before fact tables so that foreign keys resolve correctly.
- **FR-017** — The warehouse must use upsert logic (insert or update) so that repeated loads do not create duplicate rows.

## Historical Records

- **FR-018** — The **data warehouse** must preserve historical attribute changes using Slowly Changing Dimension (SCD) Type 2 logic, keeping both the old and new versions of changed records. The OLTP database is not required to maintain history; it stores current operational state only.
- **FR-019** — Every SCD record must track validity start date, end date, and a current-record flag.
- **FR-020** — Fact records must reference the correct dimension version in effect at the time of the event.

## Analytical Queries

- **FR-021** — The platform must support the analytical queries defined in `business-questions.md`.
- **FR-022** — The platform must provide views and materialized views that pre-compute frequently used aggregations.
- **FR-023** — The platform must support time-based analysis (daily, monthly, yearly) and period-over-period comparison.

## KPI Generation

- **FR-024** — The platform must compute all KPIs defined in `kpi-definitions.md` from warehouse data.
- **FR-025** — KPI calculations must follow the exact definitions and inclusion/exclusion rules documented in `kpi-definitions.md`.
- **FR-031** — The platform must compute Gross Merchandise Value, Seller Sales, and Zavy commission revenue as separate measures, per the marketplace economics in `kpi-definitions.md`.
- **FR-032** — The warehouse must maintain historical inventory snapshots (e.g., daily quantity on hand per product per warehouse) to support inventory turnover and out-of-stock analytics.

## Power BI Reporting

- **FR-026** — The platform must expose warehouse data to Power BI in a form suitable for dashboarding.
- **FR-027** — Dashboards must cover executive overview, sales, customer analytics, product/inventory, and seller performance.

## Pipeline and Data-Quality Tracking

- **FR-028** — Each pipeline run must record its start and end time, source counts, processed counts, and rejected counts.
- **FR-029** — The platform must run data-quality checks after loading (e.g., row reconciliation between source and warehouse) and report failures.
- **FR-030** — Pipeline and data-quality results must be inspectable so that problems can be diagnosed and traced.

## Split Fulfilment

- **FR-033** — The platform must record shipment-item allocation so that split fulfilment of order items across shipments is represented.