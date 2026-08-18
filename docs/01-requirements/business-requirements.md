# Business Requirements

The requirements below are grouped by domain. Each one has a stable identifier (BR-001, BR-002, …) and is stated so it can be verified against the implementation.

**Terminology note:** sales value (Gross Merchandise Value / GMV, also Seller Sales) is recognized for orders in status `CONFIRMED`, `PROCESSING`, `SHIPPED`, or `DELIVERED` (cancelled and `PLACED`-only orders are excluded). Zavy is a marketplace: the product selling price is GMV/Seller Sales and belongs to the seller; Zavy's own revenue is the commission earned on those sales (per-seller commission rate). See `kpi-definitions.md` and `business-rules.md`.

## Customer Management

- **BR-001** — The system must store a unique customer record with contact information, registration date, and a derived customer segment.
- **BR-002** — A customer must be able to maintain one or more addresses, with one designated default address.
- **BR-003** — The system must identify each customer by a unique identifier so that all orders, payments, returns, and reviews can be linked to the correct customer.

## Product Management

- **BR-004** — The system must store products with a unique SKU, name, category, brand, list price, cost, and active/inactive status.
- **BR-005** — Each product must belong to exactly one category and one brand.
- **BR-006** — Only active products may be listed for sale by sellers.

## Seller Management

- **BR-007** — The system must store seller records with contact and location information, registration date, status, and commission rate.
- **BR-008** — A seller must be able to list products, and each listed product must be linked to exactly one seller.
- **BR-009** — Only active sellers may list products and receive orders.

## Order Management

- **BR-010** — A customer must be able to place an order containing one or more product line items.
- **BR-011** — The system must track the full order lifecycle: `PLACED → CONFIRMED → PROCESSING → SHIPPED → DELIVERED`, with a `CANCELLED` path available before shipment.
- **BR-012** — Each order line item must record the unit price, unit cost, and seller commission rate in effect at the time of order so that later changes to any of them do not affect historical orders, gross margin, or commission calculations.
- **BR-013** — The system must be able to answer, for every order: which customer placed it, when, which items it contains, and the order totals.

## Payment Processing

- **BR-014** — Every order must generate at least one payment record.
- **BR-015** — The system must track payment statuses (`PENDING`, `PAID`, `FAILED`, `REFUNDED`, `PARTIALLY_REFUNDED`) and payment method.
- **BR-016** — An order must not proceed to fulfilment until it is paid, except for cash-on-delivery orders which are paid at delivery.

## Shipment Tracking

- **BR-017** — The system must record shipment information for order items, including carrier, tracking number, shipment and delivery dates, and shipment status.
- **BR-018** — An order must be considered delivered only when all of its line items have been delivered.

## Inventory Management

- **BR-019** — The system must track stock quantity per product per warehouse and must prevent stock from going negative.
- **BR-020** — Every stock change must be recorded as a stock movement with a type (`RECEIPT`, `SALE`, `ADJUSTMENT`, `RETURN_IN`), timestamp, and quantity.
- **BR-021** — The system must be able to report current stock levels, low-stock products, and stock turnover.

## Returns

- **BR-022** — A customer must be able to request a return for delivered items within the return window (30 days after delivery).
- **BR-023** — The system must track return requests through statuses `REQUESTED → APPROVED/REJECTED → RETURNED → REFUNDED`.
- **BR-024** — Returned quantity for an item must never exceed the quantity the customer purchased.
- **BR-025** — Approved and received returns must result in a refund recorded against the original payment.

## Reviews

- **BR-026** — Only customers who purchased a delivered product may review it.
- **BR-027** — Each review must have an integer rating from 1 to 5 and an optional comment.

## Analytics and Reporting

- **BR-028** — The platform must produce analytical answers for the key business questions defined in `business-questions.md`.
- **BR-029** — The platform must calculate the KPIs defined in `kpi-definitions.md` consistently.
- **BR-030** — The platform must provide reporting via Power BI dashboards covering executive, sales, customer, product/inventory, and seller views.
- **BR-031** — Historical attribute changes (e.g., a customer moving to a new city) must be preserved in the data warehouse via SCD Type 2 so that analyses reflect the state at the time of each order. The OLTP database maintains only current operational state.
- **BR-032** — The data pipeline must process new and changed operational data incrementally rather than reloading the entire dataset on every run.
- **BR-033** — The platform must distinguish Gross Merchandise Value (total value of seller sales on the platform) from Zavy's commission revenue (commission applied to seller sales).
- **BR-034** — Seller commission rates must be preserved so that Zavy commission revenue can be calculated per seller and over time.
- **BR-035** — The system must record which order items are allocated to which shipment (shipment items) so that split fulfilment is fully represented.
- **BR-036** — Order financial totals must follow: `subtotal` = sum of line-item totals; `total` = `subtotal + shipping + tax`. No discounts are modelled.