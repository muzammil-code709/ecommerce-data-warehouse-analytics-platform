# Integrity Rules

This document maps the Stage 1 business rules into concrete database-level constraints and identifies rules that must be enforced by the application or pipeline. It includes treatment of order totals, inventory, shipment allocation, returns, reviews, addresses, and order-item snapshots.

## Database-enforceable rules (PK, FK, UNIQUE, NOT NULL, CHECK)

The following constraints should be enforced directly by the database where feasible.

Primary keys (PK)
- Each table has a surrogate primary key column (e.g., `customer_id`, `order_id`, `order_item_id`).

Foreign keys (FK)
- Enforce referential integrity for core relationships:
  - `orders.customer_id -> customers.customer_id`
  - `customer_addresses.customer_id -> customers.customer_id`
  - `products.seller_id -> sellers.seller_id`
  - `products.category_id -> categories.category_id`
  - `products.brand_id -> brands.brand_id`
  - `inventory.product_id -> products.product_id`
  - `inventory.warehouse_id -> warehouses.warehouse_id`
  - `stock_movements.product_id -> products.product_id`
  - `stock_movements.warehouse_id -> warehouses.warehouse_id`
  - `order_items.order_id -> orders.order_id`
  - `order_items.product_id -> products.product_id`
  - `payments.order_id -> orders.order_id`
  - `shipments.order_id -> orders.order_id`
  - `shipments.warehouse_id -> warehouses.warehouse_id`
  - `shipment_items.shipment_id -> shipments.shipment_id`
  - `shipment_items.order_item_id -> order_items.order_item_id`
  - `return_items.return_id -> returns.return_id`
  - `return_items.order_item_id -> order_items.order_item_id`
  - `reviews.customer_id -> customers.customer_id`
  - `reviews.product_id -> products.product_id`
  - `reviews.order_item_id -> order_items.order_item_id`

Unique constraints
- `customers.email` UNIQUE
- `sellers.email` UNIQUE
- `products.sku` UNIQUE
- `orders.order_number` UNIQUE
- `inventory` UNIQUE(product_id, warehouse_id)
- `reviews` UNIQUE(customer_id, order_item_id)

NOT NULL constraints
- Key operational columns should be NOT NULL: surrogate PKs, timestamps (`created_at`), `orders.status`, required identifiers for relationships.

CHECK constraints (examples)
- `products.list_price > 0`
- `products.cost >= 0`
- `sellers.commission_rate BETWEEN 0 AND 0.30`
- `order_items.quantity >= 1`
- `order_items.unit_price > 0`
- `order_items.unit_cost >= 0`
- `order_items.commission_rate BETWEEN 0 AND 0.30`
- `reviews.rating BETWEEN 1 AND 5`
- `inventory.quantity_on_hand >= 0`
- `stock_movements.movement_type IN ('RECEIPT','SALE','ADJUSTMENT','RETURN_IN')`
- `payments.amount >= 0`

These CHECKs prevent simple domain-level errors; they are not a substitute for procedural validations.

## Signed quantity convention for stock movements

To keep inventory updates consistent, the pipeline uses a signed-quantity convention for `stock_movements.quantity`:

- **RECEIPT** → positive (adds stock)
- **SALE** → negative (subtracts stock)
- **RETURN_IN** → positive (adds stock back into inventory)
- **ADJUSTMENT** → positive or negative (manual corrections)

The inventory application computes:
```
inventory.quantity_on_hand = inventory.quantity_on_hand + stock_movement.quantity
```
Application logic must ensure that applying a movement does not make `quantity_on_hand` negative. While the database can enforce `inventory.quantity_on_hand >= 0` via CHECK for current-state protection, preventing transient negative values under concurrency requires transactional locking (e.g., SELECT FOR UPDATE) or optimistic concurrency patterns at the application level.

## Order totals and financial integrity

- Business rule: `subtotal = SUM(order_item.line_total)` and `total = subtotal + shipping + tax`.
- Enforcement approach: Validate these equations in the application at order submission and in the ETL pipeline during load. Optionally, a defensive database trigger can assert the equality on order insert/update and reject invalid rows.

## Shipment allocation and partial fulfilment

- Business rule: The sum of `shipment_items.quantity_shipped` across all shipments for a given `order_item` must not exceed the `order_item.quantity`.
- Enforcement approach: Validate in the application when creating shipment items (recommended). For defense-in-depth, implement a database trigger that recalculates the total shipped quantity for the order_item and rejects inserts/updates that would exceed the ordered quantity.

## Returns and `returns.customer_id` consistency

- Design note: The schema currently includes `returns.customer_id` for operational convenience (quick lookups and auditing). However, this value is redundant because the owning `order_id` identifies the customer via `orders.customer_id`.
- Enforcement strategy if `returns.customer_id` is retained:
  - **Primary (application)**: When creating a return, the application verifies that `returns.customer_id` equals the `orders.customer_id` for the referenced `order_id`.
  - **Secondary (DB trigger)**: Add a defensive trigger to reject any `returns` row where `returns.customer_id` does not match `orders.customer_id` for the same `order_id`.
- Alternative: Drop `returns.customer_id` and derive the customer via a join to `orders` when needed. That avoids redundancy but increases the need for a join in common return-report queries.

## Return quantity validation

- Business rule: `1 <= return_items.quantity <= order_items.quantity`.
- Enforcement approach: Implement `return_items.quantity >= 1` as a DB CHECK; enforce the upper bound in the application/pipeline by retrieving `order_items.quantity` and validating the requested return quantity does not exceed the purchased amount. A defensive trigger can enforce the upper bound as well.

## Review rules

- Enforce `reviews.rating BETWEEN 1 AND 5` with a CHECK.
- Enforce `UNIQUE(customer_id, order_item_id)` to prevent duplicate reviews for the same order item by the same customer.
- Validate that the `order_item` referenced was delivered before accepting a review (application-level check).

## Address validation for orders

- Business rule: An order must not reference another customer's address for billing or shipping.
- Enforcement approach:
  - **Primary**: The application must validate on order creation that `billing_address_id` and `shipping_address_id` both belong to the same `customer_id` as the order.
  - **Defensive DB trigger**: Implement a trigger that verifies `customer_addresses.customer_id = orders.customer_id` for the referenced addresses and rejects mismatches.

## Order-item snapshots

- Business requirement: `order_items` store `unit_price`, `unit_cost`, `commission_rate` as snapshots captured at order time.
- Enforcement: The application must copy these values from the product and seller records at order creation. Database CHECKs ensure values are in expected ranges; the act of snapshotting is performed by the application.

## Concurrency and negative inventory

- While the DB can enforce `inventory.quantity_on_hand >= 0`, concurrent sales require transactional patterns to avoid race conditions:
  - Use row-level locks (`SELECT ... FOR UPDATE`) when decrementing inventory.
  - Implement retries and conflict detection in the application if lock contention occurs.

## Summary: Database vs Application responsibilities

### Database-enforceable
- PKs, FKs, UNIQUE, NOT NULL, and CHECK constraints for single-row and domain rules.

### Application / Pipeline validation
- Multi-row, cross-table constraints (order totals, shipment allocation, return upper bounds, order-address ownership, and workflow state transitions).

Implement application-level validation as the primary enforcement, with DB triggers used as defensive checks where warranted.
