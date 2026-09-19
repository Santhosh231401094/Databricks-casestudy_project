# RetailNova — Databricks Lakehouse

> An end-to-end Databricks Data Engineering case study implementing a
> Medallion Architecture for an omnichannel retail environment.

---

## 📌 Project Overview

**RetailNova** is a fictional omnichannel retail company operating
across multiple cities in India.

The project demonstrates how operational retail data can be ingested,
cleaned, transformed, and modeled into a business-ready analytical
platform using **Databricks**.

The solution implements:

- Medallion Architecture
- Incremental data processing
- Auto Loader
- High-watermark processing
- Data quality checks
- SCD Type 1
- SCD Type 2
- Identity surrogate keys
- Dimensional modeling
- Star schema
- Delta Lake
- Delta Lake optimization and maintenance
- Lakeflow Jobs
- Unity Catalog

> **Note:** All datasets used in this project are **synthetic datasets**
> created specifically for this case study. No real customer,
> transaction, or business data is used.

---

# 🏗️ Architecture

The solution follows the **Bronze → Silver → Gold** Medallion
Architecture.
<img width="1200" height="678" alt="image" src="https://github.com/user-attachments/assets/6c5ace4b-29c5-4239-aee9-7485ba6629e9" />


### Data Flow
<img width="526" height="290" alt="image" src="https://github.com/user-attachments/assets/cd6984a9-0947-4404-87fa-6552853bccb5" />


---

# ⭐ Star Schema

The Gold layer follows a star-schema design.

<img width="1531" height="1621" alt="starchema" src="https://github.com/user-attachments/assets/27c0f145-3272-4e4f-88ed-c1ef6d3535dc" />



```text
                 dim_customer
                      |
                      |
dim_product ---- fact_sales ---- dim_store
                      |
                      |
                  dim_date
```

### Gold Tables

| Type | Table |
|---|---|
| Dimension | `dim_customer` |
| Dimension | `dim_product` |
| Dimension | `dim_store` |
| Dimension | `dim_date` |
| Fact | `fact_sales` |

---

# 🔄 Data Processing

## Customers

```text
Customer Files
      |
      v
Bronze Customers
      |
      v
Silver Customers
      |
      v
dim_customer
      |
      v
SCD Type 2
```

Customers use **high-watermark incremental processing** based on
`updated_at`.

---

## Products

```text
Product Files
      |
      v
Bronze Products
      |
      v
Silver Products
      |
      v
dim_product
```

Products use **high-watermark incremental processing** based on
`updated_at`.

---

## Orders

```text
Order Files
      |
      v
Auto Loader
      |
      v
Bronze Orders
      |
      v
Silver Orders
```

Orders use **Auto Loader** because new files arrive incrementally
throughout the day.

---

## Sales Fact

```text
Silver Orders
      |
      +---- dim_customer
      |
      +---- dim_product
      |
      +---- dim_store
      |
      +---- dim_date
      |
      v
fact_sales
```

The sales fact is incrementally appended after checking for existing
records.

---

# 🧱 Medallion Architecture

## Bronze

The Bronze layer contains ingested source data with minimal
transformation.

```text
retailnova.bronze.customers
retailnova.bronze.products
retailnova.bronze.orders
retailnova.bronze.etl_control
```

---

## Silver

The Silver layer performs:

- Data cleaning
- Type conversion
- Standardization
- Data quality checks
- Duplicate handling
- Transformation required for Gold processing

```text
retailnova.silver.customers
retailnova.silver.products
retailnova.silver.orders
```

---

## Gold

The Gold layer contains business-ready analytical data.

```text
retailnova.gold.dim_customer
retailnova.gold.dim_product
retailnova.gold.dim_store
retailnova.gold.dim_date
retailnova.gold.fact_sales
```

---

# ⚙️ Key Engineering Decisions

| Requirement | Implementation |
|---|---|
| Raw ingestion | Bronze |
| Data cleaning | Silver |
| Analytical data | Gold |
| Incremental order files | Auto Loader |
| Customer incremental processing | High-watermark |
| Product incremental processing | High-watermark |
| Customer history | SCD Type 2 |
| Product updates | SCD Type 1 using Delta MERGE |
| Dimension keys | Identity surrogate keys |
| Analytical model | Star schema |
| Transaction processing | Incremental fact append |
| Delta optimization | Optimized Writes, Auto Compaction, Liquid Clustering |
| Table maintenance | OPTIMIZE and VACUUM |
| Orchestration | Lakeflow Jobs |
| Storage format | Delta Lake |
| Governance | Unity Catalog |
---

## Why Auto Loader?

Order files arrive throughout the day.

Auto Loader is used to discover and process newly arriving files
incrementally.

```text
New Order File
      |
      v
Auto Loader
      |
      v
Bronze Orders
```

---

## Why High-Watermark Processing?

Customers and Products contain an `updated_at` column.

An ETL control table stores the last successfully processed timestamp.

```text
source_name    | last_processed_at
---------------|--------------------------
customers      | 2026-09-10 10:30:00
products       | 2026-09-10 11:00:00
```

The next execution processes records newer than the stored watermark.

---

## Why SCD Type 2?

Customer attributes can change over time.

Instead of overwriting the old customer record, the previous version is
closed and a new version is created.

```text
customer_id | tier   | effective_from | effective_to | is_current
------------|--------|----------------|--------------|------------
C1001       | Silver | 2026-01-01     | 2026-06-15   | false
C1001       | Gold   | 2026-06-15     | NULL         | true
```

This preserves customer history.

---


## Why SCD Type 1 for Products?

Product attributes are maintained using **SCD Type 1** processing
because historical versions of product attributes are not required.

Existing product records are updated with the latest values, while new
products are inserted.

```text
Existing Product → UPDATE
New Product      → INSERT
```
---

## Why Surrogate Keys?

Gold dimensions use system-generated surrogate keys.

```text
customer_id  → customer_key
product_id   → product_key
store_id     → store_key
```

This separates source-system business identifiers from analytical
dimension keys.

---
---

# ⚡ Delta Lake Optimization

Delta Lake optimization techniques are applied to selected Gold-layer
tables to improve write efficiency, data layout, and table maintenance.

### Optimized Tables

| Table | Optimizations |
|---|---|
| `gold.dim_customer` | Optimized Writes, Auto Compaction, Liquid Clustering, OPTIMIZE, VACUUM |
| `gold.dim_product` | Optimized Writes, Auto Compaction, Liquid Clustering, OPTIMIZE, VACUUM |
| `gold.fact_sales` | Optimized Writes, Auto Compaction, Liquid Clustering, OPTIMIZE, VACUUM |

### Optimized Writes

Optimized Writes are enabled for Gold tables to reduce the creation of
small files during write operations.

### Auto Compaction

Auto Compaction is enabled to automatically compact small files created
during data writes.

### Liquid Clustering

Liquid Clustering is applied to selected Gold tables using columns that
are commonly used for filtering and joining.

**Clustering columns:**

| Table | Clustering Column(s) |
|---|---|
| `gold.dim_customer` | `customer_id` |
| `gold.dim_product` | `product_id` |
| `gold.fact_sales` | Selected frequently filtered/joined columns |

Liquid Clustering provides an adaptive data layout without relying on
traditional static partitioning.

### OPTIMIZE

`OPTIMIZE` is used to reorganize existing Delta files and improve file
layout for analytical queries.

### VACUUM

`VACUUM` is used for Delta table maintenance by removing obsolete data
files that are no longer required after the configured retention period.

The complete optimization and maintenance operations are implemented
in the Delta optimization notebook.
---

# 🔧 Technology Stack

| Area | Technology |
|---|---|
| Data Platform | Databricks |
| Processing | PySpark |
| Storage | Delta Lake |
| Delta Optimization | Optimized Writes, Auto Compaction, Liquid Clustering |
| Table Maintenance | OPTIMIZE, VACUUM |
| Governance | Unity Catalog |
| File Ingestion | Auto Loader |
| Orchestration | Lakeflow Jobs |
| Architecture | Medallion Architecture |
| Data Modeling | Star Schema |
| Incremental Processing | High-Watermark |
| Historical Tracking | SCD Type 2 |
| Source File Storage | Unity Catalog Volume |

---

# 🚀 Lakeflow Jobs

The processing is divided into separate jobs.

### Customer Job
<img width="965" height="303" alt="image" src="https://github.com/user-attachments/assets/1a6ad962-975b-4525-a041-2f4c4bf72a02" />

### Product Job
<img width="821" height="245" alt="image" src="https://github.com/user-attachments/assets/cbcae570-1168-4194-bc52-b0854f3ad950" />


### Orders Job
<img width="615" height="251" alt="image" src="https://github.com/user-attachments/assets/1a320d73-23fc-4d41-a297-49254cd412c6" />


### Main Fact Job
<img width="750" height="461" alt="image" src="https://github.com/user-attachments/assets/4ecd2734-35cc-4c81-9dff-10636f0a451b" />


---


# 🗂️ Unity Catalog Structure

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

---

# 📂 Source Data

The source files are stored in a Unity Catalog Volume during execution.

```text
/Volumes/retailnova/bronze/retailnova_source/
```

### Dataset Information

The datasets are **synthetic** and were generated specifically for
this project.

They are designed to simulate realistic retail data and intentionally
include scenarios such as:

- Duplicate records
- Missing values
- Invalid values
- Product updates
- Customer updates
- Multiple order files
- Incremental data arrivals

No real customer or business data is used.

### Dataset Repository Location

Large source datasets are **not included in this repository** to keep
the Git repository lightweight.

A small sample dataset or dataset documentation can be placed under:

```text
data/
└── README.md
```

The `data/README.md` file documents the dataset structure and explains
how the source files are loaded into the Unity Catalog Volume.

---

# 📁 Repository Structure

```text
retailnova_casestudy/
│
├── README.md
│
├── case-study/
│   └── retailnova-case-study.md
│
├── Architecture/
│   ├── Architecture_diagram.png
│   └── starschema.png
│
│
├── data/
│   └── README.md
│
├── notebooks/
│   ├── setup/
│   ├── bronze/
│   ├── silver/
│   └── gold/
│
├── jobs/
    ├── customers-job.yml
    ├── orders-job.yml
    ├── products-job.yml
    └── retailnova-main-job.yml

```

---

# 📚 Documentation

| Document | Description |
|---|---|
| [Case Study](case-study/retailnova-case-study.md) | Complete business scenario and requirements |
| [Architecture](architecture/) | Architecture diagrams |
| [Data](data/README.md) | Synthetic dataset information |

---

# ✅ Implemented Scope

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
- [x] Product SCD Type 1
- [x] Identity surrogate keys
- [x] Star schema
- [x] Incremental fact loading
- [x] Lakeflow Jobs
- [x] Delta Lake
- [x] Optimized Writes
- [x] Auto Compaction
- [x] Liquid Clustering
- [x] OPTIMIZE
- [x] VACUUM
- [x] Unity Catalog
- [x] Unity Catalog Volume

---

# 📌 Project Status

| Component | Status |
|---|---|
| Bronze Customers | ✅ Implemented |
| Silver Customers | ✅ Implemented |
| Customer SCD Type 2 | ✅ Implemented |
| Bronze Products | ✅ Implemented |
| Silver Products | ✅ Implemented |
| Product SCD Type 1 | ✅ Implemented |
| Store Dimension | ✅ Implemented |
| Date Dimension | ✅ Implemented |
| Orders Auto Loader | ✅ Implemented |
| Orders Silver | ✅ Implemented |
| Sales Fact | ✅ Implemented |
| Lakeflow Jobs | ✅ Implemented |
| Delta Optimization & Maintenance | ✅ Implemented |

---

# 📝 Case Study

For the complete business requirements, source environment,
problems, processing requirements, dimensional model, and engineering
decisions:

➡️ **[Read the complete RetailNova Case Study](case-study/retailnova-case-study.md)**

---

# ⚠️ Project Note

This is a **portfolio and learning project** designed to demonstrate
practical Data Engineering concepts using Databricks.

All business names, records, customers, products, orders, and other
dataset values are synthetic and created for this case study.
