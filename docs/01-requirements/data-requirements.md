# Data Requirements

The data domains Zavy captures, their expected characteristics, and the target volumes used for development and performance testing.

## Data Domains

### Customers
- Identity: unique customer ID, name, email (unique), phone, date of birth (optional), gender (optional).
- Registration: registration date, registration source.
- Segment: derived (`NEW`, `REGULAR`, `VIP`).
- Characteristics: hundreds of thousands of records; stable volume, growing with new registrations.

### Addresses
- Address lines, city, state/province, postal code, country, address type (billing/shipping), default flag.
- Characteristics: one or more per customer; updated over time (subject of SCD Type 2 tracking).

### Products
- Identity: unique product ID, SKU (unique), name, description.
- Classification: category, brand, status (`ACTIVE`/`INACTIVE`).
- Pricing: list price, cost.
- Characteristics: tens of thousands of records; price and cost change rarely; category/brand changes possible.

### Categories
- Unique category ID, name, optional parent (hierarchical), status.
- Characteristics: small, stable reference data (tens to hundreds of rows).

### Brands
- Unique brand ID, name.
- Characteristics: small, stable reference data (hundreds of rows).

### Sellers
- Identity: unique seller ID, seller name, email (unique), phone.
- Registration: registration date, location.
- Business: status (`ACTIVE`/`SUSPENDED`), commission rate (0–30%).
- Characteristics: ~1,000 records; stable but slow-growing.
- Note: the commission rate is used to compute Zavy's revenue (commission on seller sales), per `kpi-definitions.md`.

### Orders
- Identity: unique order ID, order number.
- Key references: customer, addresses (billing/shipping).
- Timing: order date (timestamp), status timestamps.
- Status: `PLACED`, `CONFIRMED`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`.
- Totals: subtotal, shipping, tax, total — where `subtotal` = sum of order-item line totals and `total` = `subtotal + shipping + tax`.
- Characteristics: ~1,000,000 records; the central transactional entity; continuously growing.

### Order Items
- Identity: unique order item ID.
- Key references: order, product.
- Values: quantity, unit price (snapshot at order time), unit cost (snapshot at order time), commission rate (snapshot at order time), line total.
- Note: unit cost and seller commission rate are snapshotted alongside unit price so that historical gross margin and Zavy commission revenue remain reproducible after product cost or seller commission changes.
- Characteristics: ~3,000,000 records (~3 items per order); one of the largest tables.

### Payments
- Identity: unique payment ID.
- Key references: order.
- Values: amount, payment method (`CARD`, `WALLET`, `BANK_TRANSFER`, `COD`), payment date, status (`PENDING`, `PAID`, `FAILED`, `REFUNDED`, `PARTIALLY_REFUNDED`).
- Characteristics: one or more records per order.

### Shipments
- Identity: unique shipment ID, carrier, tracking number.
- Key references: order, warehouse.
- Timing: ship date, delivery date.
- Status: `PENDING_PICKUP`, `IN_TRANSIT`, `DELIVERED`, `FAILED_DELIVERY`, `RETURNED_TO_SENDER`.
- Characteristics: one or more per order.

### Shipment Items
- Identity: unique shipment item ID.
- Key references: shipment, order item.
- Values: quantity shipped.
- Characteristics: one or more per shipment; represents how order items are allocated across shipments. Because partial fulfilment is allowed, an order item may appear in more than one shipment; the sum of shipment-item quantities for an order item must never exceed the order-item quantity.

### Warehouses
- Identity: unique warehouse ID, name, city, country.
- Characteristics: small, stable reference data (tens of records).

### Inventory
- Key: product + warehouse.
- Values: quantity on hand, last-updated timestamp.
- Characteristics: one row per product–warehouse combination.
- Note: this table holds current on-hand only. Historical inventory levels are reconstructed in the warehouse layer from stock movements into inventory snapshots (e.g., daily quantity on hand per product per warehouse) used for inventory turnover and out-of-stock analytics.

### Stock Movements
- Identity: unique stock movement ID, type (`RECEIPT`, `SALE`, `ADJUSTMENT`, `RETURN_IN`), quantity (signed), movement date, optional reference (e.g., order item).
- Characteristics: high volume; grows with every sale, receipt, and return.

### Returns
- Identity: unique return ID.
- Key references: order, customer, payment (refund link).
- Timing: request date, received date, refund date.
- Status: `REQUESTED`, `APPROVED`, `REJECTED`, `RETURNED`, `REFUNDED`.
- Characteristics: fraction of delivered orders (tens of thousands).

### Return Items
- Identity: unique return item ID.
- Key references: return, order item, product.
- Values: quantity, return reason (`WRONG_ITEM`, `DAMAGED`, `NOT_AS_DESCRIBED`, `SIZE_OR_FIT`, `CHANGED_MIND`, `OTHER`), condition on receipt.
- Characteristics: one or more per return.

### Reviews
- Identity: unique review ID.
- Key references: customer, product, order item.
- Values: rating (integer 1–5), comment (optional), created/updated timestamps.
- Characteristics: subset of delivered order items.

## Target Data Volumes

The initial dataset targets are:

| Entity | Target volume |
| --- | --- |
| Customers | 100,000 |
| Products | 50,000 |
| Sellers | 1,000 |
| Orders | 1,000,000 |
| Order items | ~3,000,000 |
| Shipment items | ~3,000,000 (aligned with order items) |
| Payments | ~1,000,000+ (including retries) |
| Stock movements | ~3,000,000+ |
| Returns | tens of thousands |
| Reviews | hundreds of thousands |

## Development vs Performance Datasets

Two dataset scales are used:

- **Development dataset (small):** a reduced set (e.g., 10,000 customers, 5,000 products, 100 sellers, 100,000 orders) used during development so that schema changes and pipeline runs are fast and easy to debug.
- **Benchmark dataset (full):** the target volumes above, used for performance testing, query optimization, indexing, and partitioning experiments.

Both datasets are generated by the same generator (with a configurable scale and a fixed random seed for reproducibility) so that behavior is identical and results are comparable.

## Derived/Enriched Data

The following values are derived rather than stored in the OLTP system and are computed in the data warehouse:
- GMV, seller sales, Zavy commission revenue, and net commission revenue (per `kpi-definitions.md`).
- Product gross margin (seller view).
- Customer segment (derived from order history).
- Product and seller performance metrics.
- Historical inventory snapshots, inventory turnover, and out-of-stock rates.