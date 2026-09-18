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
- SCD Type 2
- Identity surrogate keys
- Dimensional modeling
- Star schema
- Lakeflow Jobs
- Delta Lake
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

<img width="2302" height="2120" alt="star-schema" src="https://github.com/user-attachments/assets/29669162-c5f0-4f61-87f5-34cd163a1053" />


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
| Dimension keys | Identity surrogate keys |
| Analytical model | Star schema |
| Transaction processing | Incremental fact append |
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

# 🔧 Technology Stack

| Area | Technology |
|---|---|
| Data Platform | Databricks |
| Processing | PySpark |
| Storage | Delta Lake |
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
├── architecture/
│   ├── retailnova-architecture.png
│   └── retailnova-star-schema.png
│
├── screenshots/
│   ├── jobs-overview.png
│   ├── customer-job.png
│   ├── product-job.png
│   ├── orders-job.png
│   ├── notebooks.png
│   ├── unity-catalog.png
│   └── gold-tables.png
│
├── data/
│   └── README.md
│
├── src/
│   ├── setup/
│   ├── bronze/
│   ├── silver/
│   └── gold/
│
├── resources/
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
- [x] Identity surrogate keys
- [x] Star schema
- [x] Incremental fact loading
- [x] Lakeflow Jobs
- [x] Delta Lake
- [x] Unity Catalog
- [x] Unity Catalog Volume
- [x] Delta maintenance

---

# 📌 Project Status

| Component | Status |
|---|---|
| Bronze Customers | ✅ Implemented |
| Silver Customers | ✅ Implemented |
| Customer SCD Type 2 | ✅ Implemented |
| Bronze Products | ✅ Implemented |
| Silver Products | ✅ Implemented |
| Product Dimension | ✅ Implemented |
| Store Dimension | ✅ Implemented |
| Date Dimension | ✅ Implemented |
| Orders Auto Loader | ✅ Implemented |
| Orders Silver | ✅ Implemented |
| Sales Fact | ✅ Implemented |
| Lakeflow Jobs | ✅ Implemented |
| Delta Maintenance | ✅ Implemented |

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
