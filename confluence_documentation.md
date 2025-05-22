h1. Healthcare RCM Data Engineering Pipeline

{toc\:printable=true|maxLevel=4}

h2. Project Overview
This project automates the end-to-end data flow for Healthcare Revenue Cycle Management (RCM) on Azure, from ingestion of EMR and claims data through a Medallion architecture (Bronze → Silver → Gold) to produce analytical fact and dimension tables.

h2. Domain: Revenue Cycle Management (RCM)
RCM manages the financial lifecycle of patient services:

* Capture patient & insurance details (e.g., \$20k bill → \$15k insurance + \$5k patient)
* Generate invoices and submit claims
* Insurance adjudication (full, partial, or denied)
* Patient balance follow-up and collections
* KPI tracking (Days in AR, % AR > 90 days)

h2. Technology Stack
||Layer||Service/Tool||Purpose||
\| Orchestration | Azure Data Factory | Ingest & orchestrate pipelines |
\| Processing    | Azure Databricks   | Transform (Spark + Delta Lake) |
\| Storage       | ADLS Gen2          | Bronze/Silver/Gold layers       |
\| Source DB     | Azure SQL DB       | EMR & audit logs                |
\| Credentials   | Azure Key Vault    | Secure secrets management       |
\| Governance    | Unity Catalog (opt)| Table/catalog control           |

h2. Data Sources
||Source||Type||Frequency||
\| EMR                | Azure SQL table | Incremental/continuous |
\| Claims             | CSV flat files  | Monthly               |
\| ICD & NPI          | REST APIs       | On-demand             |
\| CPT codes          | CSV flat files  | Monthly               |

h2. Architecture
h3. Medallion Layers

* *Bronze:* raw Parquet files landing in ADLS Gen2
* *Silver:* cleaned, conformed Delta tables (Common Data Model + SCD2)
* *Gold:* aggregated fact and dimension tables (filtered by is\_current=true, is\_quarantined=false)

h2. Configuration & Mapping

* *load\_config.csv* defines pipeline metadata:
  {code\:csv}
  database,datasource,tablename,loadtype,watermark,is\_active,targetpath
  trendytech-hospital-a,hos-a,dbo.encounters,Incremental,ModifiedDate,0,hosa
  ...  (10 entries total)
  {code}
* *Lookup\_file\_table\_mapping.json*: maps landing file names to target tables
* *audit\_table\_ddl/*: SQL DDL for creating `audit.load_logs`
  {code\:sql}
  CREATE TABLE IF NOT EXISTS audit.load\_logs (
  id BIGINT IDENTITY,
  data\_source STRING,
  tablename STRING,
  numberofrowscopied INT,
  watermarkcolumnname STRING,
  loaddate TIMESTAMP
  );
  {code}

h2. Pipeline Components
h3. 1. Set up

* `audit_ddl.py` — create audit table schema in Azure SQL
* `adls_mount.py` — mount ADLS Gen2 in Databricks workspace

h3. 2. API Extracts

* `ICD Code API extract.ipynb` — pull ICD codes via public API
* `NPI API extract.ipynb` — pull NPI data via public API

h3. 3. Silver Transformations

* Python scripts: `Claims.py`, `CPT codes.py`, `Departments_F.py`, `Encounters.py`, `Patient.py`, `Providers_F.py`, `Transactions.py`
* Notebooks: `ICD Code.ipynb`, `NPI.ipynb`, `data_generator_faker_module.ipynb`

h3. 4. EMR Sample Data

* Folders: `trendytech-hospital-a/`, `trendytech-hospital-b/` each with CSVs for `departments`, `encounters`, `patients`, `providers`, `transactions`

h3. 5. Gold Transformations

* Dimension scripts: `dim_cpt_code.py`, `dim_department.py`, `dim_patient.py`, `dim_provider.py`
* Notebooks: `dim_icd_code.ipynb`, `dim_npi.ipynb`
* Fact table: `fact_transaction.sql`

h3. 6. Datasets

* `datasets/claims/`: `hospital1_claim_data.csv`, `hospital2_claim_data.csv`
* `datasets/cptcodes/`: `cptcodes.csv`

h3. 7. Lookup & Audit

* `Lookup_file_table_mapping.json` for landing-to-target mapping
* `audit_table_ddl/` contains SQL scripts for audit table setup

h2. File Structure
{code\:text}
Trendytech-Azure-Project-main/
├── 1. Set up/
│   ├── audit\_ddl.py
│   └── adls\_mount.py
├── 2. API extracts/
│   ├── ICD Code API extract.ipynb
│   └── NPI API extract.ipynb
├── 3. Silver/
│   ├── CPT codes.py
│   ├── Claims.py
│   ├── Departments\_F.py
│   ├── Encounters.py
│   ├── ICD Code.ipynb
│   ├── NPI.ipynb
│   ├── Patient.py
│   ├── Providers\_F.py
│   ├── Transactions.py
│   ├── data\_generator\_faker\_module.ipynb
│   └── load\_config.csv
├── EMR/
│   ├── trendytech-hospital-a/
│   └── trendytech-hospital-b/
├── 4. Gold/
│   ├── dim\_cpt\_code.py
│   ├── dim\_department.py
│   ├── dim\_icd\_code.ipynb
│   ├── dim\_npi.ipynb
│   ├── dim\_patient.py
│   ├── dim\_provider.py
│   └── fact\_transaction.sql
├── datasets/
│   ├── claims/
│   └── cptcodes/
├── Lookup\_file\_table\_mapping.json
├── audit\_table\_ddl/
├── Gold Queries/
└── confluence\_documentation.md
{code}

h2. Deployment & Scheduling

* ADF triggers (tumbling-window or scheduled runs)
* Databricks jobs invoked by ADF for Silver & Gold transformations
* Monitoring via ADF run logs, Databricks job metrics, SQL audit logs

h2. Best Practices & Enhancements

* Parallelize ForEach loops for scale
* Enforce metadata-driven config (`load_config.csv`)
* Implement retries and error notifications in ADF
* Secure data via Unity Catalog & role-based access
* Adopt consistent naming conventions
* Introduce data quarantining (`is_quarantined` flag) in Silver

h2. Contributing

# Fork repository

# Create feature branch (`feature/your-change`)

# Commit & push

# Open a Pull Request


