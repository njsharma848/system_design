# Data Engineering System Design: 7 Projects You Should Build in 2026

![Cover](assets/cover.jpg)

---

## Table of Contents

1. [RideStream: Real-Time Ride Analytics Lakehouse](#1-ridestream-real-time-ride-analytics-lakehouse)
2. [Real-Time Air Quality Index (AQI) Tracking Platform](#2-real-time-air-quality-index-aqi-tracking-platform)
3. [Real-Time Stock Data Pipeline with Kafka & Spark](#3-real-time-stock-data-pipeline-with-kafka--spark)
4. [Spotify Data Pipeline with Glue, Snowflake & Power BI](#4-spotify-data-pipeline-with-glue-snowflake--power-bi)
5. [Crypto Data Pipeline on Google Cloud & BigQuery](#5-crypto-data-pipeline-on-google-cloud--bigquery)
6. [Azure Data Pipeline with Databricks & Synapse](#6-azure-data-pipeline-with-databricks--synapse)
7. [Food Order ETL Pipeline with MySQL & Power BI](#7-food-order-etl-pipeline-with-mysql--power-bi)
8. [Cross-Project Tool Comparison](#cross-project-tool-comparison)

---

## Preface: How to Read This Document

Each project section answers three questions:
1. **What does the architecture look like?** (original diagram from the PDF + annotated ASCII flow)
2. **What tools are used?** (the tech stack)
3. **WHY those tools?** (the reasoning — the part most guides skip)

The "why" column is where real engineering decisions live. Anyone can list tools. Understanding *when* to use Kafka vs Kinesis, or *why* you'd pick Snowflake over BigQuery, is what separates a practitioner from someone who followed a tutorial.

---

## 1. RideStream: Real-Time Ride Analytics Lakehouse

**Cloud:** AWS | **Difficulty:** Advanced | **Type:** Streaming + Lakehouse

### What It Does

A production-grade platform ingesting ride-booking events (think Uber/Ola) in real time, landing them into a tiered data lake, transforming them with Spark, and making them queryable via Athena.

### Architecture Diagram

![RideStream Architecture](assets/project_1_diagram.jpg)

*The diagram shows the full flow: events originate from a website/app, stream through MSK → Kinesis Firehose → S3 (Raw/Refined/Business layers). Step Functions automates the orchestration of EMR, which feeds dbt and the Glue catalog, all queryable via Athena. The right panel shows the deployment stack: CloudFormation, CodePipeline, and CodeBuild.*

### Annotated Flow

```
[Website / App Events]
         │
         ▼
   ┌─────────────┐
   │  Amazon MSK │  ← Managed Kafka: streams ride events per topic
   │  (Kafka)    │    (e.g., ride_requested, ride_completed)
   └──────┬──────┘
          │
          ▼
   ┌──────────────────┐
   │  Kinesis Firehose│  ← Buffers and batches stream → S3
   └──────┬───────────┘
          │
          ▼
   ┌────────────────────────────────────┐
   │         S3 Data Lake               │
   │  ┌──────────┐  ┌────────┐  ┌────┐ │
   │  │  Bronze  │→ │ Silver │→ │Gold│ │  ← Medallion layers
   │  │  (Raw)   │  │(Clean) │  │(Agg│ │
   │  └──────────┘  └────────┘  └────┘ │
   └──────────────────┬─────────────────┘
                      │
          ┌───────────┴──────────┐
          ▼                      ▼
   ┌─────────────┐        ┌───────────┐
   │  AWS Glue   │        │  Amazon   │
   │  (Catalog)  │        │  Athena   │  ← SQL queries over S3
   └─────────────┘        └───────────┘
          │
          ▼
   ┌─────────────┐
   │  EMR + Spark│  ← Heavy transformation jobs
   └──────┬──────┘
          │
          ▼
   ┌─────────────┐
   │     dbt     │  ← Data modeling layer (SQL-based transforms)
   └─────────────┘

   Orchestration: AWS Step Functions
   Infrastructure: CloudFormation (IaC)
   CI/CD: AWS CodePipeline + CodeBuild
```

### Tool-by-Tool Reasoning

| Tool | Why This Tool? |
|------|----------------|
| **Amazon MSK (Managed Kafka)** | Ride events are high-throughput and require ordering guarantees per partition (e.g., all events for ride #1234 in sequence). Kafka's log-based architecture handles this natively. MSK removes the operational burden of managing Kafka clusters yourself — no ZooKeeper management, patching, or scaling headaches. |
| **Kinesis Data Firehose** | You need to bridge the stream to S3 reliably. Firehose is the simplest "Kafka → S3" connector in AWS — it buffers records, compresses them, and lands them in S3 without you writing a consumer. Alternative (writing a custom Kafka consumer) is more flexible but requires more code and failure handling. |
| **S3 (Medallion Architecture)** | Object storage is cheap and infinitely scalable. The Bronze/Silver/Gold pattern solves a real problem: raw data is messy, but you can't delete it (you may need to reprocess). Bronze = raw as-is. Silver = cleaned, deduplicated. Gold = aggregated, business-ready. This tiering lets you replay transformations without re-ingestion. |
| **AWS Glue (Data Catalog)** | Without a catalog, S3 is just a pile of files. Glue Crawlers scan your S3 data, infer schemas, and register them in a central metastore — making data discoverable to Athena, EMR, and Spark without hardcoding schemas in every job. |
| **EMR + Spark** | For large-scale transformation jobs (joining millions of ride records against user profiles, computing surge pricing windows), you need distributed compute. Spark on EMR gives you cluster-level parallelism. Lambda would time out; Glue ETL is a simpler Spark wrapper that works for moderate workloads, but EMR gives you full Spark control for complex joins. |
| **dbt** | After Spark does heavy lifting, dbt handles the final modeling layer — turning Silver/Gold tables into clean, documented, tested business models. It works with SQL, integrates with Athena/Redshift/Snowflake, and brings software engineering practices (version control, tests, docs) to data transformation. |
| **Athena** | Serverless SQL over S3. No cluster to spin up, pay per query. Perfect for ad-hoc analysis and BI tool connections on top of your Gold layer. It's not ideal for sub-second dashboards (use Redshift for that), but for analytical queries it's cost-effective. |
| **Step Functions** | Orchestrates the multi-step pipeline (trigger Glue crawler → run EMR job → run dbt → notify). More AWS-native than Airflow and integrates tightly with Lambda, EMR, and Glue via service integrations. |
| **CloudFormation** | Infrastructure-as-code ensures reproducibility. You can tear down and recreate the entire pipeline in a different AWS region with one command. Avoids "it works in my account" problems. |

### Key Architectural Decisions

**Why MSK instead of Kinesis for ingestion?**
Kafka (MSK) is the industry standard for event streaming with existing tooling ecosystems (Kafka Connect, Schema Registry, consumer groups). Kinesis is simpler and more AWS-native but has less ecosystem support and a 7-day maximum retention. If your team knows Kafka, MSK is the right call.

**Why not just use Glue ETL instead of EMR?**
Glue ETL is a managed Spark wrapper — easier to set up but less configurable. For a production ride platform, you need control over Spark configurations (partition tuning, broadcast joins, caching strategies). EMR gives you that control at the cost of more operational overhead.

---

## 2. Real-Time Air Quality Index (AQI) Tracking Platform

**Cloud:** AWS | **Difficulty:** Intermediate | **Type:** IoT Streaming + Alerting

### What It Does

Ingests real-time AQI sensor data, processes it, triggers alerts when thresholds are breached, and visualizes trends on a live dashboard. Covers the full IoT analytics pattern.

### Architecture Diagram

![AQI Platform Architecture](assets/project_2_diagram.jpg)

*The diagram shows the dual-path design: a real-time path (Boto3 producer → Kinesis Stream → Kinesis Analytics → Stats Stream → Lambda → SNS + CloudWatch → Grafana) and a batch path (Kinesis Firehose → S3 Raw Data Lake → Glue Crawler → S3 Analytical). Both paths converge to provide real-time alerting AND historical analysis from the same ingestion source.*

### Annotated Flow

```
[AQI Sensors / Boto3 Producer]
           │
           ├─────────────────────────┐
           ▼                         ▼
   ┌──────────────┐         ┌─────────────────┐
   │  Kinesis     │         │  Kinesis         │
   │  Data Stream │         │  Firehose (Batch)│
   │  (AQI Stream)│         └────────┬─────────┘
   └──────┬───────┘                  │
          │                          ▼
          ▼                   ┌─────────────┐
   ┌──────────────┐           │  S3 Raw     │
   │  Kinesis     │           │  Data Lake  │
   │  Analytics   │           └──────┬──────┘
   │  (AQI        │                  │
   │  Analysis)   │           ┌──────┴──────┐
   └──────┬───────┘           │ Glue Crawler│
          │                   └──────┬──────┘
          ▼                          │
   ┌──────────────┐           ┌──────▼──────┐
   │  Stats Stream│           │   S3        │
   └──────┬───────┘           │  Analytical │
          │                   └──────┬──────┘
     ┌────┴──────┐                   ▼
     ▼           ▼            ┌─────────────┐
  ┌──────┐  ┌──────────┐      │   Athena    │
  │  SNS │  │CloudWatch│      └─────────────┘
  │Alert)│  │(Metrics) │
  └──────┘  └──────┬───┘
                   │
                   ▼
            ┌─────────────┐
            │   Grafana   │  ← Live dashboard
            └─────────────┘
```

### Tool-by-Tool Reasoning

| Tool | Why This Tool? |
|------|----------------|
| **Kinesis Data Streams** | Unlike Kafka/MSK, Kinesis is fully managed with no cluster to provision. For IoT sensor data at moderate throughput, Kinesis is simpler and cheaper. You pay per shard/hour rather than per EC2 instance. AQI readings don't need Kafka's advanced features (compaction, consumer groups at scale). |
| **AWS Lambda** | AQI processing is event-driven: "when a new reading arrives, check if it exceeds threshold." Lambda is perfect — stateless, scales to zero, triggers directly from Kinesis. You don't need a running Spark cluster for this logic; a 50-line Python function suffices. |
| **SNS (Simple Notification Service)** | Alerting is a pub/sub problem. SNS lets you notify multiple subscribers (email, SMS, Slack via HTTP endpoint, other Lambda functions) with one publish. Adding a new alert channel is just adding a subscription — no pipeline changes needed. |
| **CloudWatch** | Native AWS monitoring. Capturing metrics like "average AQI per city over 5 minutes" can go directly into CloudWatch as custom metrics, which then feed Grafana without extra infrastructure. |
| **Grafana** | Purpose-built for time-series visualization. Connects to CloudWatch and Athena natively. For an AQI dashboard (line charts, gauges, thresholds), Grafana's templates are far more capable than building a custom React dashboard. |
| **Glue Crawlers** | Auto-discover schema from S3 files as they arrive. Since sensor data format might evolve (new sensors, new fields), Crawlers handle schema evolution without manual intervention. |

### Key Architectural Decisions

**Why dual-path (Kinesis Streams + Firehose)?**
Streams handles real-time processing (Lambda trigger for alerting). Firehose handles batch delivery to S3 for historical analysis. These are different latency requirements: alerting needs <1 second, historical analysis is fine with 5-minute batches. Separating the paths avoids blocking real-time alerts while waiting for S3 writes.

**Why Lambda instead of Spark for processing?**
AQI threshold checks are simple: `if pm2_5 > 150: trigger_alert()`. This is a stateless, per-record operation. Spark would be massive overkill — spinning up a cluster for logic that runs in milliseconds. Lambda's cold start (~200ms) is acceptable for alerting use cases where "near real-time" is sufficient.

---

## 3. Real-Time Stock Data Pipeline with Kafka & Spark

**Cloud:** Self-hosted (Docker) + Snowflake | **Difficulty:** Intermediate | **Type:** Open-Source Streaming

### What It Does

Captures live stock market data from Yahoo Finance and Alpha Vantage, streams through Kafka, processes with Spark Streaming (real-time) and Spark batch workers, stores in MinIO (S3-compatible), and loads to Snowflake for analysis.

### Architecture Diagram

![Stock Pipeline Architecture](assets/project_3_diagram.jpg)

*The diagram highlights two ingestion modes — Batch Ingestion and Stream Ingestion — both feeding into Kafka (coordinated by ZooKeeper). Spark Streaming handles the real-time path while multiple Spark Workers handle batch processing. All output lands in MinIO (3 buckets: Raw CSV, RealTime Parquet, Processed Parquet) before loading into Snowflake. Apache Airflow and PostgreSQL sit at the top as orchestration and metadata store, all wrapped in Docker.*

### Annotated Flow

```
┌─────────────────────────┐
│     Data Sources        │
│  Yahoo Finance API      │
│  Alpha Vantage API      │
└────────────┬────────────┘
             │
    ┌────────┴─────────┐
    ▼                  ▼
[Batch Ingestion]  [Stream Ingestion]
    │                  │
    └────────┬─────────┘
             ▼
   ┌─────────────────────┐
   │    Apache Kafka     │
   │  (+ ZooKeeper)      │
   │  Topics:            │
   │  - stocks_raw       │
   │  - stocks_processed │
   └──────────┬──────────┘
              │
    ┌─────────┴──────────┐
    ▼                    ▼
┌──────────────┐   ┌───────────────────┐
│  Spark       │   │  Spark            │
│  Streaming   │   │  Batch Workers    │
│  (real-time) │   │  (Worker x4)      │
└──────┬───────┘   └────────┬──────────┘
       └──────────┬──────────┘
                  ▼
   ┌──────────────────────────┐
   │         MinIO            │
   │  Raw (CSV) │ RT Parquet  │
   │  Processed Parquet       │
   └─────────────┬────────────┘
                 ▼
        ┌─────────────────┐
        │    Snowflake    │
        └─────────────────┘

   Orchestration: Apache Airflow
   Containerization: Docker + PostgreSQL (Airflow backend)
```

### Tool-by-Tool Reasoning

| Tool | Why This Tool? |
|------|----------------|
| **Apache Kafka (self-hosted)** | This project deliberately avoids cloud lock-in. Self-hosting Kafka gives you full control over retention, partitioning, and consumer groups. For a portfolio project, it also demonstrates deeper operational knowledge than "I used MSK." ZooKeeper manages cluster state. |
| **Spark Streaming** | Stock data requires windowed aggregations: "average price over last 60 seconds," "volume spike detection." Spark Streaming's micro-batch model handles this elegantly with its `window()` and `watermark()` APIs. Lambda can't maintain state across records; Flink is more powerful but harder to learn. |
| **Spark Batch Workers** | End-of-day processing (daily OHLC calculations, feature engineering for ML) is a batch job. Running multiple Spark workers in parallel processes the day's accumulated data fast enough for overnight batch SLAs. |
| **Airflow** | Orchestrates both the streaming jobs and batch pipelines. The DAG-based model is perfect for: "at market close → trigger batch → after batch → load to Snowflake → after load → run data quality checks." Airflow's retry logic and alerting are essential for production reliability. |
| **MinIO** | S3-compatible object storage that runs locally in Docker. Means your pipeline code uses the same S3 API as AWS — zero code changes if you migrate to the cloud. CSV for raw dumps (human-readable, easy debugging), Parquet for processed data (columnar, 5-10x compression vs CSV, faster Spark reads). |
| **Snowflake** | Cloud-neutral data warehouse (runs on AWS/GCP/Azure). Separates storage from compute — you can scale query engines without scaling storage costs. For stock analytics, Snowflake's columnar storage and query optimizer outperform a traditional RDBMS. |
| **Docker** | Entire infrastructure (Kafka, ZooKeeper, Spark, MinIO, Airflow, PostgreSQL) runs with `docker-compose up`. Reproducible across laptops, CI/CD, and eventually Kubernetes. Eliminates "works on my machine" problems. |

### Key Architectural Decisions

**CSV vs Parquet — when to use which?**
Raw dump → CSV. It's human-readable, easy to debug when something goes wrong. Processed/analytical data → Parquet. Columnar format means queries like "give me all closing prices" read only one column, not every row. 5-10x smaller than CSV, 10-100x faster for analytical queries.

**Why not just use Spark Streaming for everything?**
Streaming handles incremental, stateful processing. Batch handles full historical reprocessing and complex joins that don't fit in a streaming window (e.g., joining today's prices against 2-year historical averages). The two modes are complementary, not redundant.

---

## 4. Spotify Data Pipeline with Glue, Snowflake & Power BI

**Cloud:** AWS + Snowflake | **Difficulty:** Beginner-Intermediate | **Type:** ETL + BI

### What It Does

Extracts data from Spotify's Web API (tracks, artists, play counts), transforms with AWS Glue, loads into Snowflake via Snowpipe (event-driven), and visualizes in Power BI. Covers the complete ETL lifecycle — best beginner-friendly project on the list.

### Architecture Diagram

![Spotify Pipeline Architecture](assets/project_4_diagram.jpg)

*The diagram is organized into three horizontal layers: (1) Extraction — Spotify API → Lambda (data extraction); (2) Storage — two S3 buckets (raw data and transformed data) connected by Glue/Spark, then Snowpipe auto-ingests from S3; (3) Orchestrate — Docker + Apache Airflow managing the workflow. The final output is Snowflake feeding Power BI dashboards.*

### Annotated Flow

```
   ┌─────────────────┐
   │   Spotify API   │
   └────────┬────────┘
            │  HTTP requests (Python / Boto3)
            ▼
   ┌─────────────────┐
   │  AWS Lambda     │  ← Serverless extraction, triggered on schedule
   │  (data extract) │    (EventBridge: every 6 hours)
   └────────┬────────┘
            ▼
   ┌─────────────────┐
   │  Amazon S3      │  ← Raw JSON: artists.json, tracks.json, etc.
   │  (raw data)     │
   └────────┬────────┘
            │  S3 Event Notification → triggers Glue
            ▼
   ┌─────────────────┐
   │  AWS Glue       │  ← Reads JSON, transforms, writes Parquet
   │  (Spark ETL)    │
   └────────┬────────┘
            ▼
   ┌─────────────────┐
   │  Amazon S3      │  ← Cleaned, typed, deduplicated Parquet files
   │  (transformed)  │
   └────────┬────────┘
            │  S3 Event → Snowpipe auto-ingest
            ▼
   ┌─────────────────┐
   │   Snowpipe      │  ← Continuously ingests new S3 files → Snowflake
   └────────┬────────┘
            ▼
   ┌─────────────────┐
   │   Snowflake     │  ← tracks, albums, artists tables
   └────────┬────────┘
            ▼
   ┌─────────────────┐
   │   Power BI      │  ← Top tracks, genre trends, artist growth
   └─────────────────┘

   Orchestration: Apache Airflow (Docker)
```

### Tool-by-Tool Reasoning

| Tool | Why This Tool? |
|------|----------------|
| **AWS Lambda (extraction)** | Spotify API calls are short-lived HTTP requests — perfect for Lambda's stateless, ephemeral model. You're not running 8-hour Spark jobs; you're making API calls and saving JSON. Lambda costs ~$0/month for hourly triggers at this scale vs. keeping an EC2 instance running 24/7. |
| **AWS Glue (Spark ETL)** | Glue provides a managed Spark environment for the transformation layer. Unlike raw EMR, Glue handles cluster provisioning, job bookmarking (so you don't reprocess old files), and schema inference. For moderate data volumes, Glue's serverless Spark is cost-effective. |
| **S3 (dual buckets: raw + transformed)** | Separating raw from transformed storage is a best practice. If your Glue transformation has a bug, you haven't lost the raw data — you can reprocess. The raw bucket is append-only (never modified), the transformed bucket is the "source of truth" for Snowflake. |
| **Snowpipe** | Traditional loading (COPY INTO in Snowflake) requires a scheduled job. Snowpipe is event-driven: when a new Parquet file lands in S3, it automatically loads within minutes. No Airflow DAG needed for the load step — just configure the S3 notification and SQS queue once. |
| **Snowflake** | Spotify analytics involves cross-dimensional queries: "top artists by stream count per genre per month." Snowflake's optimizer handles these well. Its separation of compute and storage means you can run heavy analytical queries without affecting your loading pipeline. |
| **Power BI** | Microsoft's BI tool has a native Snowflake connector, DirectQuery support (live queries instead of imported snapshots), and a familiar interface for business users. Power BI dashboards screenshot well and demonstrate end-to-end delivery in a portfolio. |
| **Airflow (Docker)** | Orchestrates the extraction schedule, monitors job status, and handles retries. Airflow gives you dependency management: "only run Glue after Lambda succeeds; only trigger Snowpipe check after Glue completes." |

### Key Architectural Decisions

**Why Snowpipe instead of scheduled COPY INTO?**
Snowpipe turns loading from a scheduled batch into a near-real-time, event-driven operation. If Spotify releases new data multiple times a day, Snowpipe ensures it's available in your warehouse within minutes rather than waiting for the next scheduled job window.

**Why Glue over Lambda for transformation?**
Lambda has a 15-minute timeout and 10GB memory limit. For transforming a full Spotify dataset (millions of tracks, audio features, playlists), you'd hit those limits. Glue runs as a Spark cluster with no time limit and horizontal scalability. Lambda handles extraction; Glue handles transformation.

---

## 5. Crypto Data Pipeline on Google Cloud & BigQuery

**Cloud:** GCP | **Difficulty:** Intermediate | **Type:** API Ingestion + Warehousing

### What It Does

Pulls live cryptocurrency market data from CoinGecko API, stores in GCP Cloud Storage, warehouses in BigQuery, and visualizes with Looker and Data Studio. The portfolio differentiator — most DE projects use AWS; this one covers GCP.

### Architecture Diagram

![Crypto GCP Architecture](assets/project_5_diagram.jpg)

*The diagram shows the full GCP pipeline organized into five stages: Data Sources (CoinGecko) → Data Ingestion (Cloud Storage: RAW_DATA_STORAGE) → Data Storage (Cloud Storage: TRANSFORMED_DATA_STORAGE) → Data Warehouse (BigQuery: SQL Queries / ANALYZE_DATA) → Applications & Insights (Looker: Embedded Analytics + Data Studio: Dashboards). Cloud Composer (Apache Airflow) spans the bottom as the pipeline orchestrator managing the full API-to-BigQuery pipeline.*

### Annotated Flow

```
   ┌──────────────────────┐
   │    CoinGecko API     │
   │  /coins/markets      │
   │  /coins/{id}/history │
   └──────────┬───────────┘
              ▼
   ┌──────────────────────┐
   │  Cloud Composer      │  ← Managed Airflow, schedules entire pipeline
   │  DAG: crypto_pipeline│
   │  ├── fetch_data      │
   │  ├── validate_data   │
   │  ├── load_to_gcs     │
   │  └── bq_load         │
   └──────────┬───────────┘
              │
     ┌────────┴──────────┐
     ▼                   ▼
┌─────────────┐   ┌─────────────────┐
│  GCS        │   │  GCS            │
│  Raw Data   │   │  Transformed    │
│  (RAW_DATA_ │   │  (TRANSFORMED_  │
│   STORAGE)  │   │   DATA_STORAGE) │
└──────┬──────┘   └────────┬────────┘
       └─────────┬──────────┘
                 ▼
   ┌─────────────────────────┐
   │        BigQuery         │
   │  crypto_raw             │
   │  crypto_transformed     │
   │  crypto_analytics       │
   └──────────┬──────────────┘
              │
    ┌─────────┴──────────┐
    ▼                    ▼
┌─────────────┐   ┌────────────────┐
│   Looker    │   │  Data Studio   │
│  (Embedded) │   │  (Dashboards)  │
└─────────────┘   └────────────────┘
```

### Tool-by-Tool Reasoning

| Tool | Why This Tool? |
|------|----------------|
| **CoinGecko API** | Free tier with no API key required for basic endpoints. Rate limits are reasonable for hourly polling. Provides comprehensive data: prices, volumes, market caps, historical OHLC. Binance API requires account registration and focuses on trading pairs rather than market-wide analytics. |
| **Cloud Composer (managed Airflow)** | GCP's managed Airflow service. Like MSK vs self-hosted Kafka — you're not managing Airflow workers, schedulers, and databases yourself. On GCP, Composer integrates natively with GCS, BigQuery, and Dataflow with pre-built operators (`BigQueryInsertJobOperator`, `GCSToGCSOperator`). |
| **GCP Cloud Storage (GCS)** | GCP's equivalent of S3. Same role: cheap, durable object storage as the raw data lake. Separating raw and transformed storage follows the same immutability principle — raw data is never modified, enabling reruns. |
| **BigQuery** | Google's flagship analytical database. Serverless, columnar, handles petabyte-scale queries in seconds. For crypto analytics ("which coins had the highest 30-day volatility?"), BigQuery's partitioned tables and clustering make time-series queries extremely fast. No cluster sizing decisions — just SQL. |
| **Looker** | Uses a semantic layer (LookML) that defines metrics and dimensions in a reusable way. Instead of every analyst writing their own "market cap" calculation, you define it once in LookML. Consistent metrics across all dashboards. Industry standard at GCP-heavy companies. |
| **Data Studio (Looker Studio)** | Free, shareable, embeddable dashboards. Connects directly to BigQuery. For public-facing reports, Data Studio links can be shared without requiring Looker access. |

### Key Architectural Decisions

**Why BigQuery instead of Snowflake on GCP?**
When you're fully on GCP, BigQuery is the default choice. It's native to the platform, integrates with every GCP service via IAM roles (no separate credentials to manage), and is priced on query cost rather than compute uptime. Snowflake on GCP works but adds another vendor to manage.

**The multi-cloud portfolio argument:**
Projects 1–4 are AWS-heavy. Adding a GCP project to your portfolio demonstrates you understand cloud concepts abstractly — not just "where the button is in the AWS console." Cloud Composer = Airflow, GCS = S3, BigQuery = Redshift/Athena. The concepts transfer; the interfaces differ.

---

## 6. Azure Data Pipeline with Databricks & Synapse

**Cloud:** Azure | **Difficulty:** Intermediate-Advanced | **Type:** Enterprise Lakehouse

### What It Does

An enterprise-style pipeline ingesting e-commerce data (from data.world), flowing through Azure Data Factory into ADLS Gen2, transformed by Databricks with Delta Lake's medallion architecture, and analyzed via Synapse Analytics.

### Architecture Diagram

![Azure Databricks Architecture](assets/project_6_diagram.jpg)

*The diagram is organized into two phases: Ingest and Process. On the Ingest side: E-Com data from data.world → Azure Data Factory (ADF) → Azure Data Lake Storage. On the Process side: Azure Databricks + Apache Spark run the Bronze → Silver → Gold medallion pipeline. The processed Delta Lake files persist back to ADLS, from where Synapse Analytics and BI tools (Power BI, Looker, Tableau) serve the final consumers.*

### Annotated Flow

```
   ┌──────────────────────┐
   │   Data Sources       │
   │  data.world          │
   │  (E-Com Data)        │
   └──────────┬───────────┘
              ▼
   ┌──────────────────────┐
   │  Azure Data Factory  │  ← Orchestrates ingestion (Copy Activity)
   │  (ADF)               │    Triggers on schedule or event
   └──────────┬───────────┘
              ▼
   ┌──────────────────────┐
   │  Azure Data Lake     │
   │  Storage Gen2 (ADLS) │  ← Hierarchical namespace (atomic dir ops)
   └──────────┬───────────┘
              ▼
   ┌──────────────────────────────────────────┐
   │          Azure Databricks                 │
   │  ┌──────────┐  ┌────────┐  ┌──────────┐ │
   │  │  Bronze  │→ │ Silver │→ │  Gold    │ │
   │  │ (Raw     │  │(Cleaned│  │(Business │ │
   │  │ Ingest)  │  │ & Join)│  │ Metrics) │ │
   │  └──────────┘  └────────┘  └──────────┘ │
   │  Engine: Apache Spark                    │
   │  Format: Delta Lake (ACID transactions)  │
   └──────────────────┬───────────────────────┘
                      ▼
   ┌──────────────────────┐
   │  Azure Synapse       │  ← Serverless SQL pools query Delta Lake
   │  Analytics           │    directly; dedicated pools for BI workloads
   └──────────┬───────────┘
              │
    ┌─────────┴──────────────┐
    ▼          ▼             ▼
┌────────┐ ┌────────┐ ┌────────────┐
│Power BI│ │ Looker │ │  Tableau   │
└────────┘ └────────┘ └────────────┘
```

### Tool-by-Tool Reasoning

| Tool | Why This Tool? |
|------|----------------|
| **Azure Data Factory (ADF)** | ADF is the standard ETL orchestrator in Azure, equivalent to AWS Glue workflows or GCP Cloud Composer for data movement. It has 90+ built-in connectors (Salesforce, SAP, on-prem SQL Server, REST APIs) and a visual pipeline designer. For enterprise environments with diverse source systems, ADF's connectivity is hard to beat. |
| **ADLS Gen2** | Azure Data Lake Storage Gen2 is regular Azure Blob Storage with a hierarchical namespace enabled. The hierarchical namespace means you can do atomic directory operations (rename, delete folder) — critical for Spark's job commit protocol. Without it, Spark jobs that rename files during commit can be slow or fail at scale. |
| **Azure Databricks** | Databricks is Spark on Azure, but with a significant difference from raw EMR: it includes a collaborative notebook environment, Delta Lake (ACID transactions on data lakes), MLflow integration, and a photon engine (optimized vectorized query execution). For enterprise data teams with mixed DE + DS roles, Databricks provides a unified platform. |
| **Delta Lake** | Regular Parquet files on ADLS have no transaction support. If your Spark job fails halfway through writing, you have a corrupted partial write. Delta Lake adds ACID transactions, schema enforcement, time travel (query data as it was yesterday), and `MERGE INTO` for upserts — table-stakes features for production data. |
| **Medallion Architecture (Bronze → Silver → Gold)** | Delta Lake makes this pattern more robust: time travel lets you query Bronze data from a week ago even after Gold has been updated, enabling historical debugging. Delta's `OPTIMIZE` and `VACUUM` commands also keep file sizes healthy as data accumulates. |
| **Synapse Analytics** | Combines data warehousing and big data analytics in one service. Serverless SQL pools let you query Delta Lake tables directly from ADLS (no data movement) — great for ad-hoc analysis. Dedicated SQL pools give you traditional DWH performance for heavy BI workloads. |
| **Multi-tool BI (Power BI + Looker + Tableau)** | This project intentionally shows all three. Enterprises often have heterogeneous BI environments (legacy Tableau, newer Power BI rollouts, Looker for embedded analytics). Demonstrating you can connect multiple tools to the same warehouse is a real-world skill. |

### Key Architectural Decisions

**Why Databricks over Azure Synapse Spark pools for transformation?**
Both run Spark. Databricks wins because: (1) Delta Lake is first-class (Delta was created by Databricks); (2) better job scheduling and cluster autoscaling; (3) more mature MLflow and feature store integrations if you want to extend into ML. Synapse Spark is fine for simpler workloads, but Databricks is the enterprise standard.

**Azure's enterprise advantage:**
Fortune 500 companies often run on Azure (Microsoft licensing, Active Directory integration, compliance certifications). Learning Azure Databricks + Synapse directly maps to enterprise job requirements.

---

## 7. Food Order ETL Pipeline with MySQL & Power BI

**Cloud:** Azure / On-Prem (Hybrid) | **Difficulty:** Beginner | **Type:** Traditional DWH + ETL

### What It Does

A production-style traditional ETL pipeline — food orders flow from a staging MySQL database through a business database, migrate to SQL Server via Microsoft Fabric, model into a star schema warehouse, and surface in Power BI. Teaches the fundamentals that underpin ALL modern data warehousing. **Start here if you're new.**

### Architecture Diagram

![Food Order ETL Architecture](assets/project_7_diagram.jpg)

*The diagram shows the full hybrid architecture: two websites send order data into a Stored Procedures ETL layer (handling Insertion, Updation, Deletion with Job Tracking). This feeds a Stage Database (MySQL) and Business Database (MySQL). A cloud migration path runs through Microsoft Fabric & Data Gateway into a SQL Server Business DWH. The star schema is fully visible: DIM_CUSTOMERS, DIM_RESTAURANTS, DIM_DATE, and the central FACT_ORDER_PATTERNS table with all its measures. The final output is an Interactive Reporting & Dashboard in Power BI.*

### Star Schema Detail

```
        DIM_CUSTOMERS              DIM_RESTAURANTS
        ─────────────              ───────────────
        Customer Key (PK)          Restaurant Key (PK)
        Full Name                  Cuisine Type
        Age Group           ┌──────City / State
        Loyalty Tier        │
              │             │
              └──────┬───────┘
                     ▼
             FACT_ORDER_PATTERNS
             ────────────────────
             Customer Key (FK)
             Restaurant Key (FK)
             Date Key (FK)
             Order Count
             Total Spent
             Avg Basket Size
             Preferred Cuisine
             Preferred Order Hour
             Preferred Order Day
             Avg Delivery Distance
             Load Timestamp
                     │
                     └──────────────────┐
                                        ▼
                               DIM_DATE
                               ─────────────
                               Date Key (PK)
                               Full Date
                               Day / Month / Year
                               Day Name
```

### ETL Pipeline Flow

```
[Website A]  [Website B]
     │             │
     └──────┬───────┘
            ▼
  ┌──────────────────────────────┐
  │  Stored Procedures ETL Layer │
  │  INSERT / UPDATE / DELETE    │
  │  + Job Tracking Table        │  ← Records: what ran, when, rows affected
  └──────────────┬───────────────┘
                 │
       ┌─────────┴──────────┐
       ▼                    ▼
┌─────────────┐      ┌─────────────┐
│  Stage DB   │  →   │ Business DB │
│  (MySQL)    │      │  (MySQL)    │
└─────────────┘      └──────┬──────┘
                             │  Cloud Migration
                             ▼
                  ┌──────────────────────┐
                  │  Microsoft Fabric    │
                  │  & Data Gateway      │
                  └──────────┬───────────┘
                             ▼
                  ┌──────────────────────┐
                  │  SQL Server DWH      │
                  │  (Star Schema)       │
                  └──────────┬───────────┘
                             ▼
                  ┌──────────────────────┐
                  │      Power BI        │
                  └──────────────────────┘
```

### Tool-by-Tool Reasoning

| Tool | Why This Tool? |
|------|----------------|
| **MySQL (Staging + Business DB)** | MySQL is the most widely deployed RDBMS in the world. Staging databases are a standard enterprise pattern — raw data lands in staging first (fast writes, no constraints), then ETL validates and moves it to the business database (enforced schema, referential integrity). Two separate databases force you to think about data contracts between systems. |
| **Stored Procedures (ETL)** | Before dbt, Airflow, and Spark, stored procedures were how enterprises did ETL — and many still do. They run inside the database (no network overhead), are transactional (ACID), and are auditable. Job tracking tables (recording what ran, when, how many rows affected) are a production pattern you'll find in legacy systems everywhere. |
| **Microsoft Fabric** | Microsoft's unified analytics platform. The Data Gateway component enables on-premise MySQL to talk to Azure cloud services without exposing databases to the internet. This on-prem-to-cloud migration pattern is ubiquitous in enterprise modernization projects. |
| **SQL Server (Business DWH)** | SQL Server's T-SQL is the lingua franca of enterprise analytics. Its tight Power BI integration, Analysis Services compatibility, and familiar tooling (SSMS) make it the default choice when the organization is Microsoft-stack. |
| **Star Schema** | The dimensional model (Fact table + Dimension tables) is a 30-year-old pattern that still powers most enterprise analytics. It optimizes for READ performance and is intuitive for business users: "show me total sales (fact) by restaurant type (dim) by month (dim)." |
| **Power BI** | Native Microsoft BI tool. Direct connector to SQL Server and Fabric. DAX measures let you define calculations (like year-over-year growth) once and reuse them across all reports. No additional vendor relationships for a Microsoft-stack organization. |

### Key Architectural Decisions

**Why learn "old school" ETL if modern tools exist?**
Because most enterprise data you'll touch was built with these patterns. Being able to read a 10-year-old stored procedure pipeline, understand its job tracking tables, and migrate it to dbt or Airflow is a highly paid skill. The fundamentals — staging → transformation → warehouse, fact/dimension modeling — don't change. The tools change.

**Why a dedicated staging layer?**
Staging is your buffer: raw data lands fast (no validation, no constraints). The ETL layer then validates, deduplicates, and applies business rules before writing to the business database. If source data is malformed, it fails in staging — not in production. Your DWH stays clean.

---

## Cross-Project Tool Comparison

### Streaming Technology Choice

| Scenario | Recommended Tool | Reasoning |
|----------|-----------------|-----------|
| High-throughput, complex consumers, team knows Kafka | Apache Kafka / MSK | Ecosystem, ordering guarantees, consumer groups |
| AWS-native, moderate throughput, simpler ops | Kinesis Data Streams | Fully managed, native AWS integration, lower operational burden |
| Per-record serverless processing | Kinesis + Lambda | Event-driven model, scales to zero, no cluster management |
| Open-source, no cloud vendor, full control | Self-hosted Kafka (Docker) | Portfolio demonstration, deep operational knowledge |

### Transformation Layer Choice

| Data Volume | Complexity | Recommended Tool |
|------------|------------|-----------------|
| Small (<1GB), simple logic | Lambda / Cloud Functions | Stateless, fast, cheap |
| Medium (1GB–100GB), moderate | AWS Glue ETL / Glue Spark | Managed Spark, job bookmarking, schema inference |
| Large (100GB+), complex joins | EMR Spark / Databricks | Full Spark control, cluster tuning, Delta Lake |
| SQL-based modeling on top of Spark | dbt | Software engineering practices on top of warehouse |
| Legacy, on-prem, ACID required | Stored Procedures | Transactional, auditable, no external dependencies |

### Warehouse Choice by Cloud

| Primary Cloud | Recommended Warehouse | When to Use Cross-Cloud |
|--------------|----------------------|------------------------|
| AWS | Redshift or Snowflake | Snowflake if multi-cloud needed |
| GCP | BigQuery | Default; no reason to choose otherwise on GCP |
| Azure | Synapse Analytics or Snowflake | Synapse for Microsoft stack; Snowflake for cross-cloud |
| Cloud-neutral | Snowflake | Works on AWS, GCP, and Azure |

### Orchestration Tool Comparison

| Tool | Best For | Avoid When |
|------|----------|------------|
| Apache Airflow (self-hosted) | Full control, custom operators, Kubernetes | Team lacks Airflow operational experience |
| Cloud Composer (GCP) | GCP-native pipelines, managed Airflow | Cost-sensitive (Composer is expensive) |
| AWS Step Functions | AWS-native workflows, Lambda-heavy pipelines | Complex dependency management |
| AWS Glue Workflows | Glue-only pipelines | Mixed orchestration (non-Glue steps) |
| dbt (for transforms only) | SQL transformation scheduling | Full pipeline orchestration |

---

## Building Advice

**1. Start with the project that matches your target role.**
Cloud provider preferences vary: AWS for startups and mid-size tech, Azure for enterprise/Microsoft shops, GCP for data science-heavy orgs. Pick the project that matches where you want to work.

**2. Build end-to-end, not just the "interesting" parts.**
Stopping after Kafka + Spark with no warehouse and no dashboard isn't a complete project. The final 20% (loading, dashboarding, documentation) is what makes it portfolio-worthy.

**3. Document your WHY, not just your WHAT.**
Your README should answer: "Why did you choose Kafka over Kinesis? Why Parquet over CSV? Why star schema?" This document models that reasoning. Interviewers ask these questions.

**4. Handle failures.**
Add retry logic to your pipelines. Log failures to a job tracking table. Write a dead-letter queue for malformed records. Real production pipelines fail; showing you planned for failure demonstrates maturity.

**5. Progress path if you're new:**
Project 7 (MySQL ETL) → Project 4 (Spotify / AWS) → Project 2 (AQI / serverless) → Project 3 (open-source Kafka) → Projects 1, 5, 6 (advanced).

---

*System Design Reference — Based on Darshil Parmar's "7 Data Engineering Projects You Should Build in 2026"*
