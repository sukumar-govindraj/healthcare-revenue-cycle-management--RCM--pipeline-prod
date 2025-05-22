# Healthcare RCM Data Engineering Pipeline

An end-to-end Azure Data Engineering solution for Healthcare Revenue Cycle Management (RCM). This project automates ingestion from EMR, claims, and coding systems through a Medallion architecture (Bronze → Silver → Gold), producing high-quality fact and dimension tables.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Domain: Revenue Cycle Management (RCM)](#domain-revenue-cycle-management-rcm)
3. [Technology Stack](#technology-stack)
4. [Data Sources](#data-sources)
5. [Architecture](#architecture)

   * [Medallion Layers](#medallion-layers)
6. [Configuration & Mapping](#configuration--mapping)
7. [Pipeline Components](#pipeline-components)
8. [Project Structure](#project-structure)
9. [Deployment & Scheduling](#deployment--scheduling)
10. [Best Practices & Enhancements](#best-practices--enhancements)
11. [Contributing](#contributing)
12. [License](#license)

---

## Project Overview

This repository implements a robust data pipeline on Azure for RCM, moving raw and API-sourced data into curated Delta tables ready for reporting.

## Domain: Revenue Cycle Management (RCM)

RCM tracks patient services from registration to final payment, covering:

* **Patient Visit:** capture patient & insurance details (\$15k insurance, \$5k patient on \$20k bill)
* **Claims Processing:** insurance reviews and payments
* **Patient Collections:** follow-up on outstanding balances
* **Analytics:** KPIs like Days in AR and % AR > 90 days

## Technology Stack

* **Orchestration:** Azure Data Factory (ADF)
* **Processing:** Azure Databricks (Spark + Delta Lake)
* **Storage:** ADLS Gen2 (bronze, silver, gold layers)
* **Source DB:** Azure SQL Database (EMR)
* **Credentials:** Azure Key Vault
* **Governance:** Unity Catalog (optional)

## Data Sources

| Source               | Type       | Frequency   |
| -------------------- | ---------- | ----------- |
| EMR (Azure SQL DB)   | Table      | Incremental |
| Claims (CSV)         | Flat files | Monthly     |
| ICD/NPI (Public API) | REST API   | On-demand   |
| CPT Codes (CSV)      | Flat files | Monthly     |

## Architecture

### Medallion Layers

1. **Bronze (Raw Parquet)**: landing raw data in parquet format
2. **Silver (Delta + CDM + SCD2)**: cleaned, conformed, and slowly-changing dimension patterns
3. **Gold (Aggregations)**: business-ready fact and dimension tables filtered for current, non-quarantined records

## Configuration & Mapping

* **`load_config.csv`**: metadata-driven pipeline definitions (`database`, `table`, `load_type`, `watermark`, `is_active`, `target_path`)
* **`Lookup_file_table_mapping.json`**: JSON mapping for file-to-table relationships
* **`audit_table_ddl/`**: DDL scripts to create `audit.load_logs` in Azure SQL

## Pipeline Components

1. **Setup** (`1. Set up/`)

   * `audit_ddl.py`: create audit table schema
   * `adls_mount.py`: mount ADLS Gen2 in Databricks
2. **API Extracts** (`2. API extracts/`)

   * `ICD Code API extract.ipynb`
   * `NPI API extract.ipynb`
3. **Silver Transformations** (`3. Silver/`)

   * `Claims.py`, `CPT codes.py`, `Departments_F.py`, `Encounters.py`, `Patient.py`, `Providers_F.py`, `Transactions.py`
   * Notebooks: `ICD Code.ipynb`, `NPI.ipynb`, `data_generator_faker_module.ipynb`
   * `load_config.csv` for dynamic ingestion
4. **EMR Sample Data** (`EMR/`)

   * `trendytech-hospital-a/` and `trendytech-hospital-b/` each with `.csv` files for departments, encounters, patients, providers, transactions
5. **Gold Loads** (`4. Gold/`)

   * Dimension scripts: `dim_cpt_code.py`, `dim_department.py`, `dim_patient.py`, `dim_provider.py`
   * Notebooks: `dim_icd_code.ipynb`, `dim_npi.ipynb`
   * Fact query: `fact_transaction.sql`

## Project Structure

```text
Trendytech-Azure-Project-main/
├── 1. Set up/
│   ├── audit_ddl.py
│   └── adls_mount.py
├── 2. API extracts/
│   ├── ICD Code API extract.ipynb
│   └── NPI API extract.ipynb
├── 3. Silver/
│   ├── CPT codes.py
│   ├── Claims.py
│   ├── Departments_F.py
│   ├── Encounters.py
│   ├── ICD Code.ipynb
│   ├── NPI.ipynb
│   ├── Patient.py
│   ├── Providers_F.py
│   ├── Transactions.py
│   ├── data_generator_faker_module.ipynb
│   └── load_config.csv
├── EMR/
│   ├── trendytech-hospital-a/
│   │   ├── Readme
│   │   ├── departments.csv
│   │   ├── encounters.csv
│   │   ├── patients.csv
│   │   ├── providers.csv
│   │   └── transactions.csv
│   └── trendytech-hospital-b/
│       ├── Readme
│       ├── departments.csv
│       ├── encounters.csv
│       ├── patients.csv
│       ├── providers.csv
│       └── transactions.csv
├── 4. Gold/
│   ├── dim_cpt_code.py
│   ├── dim_department.py
│   ├── dim_icd_code.ipynb
│   ├── dim_npi.ipynb
│   ├── dim_patient.py
│   ├── dim_provider.py
│   └── fact_transaction.sql
├── datasets/
│   ├── claims/
│   │   ├── hospital1_claim_data.csv
│   │   └── hospital2_claim_data.csv
│   └── cptcodes/
│       └── cptcodes.csv
├── Lookup_file_table_mapping.json
├── audit_table_ddl/
│   └── [SQL DDL files]
├── Gold Queries/
│   └── [SQL query files]
├── Project Notes/
│   └── [Design docs, notebooks]
└── confluence_documentation.md
```

## Deployment & Scheduling

* **ADF Triggers:** schedule or tumbling-window
* **Databricks Jobs:** invoked by ADF for Silver/Gold steps
* **Monitoring:** ADF run history, Databricks job metrics, SQL audit logs

## Best Practices & Enhancements

* Parallelize ForEach for scale
* Secure with Unity Catalog & RBAC
* Configure retries in ADF activities
* Use metadata-driven load\_config.csv
* Enforce consistent naming conventions
* Implement `is_quarantined` flag for data governance

## Contributing

1. Fork this repo.
2. Create a branch: `git checkout -b feature/your-change`.
3. Commit and push changes.
4. Open a Pull Request for review.

