# Data Dictionary (OLTP)

The logical data types below are guidance for Stage 4. Use these types as logical categories rather than physical PostgreSQL types.

| Table | Column | Description | Logical Data Type | Nullable | Key/Constraint |
| ----- | ------ | ----------- | ----------------- | -------- | -------------- |
| customers | customer_id | Surrogate PK | Integer | NO | PK |
| customers | email | Customer email (unique) | String | NO | UNIQUE, NOT NULL |
| customers | name | Full name | String | NO | NOT NULL |
| customers | phone | Phone number | String | YES | |
| customers | date_of_birth | DOB | Date | YES | |
| customers | registration_date | Registration timestamp | Timestamp | NO | NOT NULL |
| customers | created_at | Created timestamp | Timestamp | NO | NOT NULL |
| customers | updated_at | Last update timestamp | Timestamp | NO | NOT NULL |

| customer_addresses | address_id | Surrogate PK | Integer | NO | PK |
| customer_addresses | customer_id | FK → customers | Integer | NO | FK |
| customer_addresses | line1 | Address line 1 | String | NO | NOT NULL |
| customer_addresses | line2 | Address line 2 | String | YES | |
| customer_addresses | city | City | String | NO | NOT NULL |
| customer_addresses | state | State/province | String | YES | |
| customer_addresses | postal_code | Postal code | String | YES | |
| customer_addresses | country | Country | String | NO | NOT NULL |
| customer_addresses | is_default | Default address flag | Boolean | NO | NOT NULL (unique per customer - application or partial index)

| sellers | seller_id | Surrogate PK | Integer | NO | PK |
| sellers | name | Seller name | String | NO | NOT NULL |
| sellers | email | Seller email | String | NO | UNIQUE, NOT NULL |
| sellers | location | Seller location | String | YES | |
| sellers | registration_date | Registration timestamp | Timestamp | NO | |
| sellers | commission_rate | Commission rate (decimal fraction) | Decimal | NO | CHECK 0..0.30 |
| sellers | status | Seller status | String | NO | NOT NULL |

| categories | category_id | PK | Integer | NO | PK |
| categories | name | Category name | String | NO | UNIQUE, NOT NULL |
| categories | parent_category_id | Optional parent | Integer | YES | FK -> categories(category_id)

| brands | brand_id | PK | Integer | NO | PK |
| brands | name | Brand name | String | NO | UNIQUE, NOT NULL |

| products | product_id | PK | Integer | NO | PK |
| products | sku | Stock keeping unit | String | NO | UNIQUE, NOT NULL |
| products | seller_id | FK → sellers | Integer | NO | FK |
| products | category_id | FK → categories | Integer | NO | FK |
| products | brand_id | FK → brands | Integer | NO | FK |
| products | name | Product name | String | NO | NOT NULL |
| products | description | Product description | String | YES | |
| products | list_price | List price | Decimal | NO | CHECK >0 |
| products | cost | Product cost | Decimal | NO | CHECK >=0 |
| products | status | ACTIVE / INACTIVE | String | NO | NOT NULL |

| warehouses | warehouse_id | PK | Integer | NO | PK |
| warehouses | name | Warehouse name | String | NO | NOT NULL |
| warehouses | city | City | String | YES | |
| warehouses | country | Country | String | YES | |

| inventory | inventory_id | PK | Integer | NO | PK |
| inventory | product_id | FK → products | Integer | NO | FK |
| inventory | warehouse_id | FK → warehouses | Integer | NO | FK |
| inventory | quantity_on_hand | Current on-hand | Integer | NO | CHECK >= 0; UNIQUE(product_id, warehouse_id)
| inventory | last_updated | Timestamp | Timestamp | NO | NOT NULL |

| stock_movements | stock_movement_id | PK | Integer | NO | PK |
| stock_movements | product_id | FK → products | Integer | NO | FK |
| stock_movements | warehouse_id | FK → warehouses | Integer | NO | FK |
| stock_movements | movement_type | Movement type | String | NO | CHECK in ('RECEIPT','SALE','ADJUSTMENT','RETURN_IN') |
| stock_movements | quantity | Signed quantity | Integer | NO | NOT NULL |
| stock_movements | reference_id | Optional external reference | Integer | YES | |
| stock_movements | created_at | Timestamp | Timestamp | NO | NOT NULL |

| orders | order_id | PK | Integer | NO | PK |
| orders | order_number | Business order number | String | NO | UNIQUE, NOT NULL |
| orders | customer_id | FK → customers | Integer | NO | FK |
| orders | billing_address_id | FK → customer_addresses | Integer | YES | FK |
| orders | shipping_address_id | FK → customer_addresses | Integer | YES | FK |
| orders | order_date | Timestamp | Timestamp | NO | NOT NULL |
| orders | status | Order status | String | NO | NOT NULL |
| orders | subtotal | Monetary subtotal | Decimal | NO | CHECK >=0 |
| orders | shipping | Shipping amount | Decimal | NO | CHECK >=0 |
| orders | tax | Tax amount | Decimal | NO | CHECK >=0 |
| orders | total | Order total | Decimal | NO | NOT NULL |

| order_items | order_item_id | PK | Integer | NO | PK |
| order_items | order_id | FK → orders | Integer | NO | FK |
| order_items | product_id | FK → products | Integer | NO | FK |
| order_items | quantity | Quantity purchased | Integer | NO | CHECK >= 1 |
| order_items | unit_price | Unit price (snapshot) | Decimal | NO | CHECK > 0 |
| order_items | unit_cost | Unit cost (snapshot) | Decimal | NO | CHECK >= 0 |
| order_items | commission_rate | Commission snapshot | Decimal | NO | CHECK 0..0.30 |
| order_items | line_total | Line total | Decimal | NO | NOT NULL |
| order_items | created_at | Timestamp | Timestamp | NO | NOT NULL |

| payments | payment_id | PK | Integer | NO | PK |
| payments | order_id | FK → orders | Integer | NO | FK |
| payments | amount | Payment amount | Decimal | NO | CHECK >= 0 |
| payments | payment_method | Payment method | String | NO | NOT NULL |
| payments | status | Payment status | String | NO | NOT NULL |
| payments | payment_date | Timestamp | YES | |

| shipments | shipment_id | PK | Integer | NO | PK |
| shipments | order_id | FK → orders | Integer | NO | FK |
| shipments | warehouse_id | FK → warehouses | Integer | NO | FK |
| shipments | carrier | Carrier name | String | YES | |
| shipments | tracking_number | Tracking ID | String | YES | |
| shipments | ship_date | Timestamp | YES | |
| shipments | delivery_date | Timestamp | YES | |
| shipments | status | Shipment status | String | NO | NOT NULL |

| shipment_items | shipment_item_id | PK | Integer | NO | PK |
| shipment_items | shipment_id | FK → shipments | Integer | NO | FK |
| shipment_items | order_item_id | FK → order_items | Integer | NO | FK |
| shipment_items | quantity_shipped | Quantity shipped | Integer | NO | CHECK >= 1 |

| returns | return_id | PK | Integer | NO | PK |
| returns | order_id | FK → orders | Integer | NO | FK |
| returns | customer_id | FK → customers | Integer | NO | FK |
| returns | status | Return status | String | NO | NOT NULL |
| returns | request_date | Timestamp | NO | NOT NULL |
| returns | received_date | Timestamp | YES | |
| returns | refund_amount | Decimal | YES | |

| return_items | return_item_id | PK | Integer | NO | PK |
| return_items | return_id | FK → returns | Integer | NO | FK |
| return_items | order_item_id | FK → order_items | Integer | NO | FK |
| return_items | product_id | FK → products | Integer | NO | FK |
| return_items | quantity | Returned quantity | Integer | NO | CHECK >= 1 |
| return_items | reason | Return reason | String | NO | NOT NULL |

| reviews | review_id | PK | Integer | NO | PK |
| reviews | customer_id | FK → customers | Integer | NO | FK |
| reviews | product_id | FK → products | Integer | NO | FK |
| reviews | order_item_id | FK → order_items | Integer | NO | FK |
| reviews | rating | Rating 1..5 | Integer | NO | CHECK 1..5 |
| reviews | comment | Text | String | YES | |
| reviews | created_at | Timestamp | NO | NOT NULL |
| reviews | updated_at | Timestamp | YES | |
