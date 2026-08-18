# Business Rules

Business rules applied during data generation, validation, and pipeline processing. Status values use the same enum names throughout the project.

## Order Lifecycle

Orders follow a fixed lifecycle:

```
PLACED → CONFIRMED → PROCESSING → SHIPPED → DELIVERED
```

Cancellation and return paths:

```
PLACED ──► CANCELLED          (customer cancels immediately)
CONFIRMED ──► CANCELLED       (before shipment)
PROCESSING ──► CANCELLED      (before shipment)
SHIPPED → DELIVERED ──► RETURN path (after delivery)
```

- **BRU-001** — An order always starts in `PLACED` when the customer submits it.
- **BRU-002** — An order moves `PLACED → CONFIRMED` once payment is accepted (or confirmed as cash-on-delivery).
- **BRU-003** — An order may be `CANCELLED` from `PLACED`, `CONFIRMED`, or `PROCESSING`. It must never be cancelled after `SHIPPED`.
- **BRU-004** — An order becomes `DELIVERED` only when all of its line items are delivered.

## Customer Rules

- **BRU-005** — Each customer must have a unique identifier and a unique email address.
- **BRU-006** — A customer's email must be a valid email format and must not be empty.
- **BRU-007** — A customer must have at least one address and exactly one designated default address.
- **BRU-008** — If a customer's date of birth is recorded, it must be a date in the past.
- **BRU-009** — A customer's segment (`NEW`, `REGULAR`, `VIP`) is derived from order history, not manually entered.
- **BRU-010** — A customer may place many orders, but each order belongs to exactly one customer.

## Product Rules

- **BRU-011** — Each product must have a unique SKU.
- **BRU-012** — A product belongs to exactly one category and exactly one brand.
- **BRU-013** — A product's list price must be greater than zero, and its cost must be non-negative.
- **BRU-014** — A product's cost must be less than or equal to its list price (a product cannot be sold below cost as a permanent state).
- **BRU-015** — A product has status `ACTIVE` or `INACTIVE`. Only `ACTIVE` products may be included in new orders.
- **BRU-016** — Each product is listed by exactly one seller.

## Seller Rules

- **BRU-017** — Each seller must have a unique identifier and a unique registered email.
- **BRU-018** — A seller's commission rate must be between 0% and 30% (inclusive).
- **BRU-019** — A seller has status `ACTIVE` or `SUSPENDED`. Only `ACTIVE` sellers may list products or receive new orders.
- **BRU-020** — A seller may list many products, but each listed product is associated with exactly one seller.

## Order Rules

- **BRU-021** — An order must contain at least one order item.
- **BRU-022** — Order date and order status dates must be valid dates in the past (no future-dated orders).
- **BRU-023** — Order financial totals follow: `subtotal` = sum of line-item totals; `total` = `subtotal + shipping + tax`. Shipping and tax must be non-negative. No discounts are modelled.
- **BRU-024** — An order references exactly one customer and exactly one billing/shipping address set.
- **BRU-025** — Status transitions must be monotonic: `PLACED` is earliest, `DELIVERED` is latest (except `CANCELLED`).

## Order-Item Rules

- **BRU-026** — Each order item references exactly one order and exactly one product.
- **BRU-027** — Order item quantity must be a positive integer (≥ 1).
- **BRU-028** — Order item unit price, unit cost, and the seller's commission rate are all captured (snapshotted) at order time. Unit price must equal the product's list price in effect at that time, unit cost must equal the product's cost in effect at that time, and the commission rate must equal the listing seller's rate in effect at that time. Later changes to product price, product cost, or seller commission do not apply to the order item.
- **BRU-029** — Line total = `quantity × unit_price`.
- **BRU-030** — Only `ACTIVE` products may appear in new order items.

## Payment Rules

- **BRU-031** — Every order must generate at least one payment record.
- **BRU-032** — Payment status values: `PENDING`, `PAID`, `FAILED`, `REFUNDED`, `PARTIALLY_REFUNDED`.
- **BRU-033** — A payment amount must be non-negative.
- **BRU-034** — An order does not progress beyond `CONFIRMED` until a payment has `PAID` status, except cash-on-delivery (`COD`) orders, which are paid at delivery.
- **BRU-035** — `REFUNDED` or `PARTIALLY_REFUNDED` payments must be linked to an approved and received return.
- **BRU-036** — A failed payment may be retried; each attempt is recorded as a payment record.

## Shipment Rules

- **BRU-037** — A shipment is created when order items move to `SHIPPED`.
- **BRU-038** — A shipment belongs to exactly one order and contains one or more shipment items, each referencing an order item (see Shipment-Item Rules below).
- **BRU-039** — Shipment status values: `PENDING_PICKUP`, `IN_TRANSIT`, `DELIVERED`, `FAILED_DELIVERY`, `RETURNED_TO_SENDER`.
- **BRU-040** — An order reaches `DELIVERED` only after all its items are marked delivered in their shipments.

## Inventory Rules

- **BRU-041** — Stock quantity is tracked per product per warehouse.
- **BRU-042** — Stock must never go negative; any movement that would make stock negative is rejected.
- **BRU-043** — Every stock change is recorded as a stock movement with type `RECEIPT`, `SALE`, `ADJUSTMENT`, or `RETURN_IN`, plus timestamp and quantity.
- **BRU-044** — `RECEIPT` and `RETURN_IN` increase stock; `SALE` decreases stock; `ADJUSTMENT` may increase or decrease.
- **BRU-045** — Stock must be available (quantity ≥ requested) before an order item can be fulfilled.

## Return Rules

- **BRU-046** — A return may be requested only for items in a `DELIVERED` order.
- **BRU-047** — A return request must be made within the return window: 30 days after the order's delivery date.
- **BRU-048** — Return status values: `REQUESTED`, `APPROVED`, `REJECTED`, `RETURNED`, `REFUNDED`.
- **BRU-049** — Returned quantity for an item must be between 1 and the purchased quantity of that item (inclusive).
- **BRU-050** — Each return item must have a return reason from: `WRONG_ITEM`, `DAMAGED`, `NOT_AS_DESCRIBED`, `SIZE_OR_FIT`, `CHANGED_MIND`, `OTHER`.
- **BRU-051** — Only `APPROVED` returns progress to `RETURNED`; only received returns (`RETURNED`) progress to `REFUNDED`.
- **BRU-052** — `REFUNDED` returns trigger a refund recorded against the original payment.
- **BRU-053** — Received returned items are restocked via a `RETURN_IN` stock movement.

## Review Rules

- **BRU-054** — A review may be submitted only by a customer who purchased the product and whose order is `DELIVERED`.
- **BRU-055** — A customer may submit at most one review per order item.
- **BRU-056** — Review rating is an integer from 1 to 5.
- **BRU-057** — A review comment is optional; an empty comment is allowed.
- **BRU-058** — A customer may edit their review; the original creation timestamp is preserved and an update timestamp is recorded.

## Analytics Rules

- **BRU-059** — Sales value (GMV) is recognized only for orders in `CONFIRMED`, `PROCESSING`, `SHIPPED`, or `DELIVERED` status, per the conventions in `kpi-definitions.md`.
- **BRU-060** — Historical attribute changes (e.g., customer city change) must be preserved in the warehouse using SCD Type 2 so that facts reference the version in effect at event time.

## Shipment-Item Rules

- **BRU-061** — A shipment belongs to exactly one order; a shipment may contain multiple order items; an order may have multiple shipments (split fulfilment, e.g., from different warehouses).
- **BRU-062** — An order item may be split across multiple shipments (partial fulfilment is allowed). The quantity on each shipment item must not exceed the remaining unfulfilled quantity of the order item, and the total shipped quantity must never exceed the order-item quantity.

## Marketplace Economics Rules

- **BRU-063** — The product selling price is Seller Sales/GMV and belongs to the seller; Zavy's revenue is the commission on those sales, calculated with the commission rate snapshotted on each order item at order time.
- **BRU-064** — Commission on received returns is refunded and reduces Zavy commission revenue.

## System Responsibility Rules

- **BRU-065** — The OLTP database maintains current operational state only. Historical attribute tracking (SCD Type 2) is implemented in the data warehouse, not in the OLTP database.