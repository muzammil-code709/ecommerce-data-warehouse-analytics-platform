# OLTP Data Model — Zavy

This document describes the logical OLTP data model for the operational system. The OLTP database represents the current operational state only; historical tracking (SCD Type 2) is implemented in the warehouse. All entities follow Stage 1 requirements.

The model covers these domains: Customers, Addresses, Products, Categories, Brands, Sellers, Orders, Order Items, Payments, Shipments, Shipment Items, Warehouses, Inventory, Stock Movements, Returns, Return Items, and Reviews.

For each entity below we describe purpose, primary identifier, important attributes, relationships, cardinality, and business constraints.

---

## Customers

Purpose
: Holds the operational record for a marketplace customer.

Primary identifier
: `customer_id`

Important attributes
: `email` (unique), `name`, `phone`, `date_of_birth` (optional), `registration_date`, `created_at`, `updated_at`.

Relationships & Cardinality
: 1 customer —< 0..* addresses (one default address required)
: 1 customer —< 0..* orders
: 1 customer —< 0..* payments (through orders)
: 1 customer —< 0..* returns
: 1 customer —< 0..* reviews

Business constraints
: `email` is unique and NOT NULL; `registration_date` NOT NULL; customers may have multiple addresses but exactly one default address (enforced by a unique constraint on address default flag scoped to customer, or by application rules—see integrity rules).

---

## Addresses

Purpose
: Stores shipping and billing addresses for customers.

Primary identifier
: `address_id`

Important attributes
: `customer_id` (FK), `line1`, `line2`, `city`, `state`, `postal_code`, `country`, `is_default` (boolean), `created_at`, `updated_at`.

Relationships & Cardinality
: 1 customer —< 1..* addresses.

Business constraints
: Each address belongs to one customer. Exactly one address per customer must be marked as default; enforceable via a unique partial index or managed in application logic.

---

## Sellers

Purpose
: Stores seller accounts and commercial terms.

Primary identifier
: `seller_id`

Important attributes
: `name`, `email` (unique), `location`, `registration_date`, `commission_rate` (snapshot on order items), `status` (`ACTIVE`/`SUSPENDED`), `created_at`, `updated_at`.

Relationships & Cardinality
: 1 seller —< 0..* products

Business constraints
: `commission_rate` is a percentage between 0% and 30% (CHECK). `email` unique and NOT NULL.

---

## Categories

Purpose
: Product classification (single category per product).

Primary identifier
: `category_id`

Important attributes
: `name`, `parent_category_id` (optional), `status`, `created_at`, `updated_at`.

Relationships & Cardinality
: 1 category —< 0..* products; optional parent-child relationship for hierarchical categories.

Business constraints
: `name` unique within scope; parent-child relationships must not form cycles (application-level validation).

---

## Brands

Purpose
: Brand registry for products.

Primary identifier
: `brand_id`

Important attributes
: `name`, `created_at`, `updated_at`.

Relationships & Cardinality
: 1 brand —< 0..* products.

Business constraints
: `name` should be unique; enforced by a unique constraint.

---

## Products

Purpose
: The platform's catalogue: products listed by sellers.

Primary identifier
: `product_id`

Important attributes
: `sku` (unique), `seller_id` (FK), `category_id` (FK), `brand_id` (FK), `name`, `description`, `list_price`, `cost`, `status` (`ACTIVE`/`INACTIVE`), `created_at`, `updated_at`.

Relationships & Cardinality
: 1 product — belongs to exactly 1 seller
: 1 product — belongs to exactly 1 category and 1 brand
: 1 product —< 0..* inventory (per warehouse)
: 1 product —< 0..* stock_movements

Business constraints
: `sku` unique and NOT NULL. `list_price` > 0; `cost` >= 0. Product `status` governs whether it can appear in new orders.

---

## Warehouses

Purpose
: Physical storage locations for inventory.

Primary identifier
: `warehouse_id`

Important attributes
: `name`, `city`, `country`, `created_at`, `updated_at`.

Relationships & Cardinality
: 1 warehouse —< 0..* inventory rows.

Business constraints
: `name` unique per deployment; warehouses are referenced by shipments and stock movements.

---

## Inventory

Purpose
: Tracks current on-hand quantity of a product in a specific warehouse (operational current state).

Primary identifier
: `inventory_id`

Important attributes
: `product_id` (FK), `warehouse_id` (FK), `quantity_on_hand`, `last_updated`.

Relationships & Cardinality
: Product × Warehouse is the unique key; one inventory row per product+warehouse.

Business constraints
: `quantity_on_hand` must be >= 0 (CHECK). Updates to inventory are recorded as stock movements; inventory is the current view used by operational fulfillment.

---

## Stock Movements

Purpose
: Immutable ledger of inventory changes.

Primary identifier
: `stock_movement_id`

Important attributes
: `product_id` (FK), `warehouse_id` (FK), `movement_type` (`RECEIPT`,`SALE`,`ADJUSTMENT`,`RETURN_IN`), `quantity` (signed integer), `reference_id` (nullable link to order item or return item), `created_at`.

Relationships & Cardinality
: Each stock movement references a product and a warehouse. Multiple stock movements accumulate to change inventory.on_hand.

Business constraints
: `movement_type` limited to the enumerated set. Quantity sign convention defined by movement type; application-level logic ensures inventory remains non-negative after applying movements.

---

## Orders

Purpose
: Operational order header capturing billing/shipping, status, and totals.

Primary identifier
: `order_id`

Important attributes
: `order_number` (business-unique), `customer_id` (FK), `billing_address_id` (FK), `shipping_address_id` (FK), `order_date`, `status` (`PLACED`,`CONFIRMED`,`PROCESSING`,`SHIPPED`,`DELIVERED`,`CANCELLED`), `subtotal`, `shipping`, `tax`, `total`, `created_at`, `updated_at`.

Relationships & Cardinality
: 1 customer —< 1..* orders
: 1 order —< 1..* order_items
: 1 order —< 0..* payments
: 1 order —< 0..* shipments

Business constraints
: `order_number` unique; `subtotal = SUM(order_item.line_total)`; `total = subtotal + shipping + tax` (validated by application/pipeline). An order must contain at least one order_item at creation.

---

## Order Items

Purpose
: Line-level order details and order-time pricing snapshots.

Primary identifier
: `order_item_id`

Important attributes
: `order_id` (FK), `product_id` (FK), `quantity`, `unit_price` (snapshot at order time), `unit_cost` (snapshot at order time), `commission_rate` (snapshot at order time), `line_total`, `created_at`.

Relationships & Cardinality
: 1 order —< 1..* order_items; 1 order_item — belongs to exactly one product.

Business constraints
: `quantity` >= 1; `unit_price` > 0; `unit_cost` >= 0; `commission_rate` between 0% and 30% (CHECK). `line_total = quantity × unit_price` (enforced in application/pipeline; optionally checked at insert).

---

## Payments

Purpose
: Payment attempts and their statuses for orders.

Primary identifier
: `payment_id`

Important attributes
: `order_id` (FK), `amount`, `payment_method` (`CARD`,`WALLET`,`BANK_TRANSFER`,`COD`), `status` (`PENDING`,`PAID`,`FAILED`,`REFUNDED`,`PARTIALLY_REFUNDED`), `payment_date`, `created_at`.

Relationships & Cardinality
: 1 order —< 1..* payments (multiple attempts possible).

Business constraints
: `amount` >= 0; refunds are linked back to original payment records when applicable.

---

## Shipments

Purpose
: Represents a physical dispatch from a warehouse for one order.

Primary identifier
: `shipment_id`

Important attributes
: `order_id` (FK), `warehouse_id` (FK), `carrier`, `tracking_number`, `ship_date`, `delivery_date`, `status` (`PENDING_PICKUP`,`IN_TRANSIT`,`DELIVERED`,`FAILED_DELIVERY`,`RETURNED_TO_SENDER`), `created_at`.

Relationships & Cardinality
: 1 order —< 0..* shipments; 1 shipment — contains 1..* shipment_items.

Business constraints
: Each shipment references exactly one order and one warehouse.

---

## Shipment Items

Purpose
: Allocation of order item quantities to a shipment (supports split/partial fulfilment).

Primary identifier
: `shipment_item_id`

Important attributes
: `shipment_id` (FK), `order_item_id` (FK), `quantity_shipped`, `created_at`.

Relationships & Cardinality
: 1 shipment —< 1..* shipment_items; 1 shipment_item references exactly one order_item.

Business constraints
: `quantity_shipped` >= 1. The sum of `quantity_shipped` across shipment_items for an order_item must not exceed the order_item.quantity (application/pipeline validation). Shipment-item quantity must not exceed remaining unfulfilled quantity for the order_item.

---

## Returns and Return Items

Purpose
: Captures customer return requests and the specific returned items.

Primary identifier
: `return_id`, `return_item_id`

Important attributes
: `return`: `order_id` (FK), `customer_id` (FK), `status` (`REQUESTED`,`APPROVED`,`REJECTED`,`RETURNED`,`REFUNDED`), `request_date`, `received_date`, `refund_amount`, `created_at`.
  `return_item`: `return_id` (FK), `order_item_id` (FK), `product_id` (FK), `quantity`, `reason`, `created_at`.

Relationships & Cardinality
: 1 order —< 0..* returns; 1 return —< 1..* return_items; return_item references one order_item.

Business constraints
: Returned quantity for an order_item must be between 1 and the purchased quantity (application-level validation). Refunds link to payment records.

---

## Reviews

Purpose
: Customer product feedback tied to a delivered order item.

Primary identifier
: `review_id`

Important attributes
: `customer_id` (FK), `product_id` (FK), `order_item_id` (FK), `rating` (1..5), `comment`, `created_at`, `updated_at`.

Relationships & Cardinality
: 1 customer —< 0..* reviews; 1 product —< 0..* reviews; 1 order_item — 0..1 review (enforced by unique constraint on order_item_id and customer_id).

Business constraints
: A review must be associated with a delivered purchase. Rating must be an integer between 1 and 5 (CHECK). A customer may submit at most one review per order item (UNIQUE(customer_id, order_item_id)).

---

## Notes on Modeling Scope

- The OLTP model stores the current operational snapshot only. Historical dimension versions (SCD Type 2) are implemented in the data warehouse.
- Order-item level snapshots (`unit_price`, `unit_cost`, `commission_rate`) are stored on `order_items` to guarantee reproducible historical calculations for KPIs.
- Operational inventory enforces a non-negative on-hand quantity as the current view; all mutations are recorded in `stock_movements`.
