1. Company Overview

RetailNova is a fictional omnichannel retail company operating across India.

The company sells products through:

E-commerce
Mobile applications
Physical retail stores

RetailNova maintains separate operational systems for customer, product, store, and order information.

As the business grows, the volume of operational data increases and the analytics team requires a centralized data platform for reporting and analysis.

The company wants to build a Databricks-based analytical platform that can ingest continuously arriving data, process changes incrementally, maintain customer history, and provide a reliable analytical model.

2. Existing Environment

RetailNova's operational systems are primarily file-based.

Different systems generate files at different frequencies.

The existing environment has the following characteristics:

Customer System
      │
      ├── Customer extracts
      │
Product System
      │
      ├── Product extracts
      │
Store System
      │
      ├── Store master
      │
Order System
      │
      └── Transaction files

The files are not directly optimized for analytical workloads.

The company needs to consolidate these sources into a centralized lakehouse.

3. Source Data Environment
3.1 Customer System

The customer system generates periodic customer extracts.

Each customer record contains:

customer_id
customer_name
email
phone
city
state
customer_tier
signup_date
updated_at

Customer information can change over time.

For example:

Customer: C000123

Previous:
City = Chennai
Tier = Silver

Updated:
City = Bengaluru
Tier = Gold

The business needs to preserve these historical changes.

3.2 Product System

The product system generates product extracts.

The data contains:

product_id
product_name
category
subcategory
brand
unit_price
cost_price
supplier_id
updated_at

Product data can contain:

Incorrect prices
Missing attributes
Inconsistent category names
Updated product information
Duplicate versions

The analytical platform must standardize and validate this data.

3.3 Store System

RetailNova maintains a store master containing approximately 500 stores.

The store data contains:

store_id
store_name
city
state
region
store_type
opening_date

Store information is treated as relatively static reference data for the current implementation.

3.4 Order System

The order system is the primary transactional source.

Each order contains:

order_id
customer_id
product_id
store_id
order_timestamp
quantity
unit_price
discount_amount
status
payment_method
updated_at

New order files can arrive throughout the day.

The platform must therefore support incremental file ingestion rather than treating the entire order dataset as one static file.

4. Business Problems
Problem 1 — Incremental File Arrivals

The order system continuously generates new files.

Processing the entire order dataset whenever a new file arrives would create unnecessary processing.

Business Requirement

New files must be detected and processed incrementally.

Problem 2 — No Standardized Data Layers

The source files contain raw operational data.

There is no clear separation between:

Raw Data
   ↓
Clean Data
   ↓
Business Data
Business Requirement

Implement a Medallion Architecture:

Bronze
   ↓
Silver
   ↓
Gold
Problem 3 — Data Quality Issues

Operational data can contain:

Null values
Invalid IDs
Invalid prices
Duplicate records
Inconsistent category values
Incorrect formatting

For example:

product_id = P000123
unit_price = -100

or:

category = Home appliance

when the standardized category should be:

Home Appliances
Business Requirement

Identify and process data-quality issues before the data reaches the analytical layer.

Problem 4 — Duplicate Records

Source files can contain duplicate records because of repeated exports or file reprocessing.

For example:

order_id    customer_id    product_id
O000001     C000123        P000456
O000001     C000123        P000456
Business Requirement

Deduplicate records before they are used for analytical processing.

Problem 5 — Customer History Must Be Preserved

Customer attributes can change over time.

The business does not want the previous customer state to simply disappear.

For example:

C000123
Chennai
Silver

changes to:

C000123
Bengaluru
Gold

The analytics team needs historical customer versions.

Business Requirement

Implement Slowly Changing Dimension Type 2 (SCD Type 2) for customers.

The dimension must maintain:

customer_key
customer_id
customer_name
city
state
customer_tier
effective_from
effective_to
is_current
5. Incremental Processing Requirements

RetailNova does not want every source to be processed using the same technique.

The ingestion method should match the source characteristics.

Customer and Product

Both contain:

updated_at

Therefore, the solution should maintain a high-watermark/control mechanism.

Last processed timestamp
        ↓
Read source
        ↓
Filter updated_at > watermark
        ↓
Process new/changed records
        ↓
Update watermark

A control table is used:

retailnova.bronze.etl_control
Orders

Orders arrive as files.

The solution should use Auto Loader for incremental file ingestion.

New File
   ↓
Auto Loader
   ↓
Bronze Orders

This is different from the batch high-watermark approach used for customers and products.

6. Data Processing Requirements

The solution must transform data through multiple layers.

Bronze

Bronze should contain data close to the source representation.

Responsibilities include:

Source ingestion
Incremental ingestion
Preserving source data
Capturing new files
Silver

Silver should contain cleaned and standardized data.

Responsibilities include:

Null handling
Data-type conversion
Trimming
Standardization
Data-quality classification
Deduplication
Incremental processing
Gold

Gold should provide business-ready analytical data.

The Gold layer should contain:

Dimensions
dim_customer
dim_product
dim_store
dim_date
Fact
fact_sales
7. Analytical Model Requirement

The analytics team needs to answer questions such as:

What are sales by customer?
What are sales by product?
What are sales by store?
What are sales by date?
Which customer attributes were applicable when an order was placed?

The source tables are not designed for these analytical queries.

Requirement

Create a Gold star schema.

                 dim_customer
                      │
                      │
dim_product ──── fact_sales ──── dim_store
                      │
                      │
                   dim_date
8. Dimension Requirements
dim_customer

Customer dimension must support SCD Type 2.

customer_key
customer_id
customer_name
city
state
customer_tier
effective_from
effective_to
is_current
dim_product

The product dimension represents the current product state.

product_key
product_id
product_name
category
subcategory
brand
unit_price
cost_price
supplier_id
updated_at
dim_store

Store is treated as a reference/master dimension in the current implementation.

store_key
store_id
store_name
city
state
region
store_type
opening_date
dim_date

The date dimension is generated as a static calendar dimension.

date_key
full_date
year
quarter
month
month_name
day
day_of_week
day_name

It does not require continuous ingestion.

9. Sales Fact Requirement

The primary analytical fact is fact_sales.

Grain

One product line within one customer order.

The fact contains:

sales_key
order_id
customer_key
product_key
store_key
date_key
order_timestamp
quantity
unit_price
discount_amount
sales_amount
status
payment_method

The fact must resolve the appropriate surrogate keys from the Gold dimensions.

10. Orchestration Requirements

The solution should separate workloads based on their processing requirements.

Customer Job
Customer Bronze
      ↓
Customer Silver
      ↓
dim_customer
Product Job
Product Bronze
      ↓
Product Silver
      ↓
dim_product
Orders Job
Auto Loader
     ↓
Orders Bronze
     ↓
Orders Silver
Gold / Fact Processing
Orders Silver
      +
dim_customer
      +
dim_product
      +
dim_store
      +
dim_date
      ↓
fact_sales

Store and Date are static/reference dimensions and therefore are not required as recurring job tasks.

11. Platform Requirements

The solution should use:

Requirement	Solution
Lakehouse	Databricks
Data organization	Unity Catalog
File landing	Unity Catalog Volume
Raw layer	Bronze Delta tables
Clean layer	Silver Delta tables
Analytical layer	Gold Delta tables
File-based incremental ingestion	Auto Loader
Batch incremental processing	High-watermark
Control mechanism	etl_control
Transformations	PySpark / SQL
Historical customer changes	SCD Type 2
Analytical model	Star schema
Surrogate keys	Delta Identity columns
Orchestration	Lakeflow Jobs
