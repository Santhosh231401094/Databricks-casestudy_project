# RetailNova — Databricks Lakehouse Modernization

> **Case Study:** RetailNova  
> **Platform:** Databricks  
> **Architecture:** Medallion Architecture  
> **Domain:** Omnichannel Retail  
> **Processing:** Batch + Incremental File Processing

---

## 1. Company Overview

**RetailNova** is a fictional omnichannel retail company operating
across multiple cities in India.

The company receives data from different operational systems such as
customer, product, store, and order management systems.

The existing data is stored as files and needs to be transformed into
a centralized analytical platform using **Databricks**.

### Business Objective

RetailNova wants to build a reliable data platform that can:

- Process continuously arriving data
- Standardize data from multiple sources
- Improve data quality
- Preserve historical customer information
- Provide business-ready analytical data
- Support incremental processing
- Reduce repeated full-data processing

---

# 2. Existing Environment

RetailNova currently receives data from multiple operational systems.

| Source System | Data Type | Arrival Pattern | Key Characteristics |
|---|---|---|---|
| Customer System | Customer master | Periodic files | Customer attributes can change |
| Product System | Product master | Periodic files | Prices and attributes can change |
| Store System | Store master | Relatively static | Approximately 500 stores |
| Order System | Transactional | Multiple files throughout the day | Largest transactional source |

The source systems are independent and do not provide a standardized
analytical data model.

---

# 3. Source Data Environment

## 3.1 Customer System

Customer files contain the following attributes:

| Column | Description |
|---|---|
| `customer_id` | Business identifier |
| `customer_name` | Customer name |
| `email` | Customer email |
| `phone` | Customer phone |
| `city` | Customer city |
| `state` | Customer state |
| `customer_tier` | Customer classification |
| `signup_date` | Customer registration date |
| `updated_at` | Last update timestamp |

### Source Characteristics

- Customer records can change over time.
- The `updated_at` column identifies changed records.
- Historical customer information must be preserved.

---

## 3.2 Product System

Product files contain:

| Column | Description |
|---|---|
| `product_id` | Business identifier |
| `product_name` | Product name |
| `category` | Product category |
| `subcategory` | Product subcategory |
| `brand` | Product brand |
| `unit_price` | Selling price |
| `cost_price` | Product cost |
| `supplier_id` | Supplier identifier |
| `updated_at` | Last update timestamp |

### Source Characteristics

Product data may contain:

- Incorrect prices
- Negative values
- Inconsistent category names
- Missing attributes
- Updated product records

---

## 3.3 Store System

Store files contain:

| Column | Description |
|---|---|
| `store_id` | Store identifier |
| `store_name` | Store name |
| `city` | Store city |
| `state` | Store state |
| `region` | Business region |
| `store_type` | Store classification |
| `opening_date` | Store opening date |

### Source Characteristics

- Store data is relatively static.
- Approximately 500 stores are maintained.
- Store data does not require continuous incremental processing.

---

## 3.4 Order System

Orders contain:

| Column | Description |
|---|---|
| `order_id` | Order identifier |
| `customer_id` | Customer identifier |
| `product_id` | Product identifier |
| `store_id` | Store identifier |
| `order_timestamp` | Order timestamp |
| `quantity` | Quantity purchased |
| `unit_price` | Selling price |
| `discount_amount` | Discount applied |
| `status` | Order status |
| `payment_method` | Payment method |
| `updated_at` | Last update timestamp |

### Source Characteristics

- Orders are the largest transactional source.
- New files arrive throughout the day.
- Duplicate records may occur.
- The source must be processed incrementally.

---

# 4. Business Problems

RetailNova has several data engineering challenges that must be solved
before the data can be used for analytics.

---

## Problem 1 — Incremental File Arrivals

Order files continuously arrive throughout the day.

A solution that repeatedly processes the entire source data would
increase processing time and resource usage.

### Requirement

The solution must process newly arriving order files incrementally
without requiring manual identification of every new file.

---

## Problem 2 — No Standardized Data Layers

The source systems provide raw operational data.

There is no clear separation between:

- Raw ingested data
- Cleaned and standardized data
- Business-ready analytical data

### Requirement

The solution must implement a **Medallion Architecture**:

```text
Bronze
   ↓
Silver
   ↓
Gold
```

Each layer should have a clear responsibility.

---

## Problem 3 — Data Quality Issues

Source data may contain data quality problems such as:

- Null identifiers
- Missing attributes
- Invalid prices
- Invalid quantities
- Negative values
- Inconsistent category names
- Duplicate records

### Requirement

The solution must identify and handle data quality issues during
Silver-layer processing.

Invalid records must not silently become business-ready Gold data.

---

## Problem 4 — Duplicate Records

Duplicate order records may occur during file ingestion.

For example:

```text
order_id
--------
ORD1001
ORD1002
ORD1002
ORD1003
```

The same transaction may therefore appear more than once.

### Requirement

The solution must apply deterministic duplicate handling before
creating the analytical sales fact.

---

## Problem 5 — Customer History Must Be Preserved

Customer attributes can change over time.

For example:

```text
Customer: C1001

2026-01-01
Customer Tier = Silver

        ↓

2026-06-15
Customer Tier = Gold
```

The previous customer state must not simply be overwritten.

### Requirement

The solution must preserve historical customer versions using
**Slowly Changing Dimension Type 2 (SCD Type 2)**.

---

# 5. Incremental Processing Requirements

Different source systems have different incremental-processing
requirements.

The solution must select an appropriate incremental strategy based on
the source characteristics.

---

## 5.1 Customer Incremental Processing

The Customer source contains an `updated_at` column.

The solution must:

1. Store the last successfully processed timestamp.
2. Read customer records from the source.
3. Select records newer than the stored timestamp.
4. Process the new or changed records.
5. Update the stored watermark after successful processing.

### Required Mechanism

**High-watermark control table**

Example:

```text
source_name    | last_processed_at
---------------|--------------------------
customers      | 2026-09-10 10:30:00
```

---

## 5.2 Product Incremental Processing

The Product source also contains an `updated_at` column.

The solution must use a high-watermark mechanism to identify new or
changed product records.

### Required Mechanism

**High-watermark control table**

Example:

```text
source_name    | last_processed_at
---------------|--------------------------
products       | 2026-09-10 11:00:00
```

---

## 5.3 Order Incremental Processing

Orders arrive as multiple files throughout the day.

The solution must use **Auto Loader** to incrementally discover and
process newly arriving files.

### Required Mechanism

**Databricks Auto Loader**

```text
New Order Files
      ↓
Auto Loader
      ↓
Bronze Orders
```

---

## 5.4 Incremental Strategy Summary

| Source | Incremental Requirement | Implementation |
|---|---|---|
| Customers | Process new/changed records | High-watermark |
| Products | Process new/changed records | High-watermark |
| Orders | Process newly arriving files | Auto Loader |
| Stores | Relatively static | Initial/static load |

> **Important:** The high-watermark mechanism used for Customers and
> Products is different from Spark Structured Streaming event-time
> watermarking. It is a batch incremental-processing control mechanism.

---

# 6. Data Processing Requirements

The solution must implement a three-layer **Medallion Architecture**.

```text
                 SOURCE DATA
                     |
                     ↓
                ┌─────────┐
                │ BRONZE  │
                │  Raw    │
                └────┬────┘
                     |
                     ↓
                ┌─────────┐
                │ SILVER  │
                │ Cleaned │
                │Standard │
                └────┬────┘
                     |
                     ↓
                ┌─────────┐
                │  GOLD   │
                │Business │
                │ Ready   │
                └─────────┘
```

---

## 6.1 Bronze Layer

The Bronze layer must contain the ingested source data with minimal
transformation.

### Bronze Tables

```text
retailnova.bronze.customers
retailnova.bronze.products
retailnova.bronze.orders
```

### Bronze Responsibilities

- Ingest source data
- Preserve source information
- Support incremental ingestion
- Provide a reliable source for downstream processing

---

## 6.2 Silver Layer

The Silver layer must contain cleaned and standardized data.

### Silver Responsibilities

- Remove or identify invalid records
- Trim string values
- Standardize categories
- Convert data types
- Validate numeric values
- Handle duplicates
- Prepare data for Gold processing

### Silver Tables

```text
retailnova.silver.customers
retailnova.silver.products
retailnova.silver.orders
```

---

## 6.3 Gold Layer

The Gold layer must contain business-ready analytical data.

The Gold layer must follow a **dimensional/star-schema design**.

### Gold Dimensions

```text
retailnova.gold.dim_customer
retailnova.gold.dim_product
retailnova.gold.dim_store
retailnova.gold.dim_date
```

### Gold Fact

```text
retailnova.gold.fact_sales
```

---

# 7. Data Quality Requirements

The Silver layer must perform quality checks before data is used by
the Gold layer.

## Customer Quality Checks

Examples:

- Validate `customer_id`
- Trim customer attributes
- Standardize text values
- Convert timestamps correctly

---

## Product Quality Checks

Examples:

- Validate `product_id`
- Validate prices
- Identify negative prices
- Standardize category names
- Handle missing attributes
- Convert numeric columns to appropriate types

---

## Order Quality Checks

Examples:

- Validate `order_id`
- Validate `customer_id`
- Validate `product_id`
- Validate `store_id`
- Validate quantity
- Validate unit price
- Validate discount
- Remove duplicate records

---

# 8. Analytical Model Requirement

The Gold layer must follow a **Star Schema**.

The model consists of:

### Dimensions

- `dim_customer`
- `dim_product`
- `dim_store`
- `dim_date`

### Fact

- `fact_sales`

---

## 8.1 Star Schema

```text
                       ┌─────────────────┐
                       │  dim_customer   │
                       │-----------------│
                       │ customer_key    │
                       │ customer_id     │
                       │ customer_name   │
                       │ customer_tier   │
                       └────────┬────────┘
                                │
                                │
┌─────────────────┐             │             ┌─────────────────┐
│   dim_product   │             │             │    dim_store    │
│-----------------│             │             │-----------------│
│ product_key     │             │             │ store_key       │
│ product_id      │             │             │ store_id        │
│ product_name    │             │             │ store_name      │
│ category        │             │             │ region          │
└────────┬────────┘             │             └────────┬────────┘
         │                      │                      │
         │                      ▼                      │
         │             ┌─────────────────┐             │
         └────────────►│   fact_sales    │◄────────────┘
                       │-----------------│
                       │ sales_key       │
                       │ order_id        │
                       │ customer_key    │
                       │ product_key     │
                       │ store_key       │
                       │ date_key        │
                       │ quantity        │
                       │ unit_price      │
                       │ discount_amount │
                       │ sales_amount    │
                       └────────┬────────┘
                                │
                                │
                       ┌────────▼────────┐
                       │    dim_date     │
                       │-----------------│
                       │ date_key        │
                       │ full_date       │
                       │ year            │
                       │ quarter         │
                       │ month           │
                       └─────────────────┘
```

---

# 9. Dimension Requirements

## 9.1 Customer Dimension

Customer history must be preserved using **SCD Type 2**.

### Table

```text
retailnova.gold.dim_customer
```

### Required Columns

| Column | Purpose |
|---|---|
| `customer_key` | Surrogate key |
| `customer_id` | Source/business key |
| `customer_name` | Customer name |
| `city` | Customer city |
| `state` | Customer state |
| `customer_tier` | Customer tier |
| `effective_from` | Start of record version |
| `effective_to` | End of record version |
| `is_current` | Indicates current version |

### Example

```text
customer_key | customer_id | customer_tier | effective_from | effective_to | is_current
-------------|-------------|---------------|----------------|--------------|-----------
1            | C1001       | Silver        | 2026-01-01     | 2026-06-15   | false
2            | C1001       | Gold          | 2026-06-15     | NULL         | true
```

---

## 9.2 Product Dimension

### Table

```text
retailnova.gold.dim_product
```

The Product dimension stores the current standardized product
information.

The solution must:

- Insert new products
- Update existing products
- Maintain one current product record per `product_id`

A Delta `MERGE` operation is used to synchronize the Product dimension.

---

## 9.3 Store Dimension

### Table

```text
retailnova.gold.dim_store
```

Store data is relatively static.

The solution does not require a recurring incremental Store job.

The Store dimension can be loaded initially and used by downstream
fact processing.

---

## 9.4 Date Dimension

### Table

```text
retailnova.gold.dim_date
```

The Date dimension is a static reference dimension.

It contains calendar attributes such as:

- `date_key`
- `full_date`
- `year`
- `quarter`
- `month`
- `month_name`
- `day`
- `day_of_week`
- `day_name`

The Date dimension does not need to be regenerated for every fact load.

---

# 10. Sales Fact Requirement

The project must create a business-ready sales fact.

### Table

```text
retailnova.gold.fact_sales
```

### Grain

The intended grain is:

> **One product line per customer order.**

### Required Columns

| Column | Purpose |
|---|---|
| `sales_key` | Surrogate fact key |
| `order_id` | Source order identifier |
| `customer_key` | Customer dimension key |
| `product_key` | Product dimension key |
| `store_key` | Store dimension key |
| `date_key` | Date dimension key |
| `order_timestamp` | Order timestamp |
| `quantity` | Quantity purchased |
| `unit_price` | Selling price |
| `discount_amount` | Discount amount |
| `sales_amount` | Calculated sales amount |
| `status` | Order status |
| `payment_method` | Payment method |

---

## Sales Amount Calculation

```text
sales_amount =
(quantity × unit_price) - discount_amount
```

---

# 11. Surrogate Key Requirements

The Gold dimensions must use surrogate keys.

The following dimensions use system-generated identity keys:

```text
dim_customer → customer_key
dim_product  → product_key
dim_store    → store_key
```

The source-system identifiers remain as business keys:

```text
customer_id
product_id
store_id
```

### Example

```text
customer_key | customer_id
-------------|------------
1            | C1001
2            | C1002
3            | C1003
```

The surrogate key is generated by the Gold dimension rather than being
calculated manually from the source data.

---

# 12. Orchestration Requirements

The solution must use separate processing jobs for the major data
domains.

---

## 12.1 Customer Job

```text
Bronze Customers
       ↓
Silver Customers
       ↓
dim_customer
```

The Customer job handles:

- Incremental Bronze processing
- Silver transformation
- Customer SCD Type 2 processing

---

## 12.2 Product Job

```text
Bronze Products
       ↓
Silver Products
       ↓
dim_product
```

The Product job handles:

- Incremental Bronze processing
- Silver data cleaning
- Product dimension updates

---

## 12.3 Orders Job

```text
Order Files
     ↓
Auto Loader
     ↓
Bronze Orders
     ↓
Silver Orders
```

The Orders job handles:

- Incremental file discovery
- Bronze ingestion
- Silver cleaning
- Duplicate handling

---

## 12.4 Main Fact Job

```text
                    Silver Orders
                         |
        +----------------+----------------+
        |                |                |
        ↓                ↓                ↓
  dim_customer     dim_product       dim_store
        |                |                |
        +----------------+----------------+
                         |
                     dim_date
                         |
                         ↓
                    fact_sales
```

The Main Fact job creates the analytical sales fact from the cleaned
orders and Gold dimensions.

---

# 13. Platform Requirements

The solution must use the Databricks platform and the following
components:

| Category | Technology |
|---|---|
| Data Platform | Databricks |
| Processing | PySpark |
| Storage Format | Delta Lake |
| Governance | Unity Catalog |
| File Ingestion | Auto Loader |
| Orchestration | Lakeflow Jobs |
| Architecture | Medallion Architecture |
| Data Modeling | Star Schema |
| Incremental Processing | High-Watermark |
| Historical Tracking | SCD Type 2 |
| Source Storage | Unity Catalog Volume |

---

# 14. Unity Catalog Structure

The project uses the following Unity Catalog structure:

```text
retailnova
│
├── bronze
│   ├── customers
│   ├── products
│   ├── orders
│   └── etl_control
│
├── silver
│   ├── customers
│   ├── products
│   └── orders
│
└── gold
    ├── dim_customer
    ├── dim_product
    ├── dim_store
    ├── dim_date
    └── fact_sales
```

Source files are stored in a Unity Catalog Volume:

```text
/Volumes/retailnova/bronze/retailnova_source/
```

---

# 15. ETL Control Requirements

The Customer and Product incremental pipelines must maintain a
centralized control table.

### Table

```text
retailnova.bronze.etl_control
```

### Structure

| Column | Description |
|---|---|
| `source_name` | Source/pipeline identifier |
| `last_processed_at` | Last successfully processed timestamp |

### Example

```text
source_name       | last_processed_at
------------------|--------------------------
customers         | 2026-09-10 10:30:00
products          | 2026-09-10 11:00:00
```

### Processing Logic

```text
Read Last Watermark
        ↓
Read Source Data
        ↓
Filter updated_at > watermark
        ↓
Process New/Changed Records
        ↓
Write Target
        ↓
Update Watermark
```

---

# 16. Implementation Constraints

The current implementation uses a **Unity Catalog Volume** for source
files.

An active external ADLS landing zone is not part of the current
environment.

Therefore:

```text
Source Files
     ↓
Unity Catalog Volume
     ↓
Databricks
```

The processing architecture is designed so that the landing layer can
later be changed to an external cloud storage location if required.

---

# 17. Engineering Decisions

## Why Medallion Architecture?

The Bronze, Silver, and Gold layers separate different processing
responsibilities.

```text
Bronze → Ingestion
Silver → Cleaning + Standardization
Gold   → Business + Analytics
```

This makes the pipeline easier to understand, maintain, and
troubleshoot.

---

## Why Auto Loader for Orders?

Orders arrive as multiple files throughout the day.

Auto Loader is used because the requirement is incremental **file
discovery and ingestion**.

```text
New File
   ↓
Auto Loader
   ↓
Bronze
```

---

## Why High-Watermark Processing for Customers and Products?

Customers and Products contain an `updated_at` column.

The ETL control table stores the last successfully processed timestamp.

The next run only processes records newer than that timestamp.

```text
Control Table
      ↓
Last Processed Timestamp
      ↓
Filter updated_at
      ↓
New/Changed Records
```

---

## Why SCD Type 2 for Customers?

Customer attributes can change over time.

SCD Type 2 preserves previous versions instead of overwriting them.

This allows historical customer states to remain available.

---

## Why Identity Surrogate Keys?

Surrogate keys provide Gold dimensions with their own internal keys.

The source-system business keys remain available separately.

```text
Business Key        Surrogate Key

customer_id  ─────► customer_key
product_id   ─────► product_key
store_id     ──────► store_key
```

The keys are generated by the Gold dimension rather than being manually
calculated from Spark partitions.

---

## Why Append for Fact Sales?

Sales transactions are treated as append-oriented fact records.

Before inserting records into `fact_sales`, the pipeline checks for
already-existing fact records.

```text
Incoming Fact Data
        ↓
Compare With Existing Fact Data
        ↓
Keep New Records
        ↓
Append
```

This prevents the same incoming records from being inserted repeatedly
during reruns.

---

## Why Store Is Not in a Recurring Job?

Store data is relatively static.

Therefore, Store does not need to be processed every time the sales
fact runs.

The fact pipeline reads the existing `dim_store`.

---

## Why Date Is Not in a Recurring Job?

The Date dimension is a static calendar/reference table.

It can be created once and extended when future dates are required.

There is no need to regenerate it during every sales-fact execution.

---

# 18. Final Solution Architecture

```text
                         SOURCE FILES
                              |
              +---------------+---------------+
              |               |               |
              ▼               ▼               ▼
         CUSTOMERS        PRODUCTS          ORDERS
              |               |               |
              ▼               ▼               ▼
           BRONZE          BRONZE         AUTO LOADER
              |               |               |
              ▼               ▼               ▼
           SILVER          SILVER          BRONZE
              |               |               |
              ▼               ▼               ▼
        DIM_CUSTOMER    DIM_PRODUCT        SILVER
              |               |               |
              +---------------+---------------+
                              |
                   +----------+----------+
                   |          |          |
                   ▼          ▼          ▼
               DIM_STORE  DIM_DATE  SILVER ORDERS
                   |          |          |
                   +----------+----------+
                              |
                              ▼
                         FACT_SALES
```

---

# 19. Implemented Project Scope

The current implemented scope includes:

- [x] Customer ingestion
- [x] Product ingestion
- [x] Store ingestion
- [x] Order ingestion
- [x] Bronze layer
- [x] Silver layer
- [x] Gold layer
- [x] Auto Loader
- [x] High-watermark processing
- [x] ETL control table
- [x] Customer SCD Type 2
- [x] Identity surrogate keys
- [x] Star schema
- [x] Incremental fact loading
- [x] Lakeflow Jobs
- [x] Delta Lake
- [x] Unity Catalog
- [x] Unity Catalog Volumes
- [x] Delta maintenance

---

# 20. Implementation Status

| Component | Status |
|---|---|
| Customer Bronze | ✅ Implemented |
| Customer Silver | ✅ Implemented |
| Customer SCD Type 2 | ✅ Implemented |
| Product Bronze | ✅ Implemented |
| Product Silver | ✅ Implemented |
| Product Gold | ✅ Implemented |
| Store Gold | ✅ Implemented |
| Date Dimension | ✅ Implemented |
| Orders Auto Loader | ✅ Implemented |
| Orders Silver | ✅ Implemented |
| Sales Fact | ✅ Implemented |
| Lakeflow Jobs | ✅ Implemented |
| Delta Maintenance | ✅ Implemented |

---

# 21. Project Structure

```text
retailnova_casestudy/
│
├── README.md
│
├── case-study/
│   └── retailnova-case-study.md
│
├── architecture/
│   ├── retailnova-architecture.png
│   └── retailnova-star-schema.png
│
├── src/
│   ├── setup/
│   ├── bronze/
│   ├── silver/
│   └── gold/
│
├── resources/
│   └── Lakeflow Job definitions
│
├── tests/
│
├── fixtures/
│
├── databricks.yml
├── pyproject.toml
├── .gitignore
├── AGENTS.md
└── CLAUDE.md
```

---

# 22. Final Outcome

The RetailNova project demonstrates an end-to-end Databricks Data
Engineering solution that transforms operational retail files into a
structured analytical platform.

The implementation demonstrates:

- Incremental file ingestion
- High-watermark processing
- Medallion Architecture
- Data quality processing
- Customer SCD Type 2
- Identity surrogate keys
- Dimensional modeling
- Star schema
- Incremental fact loading
- Delta Lake
- Unity Catalog
- Lakeflow Jobs
- Delta table maintenance

The resulting Gold layer provides a structured foundation for
downstream analytics and reporting.
