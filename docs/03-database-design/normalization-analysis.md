# Normalization Analysis (1NF, 2NF, 3NF)

This document demonstrates that the OLTP design follows standard normalization principles while remaining practical for transactional workloads.

## 1NF — First Normal Form

- All tables have atomic columns (no repeating groups or arrays in single columns).
- Every row has a primary key. Examples: `customers.customer_id`, `orders.order_id`, `order_items.order_item_id`.
- Repeating data (e.g., multiple addresses for a customer) is represented in a separate `customer_addresses` table rather than as repeating columns.

Conclusion: The design adheres to 1NF.

## 2NF — Second Normal Form

- 2NF addresses partial dependencies on composite keys. Our design uses single-column primary keys for most tables, so partial dependencies are not applicable.
- For junction-like operational tables, we avoid storing attributes that depend on only part of a composite key. Example: `order_items` uses `order_item_id` as PK; attributes such as `unit_price`, `unit_cost`, and `commission_rate` belong on the order item because they describe that specific item in that order.

Conclusion: The design avoids partial dependencies and is consistent with 2NF.

## 3NF — Third Normal Form

- 3NF requires that non-key attributes are not transitively dependent on the primary key.
- Examples checked:
  - `products` stores `list_price` and `cost` which logically belong to the product and are not derived from other non-key attributes.
  - `orders` stores `subtotal`, `shipping`, `tax`, `total`. `subtotal` is derivable from `order_items`, but the `orders` table stores the transactional totals for quick checks and integrity assertions; application/pipeline validation ensures `total = subtotal + shipping + tax`.
  - `inventory` stores current `quantity_on_hand` per product/warehouse; historical values are not stored in the OLTP table to avoid denormalization.

Conclusion: The model avoids transitive dependencies for core entities and remains effectively in 3NF for transactional use.

## Practical normalization decisions

- **Order item snapshots**: `unit_price`, `unit_cost`, and `commission_rate` are stored on `order_items` (not `products`) to ensure historical reproducibility. This is not denormalization for write performance; it is a necessary operational snapshot.
- **Inventory**: Current on-hand is stored in `inventory`; stock movements are an immutable ledger that removes the need for storing historical snapshots in the OLTP system.

## Avoiding over-normalization

- The model intentionally avoids splitting trivially related fields into extra tables where that would add needless joins for operational transactions (e.g., simple address fields remain in `customer_addresses` rather than separate micro-tables).
