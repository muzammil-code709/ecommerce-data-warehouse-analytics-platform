# KPI Definitions

Each KPI below is defined precisely enough that any query or dashboard produces the same value. These definitions are the source of truth for the analytics layer.

## Marketplace Economics

Zavy is a multi-seller marketplace: sellers own the products and Zavy earns a commission on seller sales. Every money KPI in this document follows from that split:

- **GMV (Gross Merchandise Value)** — the total value customers spend on the platform. Also called **Seller Sales**. It is the sellers' revenue, not Zavy's.
- **Zavy Commission Revenue** — the commission Zavy earns on seller sales (per-seller commission rate applied to each order item).
- **Net Commission Revenue** — Zavy commission revenue after refunding commission on received returns.
- **Platform Profit** — not defined. Zavy does not incur product COGS (that is the seller's cost) and no Zavy operating-cost data is modelled, so a platform profit cannot be computed. Product-level margin is tracked as a seller-view metric only.

## Global Conventions

These conventions apply to every KPI unless a specific KPI states otherwise:

| Term | Definition |
| --- | --- |
| Recognized order | An order with status `CONFIRMED`, `PROCESSING`, `SHIPPED`, or `DELIVERED`. Orders in `PLACED` or `CANCELLED` status are excluded. |
| Recognized order item | A line item belonging to a recognized order. |
| Sold quantity | The sum of `quantity` over recognized order items. |
| Returned quantity | The sum of `quantity` over return items whose return is approved and received (return status `RETURNED` or `REFUNDED`). Rejected and pending returns are excluded. |
| Unit price | The unit price captured on the order item at the time of the order. |
| Unit cost | The product cost captured on the order item at order time (snapshot), so gross margin stays reproducible after product cost changes. Used only for the seller-view product gross margin KPI. |
| Commission rate | The seller's commission rate (0–30%) captured on the order item at order time (snapshot), so commission revenue stays reproducible after the seller's rate changes. |
| Returned value | `Σ (return_item.quantity × order_item.unit_price)` for returns with status `RETURNED` or `REFUNDED`. |
| Reporting period | The time interval the KPI is computed for (day, month, quarter, or year) unless stated otherwise. |

## Marketplace Value KPIs

### 1. Gross Merchandise Value (GMV / Seller Sales)

- **Definition:** Total value of customer purchases on the platform before returns are deducted. This is the sellers' sales value, not Zavy's revenue.
- **Formula:** `Σ (order_item.quantity × order_item.unit_price)` over recognized order items.
- **Business meaning:** Platform sales volume; the headline growth metric.
- **Inclusion/exclusion rules:**
  - Includes only recognized orders (`CONFIRMED`+); `PLACED`-only and `CANCELLED` orders are excluded.
  - Uses the unit price at order time, not the current product price.
  - Returned items are included here and deducted when reporting GMV net of returns.

### 2. Zavy Commission Revenue

- **Definition:** The commission Zavy earns on seller sales before returns.
- **Formula:** `Σ (order_item.quantity × order_item.unit_price × order_item.commission_rate)` over recognized order items.
- **Business meaning:** Zavy's actual top-line revenue.
- **Inclusion/exclusion rules:**
  - Uses the commission rate snapshotted on each order item at order time (not the seller's current rate).
  - Derived directly from GMV; returned-item commission is refunded (see Net Commission Revenue).

### 3. Net Commission Revenue (Zavy Net Revenue)

- **Definition:** Commission revenue after refunding commission on approved and received returns.
- **Formula:** `Zavy Commission Revenue − Σ (return_item.quantity × order_item.unit_price × order_item.commission_rate)` over received returns (`RETURNED` or `REFUNDED`).
- **Business meaning:** The commission revenue Zavy actually keeps.
- **Inclusion/exclusion rules:**
  - Deducts only approved and received returns; pending and rejected returns do not reduce commission.

### 4. Revenue Growth

- **Definition:** Percentage change in a money measure between two consecutive periods.
- **Formula:** `(value(current period) − value(previous period)) / value(previous period) × 100`.
- **Business meaning:** Shows whether the business is growing or shrinking.
- **Inclusion/exclusion rules:**
  - Default basis is GMV; the same formula may be applied to Zavy Commission Revenue for the Zavy-view growth.
  - The basis and default period (month) must be stated with the result.
  - Undefined (blank) when the previous-period value is zero.

## Sales Volume KPIs

### 5. Orders (Order Count)

- **Definition:** Number of recognized orders in the reporting period.
- **Formula:** `COUNT(DISTINCT order_id)` where order status is recognized.
- **Business meaning:** Volume of completed business.
- **Inclusion/exclusion rules:**
  - Excludes `PLACED`-only and `CANCELLED` orders.
  - Each order counted once regardless of the number of line items.

### 6. Units Sold

- **Definition:** Number of product units sold net of returned units.
- **Formula:** `Sold quantity − Returned quantity`.
- **Business meaning:** Physical volume moved through the platform.
- **Inclusion/exclusion rules:**
  - Based on recognized order items.
  - Returned units are subtracted to give net units sold.

### 7. Average Order Value (AOV)

- **Definition:** Average GMV per recognized order.
- **Formula:** `GMV / Orders`.
- **Business meaning:** Average basket size in monetary terms.
- **Inclusion/exclusion rules:**
  - Uses the same recognized-order basis as Orders.
  - Uses GMV (customer spend), which is the standard marketplace AOV definition.

### 8. Product Gross Margin (Seller View)

- **Definition:** Percentage of unit price retained by the seller after product cost.
- **Formula:** `(unit_price − unit_cost) / unit_price × 100`.
- **Business meaning:** Seller-facing product economics.
- **Inclusion/exclusion rules:**
  - This is a seller-view metric. Product COGS is the seller's cost, not Zavy's; it must not be used to compute Zavy profit.
  - Uses the unit price and unit cost captured on the order item at order time, so the margin stays reproducible after later product price or cost changes.
  - Undefined when unit price is zero.

## Customer KPIs

### 9. Customer Lifetime Value (CLV)

- **Definition:** Total GMV a customer has spent across all their recognized orders (customer lifetime spend).
- **Formula:** `Σ GMV over all recognized orders of the customer`.
- **Business meaning:** Identifies the most valuable customers by spend.
- **Inclusion/exclusion rules:**
  - Computed per customer; can be aggregated (e.g., average CLV = total GMV / number of customers with ≥1 recognized order).
  - May be reported net of returns; the gross/net basis must be stated with the result.
  - Zavy's commission is a fraction of this spend; CLV is not Zavy revenue.

### 10. Repeat Purchase Rate

- **Definition:** Share of purchasing customers who have placed more than one recognized order.
- **Formula:** `(customers with ≥2 recognized orders / customers with ≥1 recognized order) × 100`.
- **Business meaning:** Loyalty and repeat business.
- **Inclusion/exclusion rules:**
  - Denominator must be > 0; otherwise undefined.
  - Only recognized orders count (cancelled orders never count).

### 11. Customer Retention

- **Definition:** Share of customers active in a base period who place at least one recognized order in a later period.
- **Formula:** `(customers active in later period / customers active in base period) × 100`.
- **Business meaning:** How well Zavy keeps its customers.
- **Inclusion/exclusion rules:**
  - "Active" means ≥1 recognized order in the period.
  - Base and later periods must be stated (default: month-over-month cohorts).

## Returns KPIs

### 12. Product Return Rate

- **Definition:** Percentage of a product's sold units that were returned and received.
- **Formula:** `(Returned quantity / Sold quantity) × 100`.
- **Business meaning:** Product quality and fit issues.
- **Inclusion/exclusion rules:**
  - Uses gross Sold quantity as denominator.
  - Includes only approved and received returns.
  - Undefined when Sold quantity is zero.

## Seller KPIs

### 13. Seller Sales (GMV per Seller)

- **Definition:** GMV attributed to a seller.
- **Formula:** `Σ GMV over recognized order items whose product is listed by that seller`.
- **Business meaning:** Each seller's sales volume on the platform.
- **Inclusion/exclusion rules:**
  - Attribution is per order item (each item belongs to the product's listing seller).
  - Returns of a seller's items reduce the seller's net sales.
  - Zavy's commission for a seller = Seller Sales × commission rate (net of returned commissions).

### 14. Seller Return Rate

- **Definition:** Percentage of a seller's sold units that were returned and received.
- **Formula:** `(Returned quantity / Sold quantity for the seller's items) × 100`.
- **Business meaning:** Seller product quality and fulfilment performance.
- **Inclusion/exclusion rules:**
  - Same conventions as Product Return Rate, aggregated per seller.

## Inventory KPIs

### 15. Inventory Turnover

- **Definition:** How many times inventory is sold and replaced during a period.
- **Formula:** `Units sold over the period / Average units on hand over the period`, where average units on hand is the mean of daily `quantity on hand` (per product per warehouse) across the period, sourced from historical inventory snapshots maintained in the warehouse.
- **Business meaning:** Efficiency of stock usage.
- **Inclusion/exclusion rules:**
  - Period must be stated (default: year).
  - Uses warehouse-side historical inventory snapshots; the OLTP inventory table holds only current on-hand and is not sufficient by itself.
  - Undefined when average units on hand is zero.

### 16. Out-of-Stock Rate

- **Definition:** Percentage of products with zero available stock across all warehouses at the end of the reporting period.
- **Formula:** `(products with Σ quantity on hand across all warehouses = 0 / total products in scope) × 100`.
- **Business meaning:** Product availability health.
- **Inclusion/exclusion rules:**
  - Availability is aggregated across all warehouses: a product with stock in any warehouse is available, so a product that is out of stock in one warehouse but stocked in another is not counted as out of stock.
  - Snapshot metric computed at the end of the period.
  - Scope (all products vs active products) must be stated with the result.