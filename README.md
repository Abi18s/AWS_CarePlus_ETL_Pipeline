# AWS_CarePlus_ETL_Pipeline
# AWS End-to-End ETL & Analytics Pipeline — Support Tickets & Logs

An end-to-end, event-driven data engineering and analytics pipeline on AWS that ingests support ticket data from MySQL and raw support logs from a local system, transforms them using AWS Lambda and AWS Glue, stores the processed output as Parquet in S3, performs ad-hoc analysis with Amazon Athena, loads curated data into Amazon Redshift for warehousing, and visualizes insights through a Power BI dashboard.

🔗 **[Live Dashboard](<PASTE_YOUR_LIVE_DASHBOARD_LINK_HERE>)**

## Architecture

```
                         ┌─────────────────────┐
   MySQL DB (support_tickets)  ─────────────►   │                     │
                                                 │   S3 (raw/)         │
   Local device (support_logs) ─────────────►   │                     │
                         └──────────┬──────────┘
                                    │
                     S3 Event Notification (PUT)
                                    │
                 ┌──────────────────┴──────────────────┐
                 ▼                                      ▼
        AWS Lambda                              AWS Lambda (trigger)
   (transforms support_logs)                     (triggers Glue job)
                 │                                      │
                 │                                      ▼
                 │                              AWS Glue Visual ETL
                 │                          (transforms support_tickets)
                 ▼                                      ▼
        ┌─────────────────────────────────────────────────┐
        │              S3 (processed/) — Parquet            │
        └──────────────────────┬──────────────────────────┘
                                │
                                ▼
                    Amazon Athena (ad-hoc SQL analysis)
                                │
                                ▼
                       Amazon Redshift (data warehouse)
                                │
                                ▼
                    Power BI (dashboard & reporting)
```

## Overview

This project simulates a realistic end-to-end data engineering and analytics workflow: pulling data from two very different sources (a relational database and local flat files), landing it in a data lake, transforming it with two different compute services depending on the workload, running ad-hoc SQL analysis directly on the data lake, promoting curated data into a cloud data warehouse, and surfacing insights through a BI dashboard.

| Stage | Source | Ingestion Method | Transformation | Output |
|---|---|---|---|---|
| Support Tickets | MySQL DB (`support_tickets.csv` export) | Python script / MySQL connector → S3 | AWS Glue Visual ETL | Parquet in `processed/` |
| Support Logs | Local device (`.log` files) | Manual/scripted upload → S3 | AWS Lambda (regex parsing + `pandas`) | Parquet in `processed/` |
| Analysis | S3 `processed/` (Parquet) | Amazon Athena | Ad-hoc SQL queries | Query results / curated tables |
| Warehousing | Athena query output | Load into Redshift | Table design & load | Redshift tables |
| Reporting | Amazon Redshift | Power BI (Redshift connector) | Data modeling & DAX | Interactive dashboard |

## Data Flow

### 1. Ingestion → S3 `raw/`
- **support_tickets.csv**: Exported/extracted from a MySQL database and uploaded to `s3://<bucket>/support_tickets/raw/`
- **support_logs**: Semi-structured application log files (custom log format with fields like `TicketID`, `SessionID`, `ResponseTime`, `CPU`, `EventType`, etc.) uploaded from a local machine to `s3://<bucket>/support_logs/raw/`

### 2. Transformation

**Support Logs — AWS Lambda**
An S3 `PUT` event on `support_logs/raw/` triggers a Lambda function that:
- Reads the raw log file from S3
- Parses each log entry using regex to extract structured fields
- Cleans the data:
  - Drops unused columns (`trace_id`)
  - Removes invalid records (e.g., negative response times)
  - Fixes inconsistent categorical values (typo correction in `log_level`)
  - Removes duplicate rows
  - Casts columns to appropriate data types (`int`, `float`, `bool`, `datetime`)
- Writes the cleaned DataFrame to `s3://<bucket>/support_logs/processed/` in Parquet format using **AWS SDK for pandas (awswrangler)**

**Support Tickets — AWS Glue Visual ETL**
A second Lambda function, triggered on `support_tickets/raw/` uploads, starts a Glue job via `start_job_run()`, passing the S3 file path as a job argument. The Glue Visual ETL job:
- Reads the raw CSV from S3
- Applies transformations (schema mapping, type casting, filtering) via Glue Studio's visual nodes
- Writes the result to `s3://<bucket>/support_tickets/processed/` in Parquet format

### 3. Ad-hoc Analysis — Amazon Athena
External tables are defined over both `processed/` folders, allowing SQL queries directly on the Parquet output without any additional data movement. This layer was used to:
- Explore and validate the transformed data (row counts, null checks, distribution of `error`/`log_level` values, response time trends)
- Prototype the aggregations and joins later used to populate Redshift
- Answer one-off analytical questions without provisioning a warehouse or moving data

### 4. Data Warehousing — Amazon Redshift
Curated/aggregated results from Athena were loaded into **Amazon Redshift** for structured, performant data warehousing:
- Redshift tables were designed to support repeatable reporting queries (fact/dimension-style structure for tickets and logs)
- Data was loaded from S3 into Redshift (e.g., via `COPY` from the `processed/` Parquet files)
- Redshift serves as the single source of truth for downstream BI reporting, separate from the ad-hoc Athena layer

### 5. Visualization — Power BI
Power BI connects directly to Amazon Redshift to build the final reporting layer:
- Data model built on top of the Redshift tables
- Dashboard visuals covering key insights such as ticket volume trends, error rates, response time distribution, and log severity breakdown by component
- Enables business/stakeholder-facing exploration without needing direct database or SQL access

🔗 **[View Live Dashboard](<PASTE_YOUR_LIVE_DASHBOARD_LINK_HERE>)**

## Tech Stack

- **Amazon S3** — data lake storage (raw + processed layers)
- **AWS Lambda** (Python) — event-driven transformation for log files; Glue job trigger for ticket files
- **AWS Glue** (Visual ETL) — schema-based transformation for structured ticket data
- **AWS SDK for pandas (awswrangler)** — Parquet read/write and S3 integration inside Lambda
- **Amazon Athena** — serverless ad-hoc SQL querying over Parquet data
- **Amazon Redshift** — cloud data warehouse for curated, reportable data
- **Power BI** — dashboarding and business intelligence, connected directly to Redshift
- **MySQL** — source database for support ticket records
- **pandas / re (regex)** — log parsing and data cleaning

## Repository Structure

```
.
├── lambda/
│   ├── log_transform_lambda.py       # Parses & cleans support_logs, writes Parquet
│   └── glue_trigger_lambda.py        # Triggers Glue job on new ticket file upload
├── glue/
│   └── support_tickets_etl.py        # Glue Visual ETL generated script
├── athena/
│   ├── create_table_logs.sql
│   └── create_table_tickets.sql
├── redshift/
│   ├── create_tables.sql          # Redshift table DDL
│   └── copy_from_s3.sql           # COPY commands to load Parquet into Redshift
├── powerbi/
│   └── support_dashboard.pbix     # Power BI dashboard file
├── sample_data/
│   ├── support_tickets_sample.csv
│   └── support_logs_sample.log
└── README.md
```

## Setup

### Prerequisites
- AWS account with permissions for S3, Lambda, Glue, and Athena
- An S3 bucket with `raw/` and `processed/` prefixes for both data types
- MySQL database access (for ticket data export)

### Deployment Steps
1. Create the S3 bucket and folder structure (`support_tickets/raw`, `support_tickets/processed`, `support_logs/raw`, `support_logs/processed`)
2. Deploy `log_transform_lambda.py` as a Lambda function
   - Attach the **AWS SDK for pandas (awswrangler)** layer matching your Lambda's Python runtime and architecture
   - Set memory to at least 512 MB
3. Deploy `glue_trigger_lambda.py` as a second Lambda function
   - Grant the execution role `glue:StartJobRun` permission
4. Configure S3 event notifications:
   - `support_logs/raw/` → triggers `log_transform_lambda`
   - `support_tickets/raw/` → triggers `glue_trigger_lambda`
5. Create the Glue Visual ETL job (`support_tickets_etl.py`) and parameterize the input/output S3 paths
6. Run the Athena `CREATE EXTERNAL TABLE` scripts in `athena/` to register both processed datasets
7. Use Athena to run ad-hoc queries and validate/explore the processed data
8. Provision a Redshift cluster or Serverless workgroup, then run `redshift/create_tables.sql` to define the warehouse schema
9. Load data into Redshift using `redshift/copy_from_s3.sql` (`COPY` command from the S3 `processed/` Parquet files)
10. Connect Power BI to Redshift using the built-in Amazon Redshift connector and build/refresh the dashboard (`powerbi/support_dashboard.pbix`)

## Challenges & Debugging Highlights

- **Diagnosed a native-library segmentation fault in Lambda.** The log-transformation Lambda intermittently crashed with `signal: segmentation fault` during pandas datetime parsing. Through systematic isolation (testing imports, DataFrame creation, cleaning steps, and type casting independently), the issue was traced to an incompatibility between `pandas`/`numpy`'s datetime parsing internals and the Python 3.12 build of the AWS SDK for pandas Lambda layer. Resolved by pinning the Lambda runtime and layer to Python 3.11.
- **Resolved a layer/package conflict** between manually-imported `pyarrow` and the awswrangler-provided version, which was causing duplicate native library loads.
- **Right-sized Lambda memory** after ruling out both out-of-memory and architecture-mismatch causes for a separate crash, using CloudWatch's `Max Memory Used` and `Init Duration` metrics to narrow down root cause.

## Future Improvements

- Add Infrastructure as Code (Terraform/CloudFormation) for reproducible deployment
- Add data quality validation before writing to `processed/`
- Partition Athena tables by date for query performance
- Automate MySQL → S3 ingestion with a scheduled Glue or Lambda job instead of manual export
- Automate the Redshift load step (e.g., scheduled Glue job or Lambda-triggered `COPY`) instead of manual loading
- Set up incremental/scheduled refresh for the Power BI dashboard instead of manual refresh
- Explore Redshift Spectrum or Zero-ETL integration to reduce duplicate storage between S3 and Redshift

## Author

Feel free to reach out with questions or suggestions for improving this pipeline.
