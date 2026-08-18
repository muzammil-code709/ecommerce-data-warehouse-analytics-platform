# Business Context — Zavy

## 1. Zavy Business Overview

**Zavy** is a simulated multi-seller e-commerce marketplace. It connects **customers** who buy products with **sellers** who list and sell them. Zavy operates the marketplace platform: it manages the catalog, orders, payments, fulfilment, and provides analytics to the business.

Zavy does **not** own the products sold on the platform. Sellers own and list their products; Zavy provides the storefront, transaction processing, and fulfilment coordination. This is the defining characteristic of a multi-seller marketplace business model.

All data used in this project is **synthetically generated** for educational and portfolio purposes. Zavy is a fictional business and no real customer, seller, or transaction data is used.

## 2. Business Model

- **Multi-seller marketplace**: many independent sellers list products; customers buy from any seller through one platform.
- **Commission-based**: Zavy's revenue is the **commission** it earns on seller sales, modelled via a per-seller commission rate. The product selling price belongs to the seller; Zavy never records it as its own revenue.
- **Marketplace economics**: total customer spend on the platform is **Gross Merchandise Value (GMV)**, also called **Seller Sales**. It is the sellers' sales value. Zavy's own revenue is the commission earned on that GMV, net of commission refunded on returns. Zavy does not incur product costs (COGS belongs to sellers), so platform profit is not modelled.
- **Product catalogue**: products belong to categories and brands and are stored across a network of warehouses.
- **Fulfilment**: Zavy coordinates inventory and shipment so that a single customer order may be fulfilled from one or more warehouses.
- **Customer acquisition**: customers register, maintain addresses, place orders, pay, receive shipments, may return items, and may leave product reviews.

## 3. Main Business Processes

1. **Registration** — customers register and manage addresses; sellers onboard and list products.
2. **Browsing & ordering** — customers place orders containing one or more product line items.
3. **Payment** — each order generates one or more payment records; fulfilment proceeds once payment is accepted (or at delivery for cash on delivery).
4. **Fulfilment & shipment** — inventory is reserved and reduced; items are picked and shipped from warehouses to the customer.
5. **Delivery** — shipments progress through delivery states until the order is delivered.
6. **Returns** — customers may request returns of delivered items within the return window; approved returns are received, restocked, and refunded.
7. **Reviews** — customers may review purchased and delivered products.
8. **Inventory management** — stock levels change through receipts, sales, adjustments, and returned items.
9. **Analytics & reporting** — transactional data is processed into a data warehouse to support business insight.

## 4. Main Business Domains

The platform is organized into the following business domains:

| Domain | Description |
| --- | --- |
| Customers | Customer identity, contact details, and addresses |
| Products | Catalogue: products, categories, and brands |
| Sellers | Seller registration, status, and performance |
| Orders | Orders and their line items |
| Payments | Payment attempts and statuses |
| Shipments | Fulfilment and delivery of order items |
| Inventory | Warehouses, stock levels, and stock movements |
| Returns | Return requests, returned items, and refunds |
| Reviews | Customer feedback on products |
| Analytics | Analytical data, KPIs, and reporting |

## 5. Transactional vs Analytical Workloads

The platform serves two very different workloads:

**Transactional workload (OLTP)**
- Supports day-to-day operations: placing orders, processing payments, updating inventory.
- High volume of short, frequent reads/writes on individual records.
- Strong focus on data integrity: primary keys, foreign keys, constraints, transactions.
- Normalized schema (3NF) to avoid redundancy and keep writes fast.

**Analytical workload (OLAP)**
- Answers business questions: revenue trends, customer lifetime value, product and seller performance.
- Large scans and aggregations across millions of rows.
- Fewer, heavier queries that read large portions of history.
- Dimensional schema (star schema) designed for fast aggregation.

Running heavy analytical queries directly against the operational database would increase database load, slow down customer transactions, and make historical analysis difficult. Zavy therefore separates the two workloads.

## 6. Why Zavy Needs a Separate Analytical Platform

1. **Performance isolation** — analytical queries no longer compete with customer-facing transactions for the same resources.
2. **Historical analysis** — the operational database is optimized for current state; the warehouse stores history and preserves changes over time (e.g., Slowly Changing Dimensions).
3. **Query efficiency** — a dimensional model with pre-aggregated views answers analytical questions far faster than normalized operational tables.
4. **Clean reporting surface** — analysts query a consistent, validated, business-ready dataset instead of raw operational data.
5. **Scalability** — the two layers can be sized, tuned, and optimized independently.

## 7. Planned Platform

The eventual platform separates data into distinct layers:

```
Operational data  →  PostgreSQL OLTP  →  Incremental extraction  →  Processing & validation  →  Data warehouse (star schema)  →  Analytical SQL  →  Power BI  →  Business insights
```

The initial implementation uses **PostgreSQL** and **Python**. Advanced technologies (Kafka, Debezium, Spark, DuckDB) are **optional** extensions and are not assumed to be mandatory.