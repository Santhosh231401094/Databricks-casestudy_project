# Dataset

This project uses synthetic datasets created for the RetailNova case study.

The datasets simulate an omnichannel retail environment containing:

- Customers
- Products
- Stores
- Orders

The data includes realistic data-engineering scenarios such as:

- Incremental customer and product updates
- Multiple order files
- Duplicate records
- Missing and invalid values
- Inconsistent category values
- Customer attribute changes for SCD Type 2 processing

## Dataset Storage

The complete synthetic datasets are not stored in this GitHub repository.

They are loaded into a Databricks Unity Catalog Volume:

`/Volumes/retailnova/bronze/retailnova_source/retailnova_datasets/`

The repository contains the processing code and configuration required to work with the datasets.

> **Note:** All datasets used in this project are synthetic and do not contain real customer or business information.
