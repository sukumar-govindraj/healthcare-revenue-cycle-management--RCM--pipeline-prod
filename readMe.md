# Healthcare RCM Data Engineering Pipeline

A comprehensive, end-to-end Azure-based data engineering solution for Healthcare Revenue Cycle Management (RCM). This project ingests, processes, and transforms clinical and claims data into analytical fact and dimension tables using a Medallion architecture (Bronze → Silver → Gold) on Azure.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Domain: Revenue Cycle Management (RCM)](#domain-revenue-cycle-management-rcm)
3. [Technology Stack](#technology-stack)
4. [Data Sources](#data-sources)
5. [Architecture](#architecture)

   * [Medallion Layers](#medallion-layers)
6. [Datasets & Configurations](#datasets--configurations)
7. [Pipeline Components](#pipeline-components)

   * [Linked Services & Datasets](#linked-services--datasets)
   * [Activities & Control Flow](#activities--control-flow)
   * [Audit & Incremental Loads](#audit--incremental-loads)
8. [Deployment & Scheduling](#deployment--scheduling)
9. [Best Practices & Enhancements](#best-practices--enhancements)
10. [Contributing](#contributing)
11. [License](#license)

---

## Project Overview

This project implements a scalable data pipeline for Healthcare RCM, automating ingestion of EMR, claims, NPI/ICD/CPT data into Azure Data Lake, transforming it through a multi-layer Medallion architecture, and producing gold‑level fact and dimension tables for reporting and analytics.

## Domain: Revenue Cycle Management (RCM)

RCM covers all financial processes from patient registration through final payment:

* **Patient Visit:** capture patient and insurance details.

  * Example: \$20,000 total charges → \$15,000 insurance, \$5,000 patient.
* **Services & Billing:** generate invoices.
* **Claims Review:** insurance accepts/rejects/partially pays.
* **Payments & Follow‑up:** collect patient balances.
* **Tracking & Improvement:** monitor KPIs like Days in AR, % AR > 90 days.

Key AR metrics:

* 93% collectable by 30 days; 85% by 60 days; 73% by 90 days.
* AR > 90 days ratio (e.g., \$200k/\$1M = 20%).
* Days in AR (e.g., \$400k AR over 40 days → \$10k/day).

## Technology Stack

* **Azure Data Factory (ADF):** orchestration and ingestion
* **Azure Databricks:** data processing & transformation
* **Azure Data Lake Storage Gen2:** raw (bronze), curated (silver), and serving (gold)
* **Azure SQL Database:** source EMR and audit logs
* **Azure Key Vault:** secure credential management
* **Unity Catalog:** governance (optional)

## Data Sources

| Source             | Type             | Ingestion Rate           |
| ------------------ | ---------------- | ------------------------ |
| EMR (Azure SQL DB) | Structured table | Continuous / Incremental |
| Claims (CSV)       | Flat files       | Monthly                  |
| NPI (Public API)   | REST API         | On-demand                |
| ICD Codes (Public) | REST API         | On-demand                |
| CPT Codes (CSV)    | Flat files       | Monthly                  |

## Architecture

### Medallion Layers

1. **Bronze (Raw Parquet)**

   * Landing zone for all sources in Parquet format.
2. **Silver (Delta + CDM + SCD2)**

   * Cleaned, enriched data in a Common Data Model.
   * Slowly Changing Dimensions Type 2 for patient, encounter, transactions.
3. **Gold (Aggregations)**

   * Business-friendly fact and dimension tables, filtered (`is_current=true`, `is_quarantined=false`).

## Datasets & Configurations

* **Landing**: raw file ingestions (CSV → Parquet)
* \*\*Config/

  * `load_config.csv`: defines database, schema, table, load type (Full/Incremental), watermark, target path.

  ```csv
  database,datasource,tablename,loadtype,watermark,is_active,targetpath
  trendytech-hospital-a,hos-a,dbo.encounters,Incremental,ModifiedDate,0,hosa
  ...
  ```
* **Audit Table** (Delta on SQL DB): logs loads

  ```sql
  CREATE TABLE IF NOT EXISTS audit.load_logs (
    id BIGINT IDENTITY,
    data_source STRING,
    tablename STRING,
    numberofrowscopied INT,
    watermarkcolumnname STRING,
    loaddate TIMESTAMP
  );
  ```

## Pipeline Components

### Linked Services & Datasets

| Linked Service       | Dataset Type        | Purpose                        |
| -------------------- | ------------------- | ------------------------------ |
| Azure SQL DB         | Table               | EMR source                     |
| ADLS Gen2            | Delimited & Parquet | Bronze landing & curated files |
| Delta Lake           | Databricks Delta    | Silver + Gold tables           |
| Key Vault            | Secrets             | Credential retrieval           |
| Databricks Workspace | Notebook/pipeline   | Transformation logic           |

### Activities & Control Flow

1. **Lookup**: read `load_config.csv` settings
2. **ForEach**: iterate over config entries in parallel or sequential
3. **Conditional Move**: if target Parquet exists in Bronze, archive by date
4. **Copy**: Full vs. Incremental load from SQL → Bronze
5. **Log**: write row counts and timestamps to `audit.load_logs`
6. **Transform**: Databricks notebooks for Silver (cleaning, SCD2, CDM)
7. **Aggregate**: build Gold fact/dimension tables

### Audit & Incremental Loads

* **Full Load**: `SELECT * FROM table` → Bronze
* **Incremental Load**: query watermark column > last audit date

  ```sql
  SELECT * FROM table
   WHERE ModifiedDate >= '<last_fetched_date>'
  ```
* **Archive Logic**: move Bronze file to `bronze/<path>/archive/YYYY/MM/DD/`
* **Logging**: store counts via ADF activity expressions

## Deployment & Scheduling

* **ADF Trigger**: Tumbling-window or schedule (e.g., daily at midnight)
* **Databricks Jobs**: triggered by ADF for silver/gold transformations
* **Monitoring**: ADF pipeline runs, Databricks job metrics, SQL audit logs

## Best Practices & Enhancements

* **Parallelization**: convert sequential ForEach to parallel for scale
* **Security**: integrate Unity Catalog and enforce RBAC
* **Retry Logic**: configure activity retries in ADF
* **Config‑driven**: use metadata files for dynamic pipelines
* **Naming Conventions**: consistent container/folder/table names
* **Governance**: implement data quarantining (`is_quarantined` flag)

## Contributing

1. Fork this repository
2. Create a feature branch: `git checkout -b feature/your-change`
3. Commit and push changes
4. Open a Pull Request for review

