# Data Flow

This document details the end-to-end data lifecycle for the Zavy platform, mapping the trajectory of records from operational creation in the OLTP database through extraction, transformation, dimensional modeling, analytical computation, and BI visualization.

---

## 1. End-to-End Pipeline Stages

```text
┌────────────────────────┐       ┌────────────────────────┐       ┌────────────────────────┐
│ 1. Transactional Write │  ──►  │ 2. Incremental Extract │  ──►  │ 3. Watermark Tracking  │
│   (PostgreSQL OLTP)    │       │ (updated_at > wm)      │       │ (Metadata Checkpoints) │
└────────────────────────┘       └────────────────────────┘       └────────────────────────┘
                                                                               │
                                                                               ▼
┌────────────────────────┐       ┌────────────────────────┐       ┌────────────────────────┐
│ 6. Clean & Transform   │  ◄──  │     5. Validation      │  ◄──  │       4. Staging       │
│ (Enrichment / Struct)  │       │ (Rules & Rejections)   │       │   (In-Memory Buffers)  │
└────────────────────────┘       └────────────────────────┘       └────────────────────────┘
            │
            ▼
┌────────────────────────┐       ┌────────────────────────┐       ┌────────────────────────┐
│    7. Deduplication    │  ──►  │  8. Dimension Loading  │  ──►  │ 9. SCD Type 2 History  │
│ (PK / Natural Keys)    │       │ (Surrogate Key Assign) │       │ (Version Flagging)     │
└────────────────────────┘       └────────────────────────┘       └────────────────────────┘
                                                                               │
                                                                               ▼
┌────────────────────────┐       ┌────────────────────────┐       ┌────────────────────────┐
│  12. BI Visualizations │  ◄──  │  11. Analytical SQL    │  ◄──  │    10. Fact Loading    │
│ (Power BI Dashboards)  │       │ (Views & Mat Views)    │       │ (Grain Reconciliation) │
└────────────────────────┘       └────────────────────────┘       └────────────────────────┘
```

### Stage 1: Operational Transaction Creation
* Events originate from marketplace business workflows (e.g., customers placing orders, sellers updating inventory, deliveries being completed).
* Data is written into normalized 3NF tables in `ecom_oltp` within atomic ACID transactions.
* Every transactional entity records its insertion (`created_at`) and modification (`updated_at`) timestamp.
* Order line items capture their immutable operational pricing snapshots: `unit_price`, `unit_cost`, and `commission_rate`.

### Stage 2: Incremental Extraction
* The extraction engine runs on a scheduled or trigger-driven batch cadence.
* Rather than extracting full tables, the query engine extracts delta slices:
  $$\text{Target Records} = \{ r \mid r.\text{updated\_at} > \text{watermark}_{\text{previous}} \land r.\text{updated\_at} \le \text{watermark}_{\text{current}} \}$$
* Extracted records are read using indexed timestamp columns to avoid sequential scans on operational tables.

### Stage 3: Watermark and Checkpoint Tracking
* A dedicated pipeline metadata ledger records execution state for each entity:
  * Entity name
  * Pipeline run ID
  * High-watermark timestamp extracted
  * Batch start and completion timestamps
  * Total source rows read, processed, and quarantined
  * Execution status (`SUCCESS`, `FAILED`, `IN_PROGRESS`)
* If a run fails, the watermark remains at the last committed checkpoint, enabling safe retry without data corruption.

### Stage 4: Staging
* Extracted batch records land in transient staging structures (in-memory DataFrames or transient staging schemas).
* Staging operates without enforcing foreign key constraints, allowing raw batch data to be ingested rapidly and examined as a whole.

### Stage 5: Validation and Rejection Handling
* The validation engine executes deterministic data quality rules against staged batches:
  * **Schema and Type Validation**: Required fields present, correct data types, valid timestamp formats.
  * **Range and Domain Checks**: Price $> 0$, cost $\ge 0$, quantity $\ge 1$, seller commission rate between $0\%$ and $30\%$.
  * **Referential Integrity Checks**: Verifies that parent identifiers exist in the warehouse dimension lookup index.
  * **Business Rule Validation**: Status transitions adhere to defined state graphs (e.g., no cancellation after `SHIPPED`).
* Records failing validation are written to a quarantine/rejection log with explicit error reason codes.
* Row reconciliation is recorded: $\text{Source Rows} = \text{Processed Rows} + \text{Rejected Rows}$.

### Stage 6: Cleaning and Transformation
* Validated records undergo standardization and enrichment:
  * Text trimming and case normalization (e.g., state/country codes, category names).
  * Date/time component parsing for time-dimension alignment.
  * Calculation of transaction-level metrics (e.g., line-item GMV, commission value, seller sales value).
  * Aggregation of stock movements into historical daily inventory snapshot records.

### Stage 7: Deduplication
* In-flight duplicates (multiple updates to the same record within a single extraction window) are resolved.
* The transformation engine retains the latest state per natural key based on `updated_at` before attempting database upserts.

### Stage 8: Dimension Loading and Surrogate Key Assignment
* Dimensions are processed prior to fact tables.
* For each record, the loader checks whether the natural key already exists in the dimension layer:
  * **New Natural Key**: Assigns a new surrogate key, sets `effective_date`, `end_date = NULL`, `is_current = TRUE`, and inserts the record.
  * **Existing Natural Key (Unchanged Attributes)**: Skips or performs an in-place update for non-tracked attributes.
  * **Existing Natural Key (Tracked Attribute Changed)**: Routes the record to the SCD Type 2 pipeline.

### Stage 9: SCD Type 2 Historical Processing
* When a tracked attribute changes (e.g., a customer changes their city/address):
  1. The existing active dimension record is updated: `end_date` is set to the change timestamp and `is_current` is flipped to `FALSE`.
  2. A new record is inserted with a new surrogate key, containing the updated attribute values, `effective_date = change_timestamp`, `end_date = NULL`, and `is_current = TRUE`.
* This preserves historical accuracy: historical fact records point to the surrogate key that was active when the event occurred, while future facts point to the new surrogate key.

### Stage 10: Fact Table Loading
* Once all dimensions are refreshed and surrogate key lookup maps are updated, fact tables are loaded.
* Each transactional event is mapped to the appropriate dimensional surrogate keys using the event timestamp:
  $$\text{Surrogate Key} = \text{Lookup}(\text{Natural Key}, \text{Event Timestamp})$$
* Grain rules are strictly enforced (for example, the sales fact grain is one row per order line item).
* Fact records are loaded via idempotent upsert operations to ensure pipeline re-runs do not create duplicate facts.

### Stage 11: Analytical SQL Computation
* The semantic layer queries the star schema to evaluate business logic and KPIs.
* Views and materialized views pre-aggregate complex multi-table joins:
  * Daily/Monthly GMV, Zavy Commission Revenue, and Net Commission Revenue.
  * Customer Lifetime Value and retention cohort matrices.
  * Seller performance and return rates.
  * Historical inventory turnover and out-of-stock rates derived from inventory snapshot tables.

### Stage 12: Power BI Consumption
* Power BI queries the PostgreSQL Data Warehouse semantic views via direct SQL queries / scheduled dataset refreshes.
* Visual models, KPI summary cards, time-series charts, and interactive slice-and-dice filters are rendered for decision-makers.

---

## 2. Specific Event Flows

### 2.1 New Order Placement Flow
```text
1. Customer places order (Items A, B)
   ↓
2. OLTP writes:
   - orders (status: PLACED, subtotal, shipping, tax, total)
   - order_items (snapshot unit_price, unit_cost, commission_rate)
   - payments (status: PENDING)
   ↓
3. Payment succeeds → payments updated to PAID; orders updated to CONFIRMED
   ↓
4. Incremental extraction captures updated order and item records
   ↓
  5. ETL maps customer and product identifiers to the current warehouse dimension keys and assigns the date dimension key
   ↓
  6. The sales fact table receives two rows (one per line item) with gross GMV and commission amounts
```

### 2.2 Customer Address / Location Change Flow (SCD Type 2)
```text
1. Customer 101 moves from Lahore to Islamabad
   ↓
2. OLTP updates:
   - customer_addresses (updates city to Islamabad, updated_at = T2)
   - OLTP maintains ONLY the current address
   ↓
3. Incremental extraction detects customer address change at T2
   ↓
4. SCD Type 2 Engine:
    - Locates the active customer dimension record for Customer 101 (Lahore)
    - Marks the old record as expired and inserts a new record for Islamabad with a new surrogate key
   ↓
5. Historical reporting integrity:
    - Orders placed before T2 reference the older customer dimension record (attributed to Lahore)
    - Orders placed after T2 reference the newer customer dimension record (attributed to Islamabad)
```

### 2.3 Order Lifecycle and Cancellation Flow
```text
1. Order placed at T1 (CONFIRMED) → Sales fact record created
   ↓
2. Customer cancels before shipment at T2:
   - OLTP updates orders (status = CANCELLED, updated_at = T2)
   - payments updated to REFUNDED
   ↓
3. Incremental extraction captures status change at T2
   ↓
4. Warehouse updates the related analytical records
   ↓
5. KPI views exclude CANCELLED orders from GMV, Order Count, and Commission Revenue
```

### 2.4 Inventory Movement and Snapshot Flow
```text
1. Warehouse receives stock or fulfills an order item
   ↓
2. OLTP writes:
   - stock_movements (type: RECEIPT or SALE, quantity, timestamp)
   - updates inventory table (current on_hand)
   ↓
3. Incremental extraction captures stock_movements
   ↓
4. Warehouse ETL processes daily movements and aggregates end-of-day balances
   ↓
5. The inventory snapshot table stores daily on_hand per product per warehouse
   ↓
6. Inventory Turnover and global Out-of-Stock KPIs compute against historical snapshots
```

### 2.5 Return Processing Flow
```text
1. Customer requests return for delivered item within 30-day return window
   ↓
2. Return workflow: REQUESTED → APPROVED → RETURNED (received at warehouse) → REFUNDED
   ↓
3. OLTP updates:
   - returns (status: REFUNDED, refund_amount)
   - return_items (reason, quantity)
   - stock_movements (type: RETURN_IN, quantity restocked)
   - payments (status: PARTIALLY_REFUNDED / REFUNDED)
   ↓
4. Incremental extraction extracts return records
   ↓
5. ETL loads return facts and adjusts net revenue metrics in analytical views:
   - Deducts returned GMV from Net GMV
   - Deducts refunded commission from Net Commission Revenue
   - Recalculates Product Return Rate and Seller Return Rate
```
