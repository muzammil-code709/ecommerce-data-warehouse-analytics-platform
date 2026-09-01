# Relational Schema Specification (Logical)

This document converts the logical OLTP model into a relational schema description. For each table we list purpose, primary key (PK), foreign keys (FK), candidate/unique keys, important attributes, nullability, and important constraints.

Naming convention: `snake_case` plural table names (e.g., `customers`, `orders`). Primary keys use `{table}_id`.

---

## customers
- **Purpose**: Operational customer records.
- **PK**: `customer_id`
- **FK**: none
- **UNIQUE**: `email`
- **Important attributes**: `email` (NOT NULL), `name` (NOT NULL), `phone` (nullable), `date_of_birth` (nullable), `registration_date` (NOT NULL), `created_at` (NOT NULL), `updated_at` (NOT NULL)
- **CHECKs**: `email` format is validated at application layer; `registration_date` <= now.

---

## customer_addresses
- **Purpose**: Customer addresses; one default per customer.
- **PK**: `address_id`
- **FK**: `customer_id` → `customers(customer_id)` (NOT NULL)
- **UNIQUE**: none enforced across all columns, but `is_default` must be unique per customer via application or partial index.
- **Important attributes**: `line1` (NOT NULL), `line2` (nullable), `city` (NOT NULL), `state` (nullable), `postal_code` (nullable), `country` (NOT NULL), `is_default` (NOT NULL, boolean), `created_at`, `updated_at`.

---

## sellers
- **Purpose**: Seller accounts and commission information.
- **PK**: `seller_id`
- **FK**: none
- **UNIQUE**: `email`
- **Important attributes**: `name` (NOT NULL), `email` (NOT NULL), `location` (nullable), `registration_date` (NOT NULL), `commission_rate` (NOT NULL), `status` (NOT NULL), `created_at`, `updated_at`
- **CHECKs**: `commission_rate` BETWEEN 0 AND 0.30

---

## categories
- **Purpose**: Product categories (optional hierarchy).
- **PK**: `category_id`
- **FK**: `parent_category_id` → `categories(category_id)` (nullable)
- **UNIQUE**: `name` (recommended)
- **Important attributes**: `name` (NOT NULL), `status` (NOT NULL), `created_at`, `updated_at`

---

## brands
- **Purpose**: Product brands.
- **PK**: `brand_id`
- **UNIQUE**: `name`
- **Important attributes**: `name` (NOT NULL), `created_at`, `updated_at`

---

## products
- **Purpose**: Catalogue products listed by sellers.
- **PK**: `product_id`
- **FK**: `seller_id` → `sellers(seller_id)` (NOT NULL), `category_id` → `categories(category_id)` (NOT NULL), `brand_id` → `brands(brand_id)` (NOT NULL)
- **UNIQUE**: `sku`
- **Important attributes**: `sku` (NOT NULL), `name` (NOT NULL), `description` (nullable), `list_price` (NOT NULL), `cost` (NOT NULL), `status` (NOT NULL), `created_at`, `updated_at`
- **CHECKs**: `list_price` > 0; `cost` >= 0; `cost` <= `list_price` (business rule - optional enforcement)

---

## warehouses
- **Purpose**: Physical storage locations.
- **PK**: `warehouse_id`
- **Important attributes**: `name` (NOT NULL), `city`, `country`, `created_at`, `updated_at`

---

## inventory
- **Purpose**: Current on-hand quantity per product per warehouse.
- **PK**: `inventory_id`
- **FK**: `product_id` → `products(product_id)` (NOT NULL), `warehouse_id` → `warehouses(warehouse_id)` (NOT NULL)
- **UNIQUE**: (`product_id`, `warehouse_id`)
- **Important attributes**: `quantity_on_hand` (NOT NULL), `last_updated`
- **CHECKs**: `quantity_on_hand` >= 0

---

## stock_movements
- **Purpose**: Immutable ledger of inventory changes.
- **PK**: `stock_movement_id`
- **FK**: `product_id` → `products(product_id)` (NOT NULL), `warehouse_id` → `warehouses(warehouse_id)` (NOT NULL), `reference_id` (nullable link to order_item or return_item)
- **Important attributes**: `movement_type` (NOT NULL), `quantity` (NOT NULL, signed), `created_at` (NOT NULL), `note` (nullable)
- **CHECKs**: `movement_type` IN ('RECEIPT','SALE','ADJUSTMENT','RETURN_IN')

---

## orders
- **Purpose**: Order header records.
- **PK**: `order_id`
- **FK**: `customer_id` → `customers(customer_id)` (NOT NULL), `billing_address_id` → `customer_addresses(address_id)` (nullable), `shipping_address_id` → `customer_addresses(address_id)` (nullable)
- **UNIQUE**: `order_number`
- **Important attributes**: `order_number` (NOT NULL), `order_date` (NOT NULL), `status` (NOT NULL), `subtotal` (NOT NULL), `shipping` (NOT NULL), `tax` (NOT NULL), `total` (NOT NULL), `created_at`, `updated_at`
- **CHECKs**: `subtotal` >= 0, `shipping` >= 0, `tax` >= 0, `total` = `subtotal + shipping + tax` (application-level verification recommended)

---

## order_items
- **Purpose**: Line items for orders and order-time pricing snapshots.
- **PK**: `order_item_id`
- **FK**: `order_id` → `orders(order_id)` (NOT NULL), `product_id` → `products(product_id)` (NOT NULL)
- **Important attributes**: `quantity` (NOT NULL), `unit_price` (NOT NULL), `unit_cost` (NOT NULL), `commission_rate` (NOT NULL), `line_total` (NOT NULL), `created_at`
- **CHECKs**: `quantity` >= 1, `unit_price` > 0, `unit_cost` >= 0, `commission_rate` BETWEEN 0 AND 0.30

---

## payments
- **Purpose**: Payment attempts for orders.
- **PK**: `payment_id`
- **FK**: `order_id` → `orders(order_id)` (NOT NULL)
- **Important attributes**: `amount` (NOT NULL), `payment_method` (NOT NULL), `status` (NOT NULL), `payment_date` (nullable), `created_at`
- **CHECKs**: `amount` >= 0

---

## shipments
- **Purpose**: Physical dispatch records per order.
- **PK**: `shipment_id`
- **FK**: `order_id` → `orders(order_id)` (NOT NULL), `warehouse_id` → `warehouses(warehouse_id)` (NOT NULL)
- **Important attributes**: `carrier` (nullable), `tracking_number` (nullable), `ship_date` (nullable), `delivery_date` (nullable), `status` (NOT NULL), `created_at`

---

## shipment_items
- **Purpose**: Allocation of order_item quantities to shipments.
- **PK**: `shipment_item_id`
- **FK**: `shipment_id` → `shipments(shipment_id)` (NOT NULL), `order_item_id` → `order_items(order_item_id)` (NOT NULL)
- **Important attributes**: `quantity_shipped` (NOT NULL), `created_at`
- **CHECKs**: `quantity_shipped` >= 1

---

## returns
- **Purpose**: Returns at the order level.
- **PK**: `return_id`
- **FK**: `order_id` → `orders(order_id)` (NOT NULL), `customer_id` → `customers(customer_id)` (NOT NULL)
- **Important attributes**: `status` (NOT NULL), `request_date`, `received_date` (nullable), `refund_amount` (nullable), `created_at`

---

## return_items
- **Purpose**: Individual return item lines and reason codes.
- **PK**: `return_item_id`
- **FK**: `return_id` → `returns(return_id)` (NOT NULL), `order_item_id` → `order_items(order_item_id)` (NOT NULL), `product_id` → `products(product_id)` (NOT NULL)
- **Important attributes**: `quantity` (NOT NULL), `reason` (NOT NULL), `created_at`
- **CHECKs**: `quantity` >= 1

---

## reviews
- **Purpose**: Customer reviews tied to delivered order items.
- **PK**: `review_id`
- **FK**: `customer_id` → `customers(customer_id)` (NOT NULL), `product_id` → `products(product_id)` (NOT NULL), `order_item_id` → `order_items(order_item_id)` (NOT NULL)
- **UNIQUE**: (`customer_id`, `order_item_id`) — at most one review per order item by the purchasing customer.
- **Important attributes**: `rating` (NOT NULL), `comment` (nullable), `created_at`, `updated_at`
- **CHECKs**: `rating` BETWEEN 1 AND 5
