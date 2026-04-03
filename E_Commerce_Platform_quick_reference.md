# 🏗️ Lead Data Engineer — System Design Interview Guide


---

> **Candidate Profile:** Senior Data Engineer / Principal Engineer with 8+ years experience  
> **Round:** System Design — Lead Data Engineer  
> **Company Context:** FAANG-scale engineering, AWS-native stack, petabyte-scale data  
> **Philosophy:** *"Design for today's scale, architect for tomorrow's growth, optimize for operational simplicity."*

---

## 🧠 Master Interview Mnemonic — **"SCARED MOST"**

Before answering ANY system design question, structure your answer using:

```
S — Scale & Scope          (clarify requirements, constraints, SLAs)
C — Components             (identify the building blocks)
A — Architecture           (draw the high-level design)
R — Reliability            (fault tolerance, redundancy, recovery)
E — Efficiency             (cost, performance, resource optimization)
D — Data Flow              (how data moves through the system)

M — Monitoring             (observability, alerting, dashboards)
O — Optimization           (future improvements, bottlenecks)
S — Security               (encryption, access control, compliance)
T — Trade-offs             (justify every design decision)
```

> 💡 Use this framework for EVERY question. Interviewers at FAANG reward structured thinking over random answers.

---

---

# ❓ Question 1: Design an End-to-End Scalable Data Platform for a Growing E-Commerce Company

### Context:
- **10 Million daily active users**
- **Real-time analytics** (live dashboards, fraud detection)
- **Batch reporting** (daily/weekly business reports)
- **ML use cases** (recommendations, churn prediction, demand forecasting)

---

## 🧠 Mnemonic to Answer Q1 — **"RICE BOWL"**

```
R — Requirements Clarification  (always clarify before designing)
I — Ingestion Layer             (how data enters the system)
C — Catalog & Storage           (where and how data is organized)
E — ETL & Processing            (how data is transformed)

B — BI & Analytics              (how data is consumed)
O — Orchestration               (how pipelines are scheduled/managed)
W — Workloads (ML)              (how ML models are trained/served)
L — Logging, Monitoring, Cost   (observability and governance)
```

---

## Step 1 — R: Requirements Clarification

**Always start by asking clarifying questions. This signals senior engineering mindset.**

```
Functional Requirements:
├── Real-time analytics: What latency? (<1 sec, <5 sec, <30 sec?)
├── Batch reporting: What frequency? (hourly, daily, weekly)
├── ML: Online inference or offline batch predictions?
├── Data sources: Clickstream, transactions, inventory, CRM?
└── Users: How many data consumers? (analysts, data scientists, executives)

Non-Functional Requirements:
├── Availability:    99.9% uptime (8.7 hours downtime/year acceptable?)
├── Durability:      Zero data loss for transactions
├── Latency:         Real-time < 5 seconds, Batch < 4 hours SLA
├── Scale:           10M DAU, ~500M events/day (assume 50 events/user)
├── Retention:       Hot data 90 days, Cold data 7 years (compliance)
└── Compliance:      PCI-DSS (payments), GDPR (EU users)

Assumed Numbers (back-of-envelope):
├── Events/day:      10M users × 50 events = 500M events/day
├── Event size:      ~1 KB average
├── Raw data/day:    500M × 1KB = ~500 GB/day
├── Peak QPS:        500M / 86400 sec × 3x peak factor ≈ 17,000 events/sec
└── Storage/year:    500GB × 365 × 3x replication ≈ ~550 TB/year
```

> 💡 **Interview tip:** Back-of-envelope math shows you think like an engineer, not just an architect.

---

## Step 2 — I: Ingestion Layer

**Mnemonic for Ingestion: "KAF" — Kinesis + API Gateway + Firehose**

```
┌─────────────────────────────────────────────────────────────────┐
│                      INGESTION LAYER                            │
│                                                                 │
│  Clickstream  ──► API Gateway ──► Kinesis Data Streams         │
│  App Events   ──►               ──► (17,000 events/sec)        │
│                                                                 │
│  Databases    ──► AWS DMS      ──► Kinesis / S3                │
│  (RDS, etc.)     (CDC)                                         │
│                                                                 │
│  3rd Party    ──► AWS AppFlow  ──► S3                          │
│  (Salesforce,                                                  │
│   Shopify)                                                     │
│                                                                 │
│  Logs/Files   ──► S3 (direct) ──► EventBridge ──► Glue        │
└─────────────────────────────────────────────────────────────────┘
```

### Component Decisions:

**Kinesis Data Streams** for real-time event ingestion:
- ✅ Handles 17,000+ events/sec with horizontal shard scaling
- ✅ 24-hour default retention (up to 365 days extended)
- ✅ Fan-out to multiple consumers (Lambda, Flink, Kinesis Analytics)
- ❌ More expensive than SQS for simple queuing
- ❌ Requires shard management (auto-scaling available but adds complexity)

**AWS DMS (Database Migration Service)** for CDC:
- ✅ Captures row-level changes from RDS/Aurora without touching source
- ✅ Supports Oracle, MySQL, PostgreSQL, SQL Server
- ❌ DMS tasks need monitoring — replication lag can grow under heavy load

**API Gateway + Kinesis direct integration:**
- ✅ No Lambda in the hot path = lower latency, lower cost
- ✅ Handles burst traffic with API Gateway throttling as circuit breaker

---

## Step 3 — C: Catalog & Storage (Data Lake — Medallion Architecture)

**Mnemonic for Storage: "BSG" — Bronze, Silver, Gold on S3**

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA LAKE ON S3                              │
│                                                                 │
│  s3://ecom-datalake/                                           │
│  ├── bronze/                  ← Raw, immutable, append-only    │
│  │   ├── clickstream/         (Parquet, partitioned by date)   │
│  │   ├── orders/              (CDC from RDS)                   │
│  │   └── inventory/                                            │
│  │                                                             │
│  ├── silver/                  ← Cleaned, deduplicated          │
│  │   ├── user_events/         (Delta Lake format)              │
│  │   ├── orders_cleaned/                                       │
│  │   └── customer_profiles/                                    │
│  │                                                             │
│  ├── gold/                    ← Business-ready aggregates      │
│  │   ├── daily_revenue/                                        │
│  │   ├── customer_360/                                         │
│  │   └── product_metrics/                                      │
│  │                                                             │
│  └── features/                ← ML feature store               │
│      ├── user_features/                                        │
│      └── product_features/                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Storage Decisions:

**Amazon S3 as Data Lake Foundation:**
- ✅ 11 nines (99.999999999%) durability
- ✅ Unlimited scale, no capacity planning
- ✅ S3 Intelligent-Tiering auto-moves cold data to cheaper storage
- ✅ Native integration with every AWS analytics service
- ❌ Not suitable for row-level updates without Delta Lake / Iceberg

**Delta Lake (via AWS Glue or EMR):**
- ✅ ACID transactions on S3
- ✅ Time travel — query data as of any past timestamp
- ✅ Schema enforcement and evolution
- ✅ Z-ordering for query optimization
- ❌ Requires Spark for writes (no direct S3 write)
- ❌ Small file problem with high-frequency streaming writes (needs compaction)

**AWS Glue Data Catalog:**
- ✅ Unified metastore for Glue, Athena, EMR, Redshift Spectrum
- ✅ Crawlers auto-discover schemas
- ❌ Crawler inference errors require manual correction

---

## Step 4 — E: ETL & Processing

**Mnemonic for Processing: "GLASS"**
```
G — Glue (batch ETL)
L — Lambda (lightweight event processing)
A — Apache Flink / Kinesis Analytics (real-time stream processing)
S — Spark on EMR (heavy computation)
S — SageMaker Processing (ML feature engineering)
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                       PROCESSING LAYER                              │
│                                                                     │
│  REAL-TIME PATH (< 5 seconds latency):                             │
│  Kinesis Streams ──► Kinesis Data Analytics (Flink)                │
│                       ├── Fraud detection (< 200ms)                │
│                       ├── Live dashboard aggregations              │
│                       └── Write to DynamoDB / ElastiCache          │
│                                                                     │
│  NEAR-REAL-TIME PATH (< 5 minutes latency):                        │
│  Kinesis Streams ──► Kinesis Firehose ──► S3 Bronze                │
│                       (buffer 60 sec, 128 MB)                      │
│                                                                     │
│  BATCH PATH (hourly / daily):                                       │
│  S3 Bronze ──► AWS Glue (ETL) ──► S3 Silver ──► S3 Gold           │
│                ├── Bronze → Silver: cleanse, dedup                  │
│                └── Silver → Gold:  aggregate, enrich               │
│                                                                     │
│  HEAVY COMPUTATION (weekly ML features, large joins):              │
│  S3 Silver ──► EMR (Spark) ──► S3 Features / Gold                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Processing Decisions:

**Kinesis Data Analytics (Apache Flink) for real-time:**
- ✅ Sub-second latency for fraud detection, live metrics
- ✅ Exactly-once processing semantics
- ✅ Serverless — auto-scales with stream throughput
- ✅ SQL or Java/Python APIs
- ❌ Stateful operations require careful state backend management
- ❌ More expensive than batch processing for the same data volume

**AWS Glue for batch ETL:**
- ✅ Serverless, no cluster management
- ✅ Pay-per-use (cost-efficient for daily jobs)
- ✅ Built-in job bookmarks for incremental loads
- ❌ 2–3 min cold start latency
- ❌ Not ideal for sub-hour batch windows

**EMR for heavy workloads:**
- ✅ Full Spark control, tunable configuration
- ✅ Cost-effective at scale with Spot Instances (60–80% savings)
- ✅ Supports Spark, Hive, Presto, HBase
- ❌ Cluster management overhead
- ❌ Cluster startup adds latency

---

## Step 5 — B: BI & Analytics (Consumption Layer)

**Mnemonic: "RAQ" — Redshift + Athena + QuickSight**

```
┌─────────────────────────────────────────────────────────────────────┐
│                     CONSUMPTION LAYER                               │
│                                                                     │
│  Real-time Dashboards:                                             │
│  DynamoDB / ElastiCache ──► Custom API ──► React Dashboard         │
│  (pre-aggregated by Flink)                                          │
│                                                                     │
│  Ad-hoc SQL Analytics:                                             │
│  S3 Gold ──► Amazon Athena ──► BI Tools (Tableau, Looker)          │
│  (serverless, pay-per-scan)                                         │
│                                                                     │
│  Complex / Concurrent Analytics:                                   │
│  S3 Gold ──► Redshift Spectrum ──► Redshift ──► Business Reports   │
│  (for high-concurrency, complex joins on PB-scale)                  │
│                                                                     │
│  Operational Metrics:                                              │
│  CloudWatch ──► QuickSight ──► Executive Dashboards                │
└─────────────────────────────────────────────────────────────────────┘
```

### Analytics Decisions:

**Amazon Athena (primary ad-hoc):**
- ✅ Serverless — zero infrastructure
- ✅ Pay per scan ($5/TB scanned)
- ✅ Partition pruning + Parquet = 90% cost reduction
- ✅ Federated queries to RDS, DynamoDB
- ❌ Slower than Redshift for high-concurrency (>20 concurrent users)
- ❌ No in-memory caching (unlike Redshift)

**Amazon Redshift (high-concurrency analytics):**
- ✅ MPP (Massively Parallel Processing) — sub-second on TBs
- ✅ Concurrency Scaling — handles burst users automatically
- ✅ Redshift Spectrum — query S3 without loading data
- ✅ ML functions built-in (CREATE MODEL)
- ❌ Not serverless by default (Redshift Serverless available)
- ❌ Requires data loading (COPY) for best performance

---

## Step 6 — O: Orchestration

**Mnemonic: "A-STEP" — Airflow + Step Functions + EventBridge + Prefect**

```
Pipeline Orchestration Strategy:
├── AWS Step Functions        — Serverless workflow for Glue + Lambda chains
├── Amazon MWAA (Airflow)     — Complex DAGs, cross-system dependencies
├── EventBridge               — Event-driven triggers (S3 upload → Glue job)
└── Glue Workflows            — Glue-native job sequencing (Crawl → ETL → Report)

Decision:
├── Use MWAA (Airflow) as primary orchestrator
│   ✅ Industry standard, rich operator ecosystem
│   ✅ Dependency management, retry logic, SLA alerts
│   ❌ Always-on cost (~$400/month for small cluster)
│
└── Use EventBridge for event-driven triggers
    ✅ Zero cost at rest
    ✅ Decoupled architecture
    ✅ Fan-out to multiple consumers
```

---

## Step 7 — W: ML Workloads

**Mnemonic: "FTSD" — Feature Store → Training → Serving → Drift**

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ML PLATFORM                                  │
│                                                                     │
│  FEATURE ENGINEERING:                                              │
│  S3 Silver ──► Glue / EMR ──► SageMaker Feature Store             │
│               (daily batch feature computation)                     │
│                                                                     │
│  MODEL TRAINING:                                                   │
│  SageMaker Feature Store ──► SageMaker Training Jobs               │
│  ├── Recommendation Engine  (Collaborative Filtering / NCF)        │
│  ├── Churn Prediction        (XGBoost)                             │
│  └── Demand Forecasting      (DeepAR / Prophet)                    │
│                                                                     │
│  MODEL SERVING:                                                    │
│  SageMaker Model Registry ──► SageMaker Endpoints                  │
│  ├── Real-time: Recommendations API (<100ms)                       │
│  └── Batch:    Churn scores (daily SageMaker Batch Transform)      │
│                                                                     │
│  MLOPS:                                                            │
│  SageMaker Pipelines ──► Model Monitor ──► CloudWatch Alerts       │
│  (automated retraining when drift detected)                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Step 8 — L: Logging, Monitoring, Security, Cost

```
MONITORING STACK:
├── CloudWatch        — Metrics, logs, alarms for all AWS services
├── AWS X-Ray         — Distributed tracing for API / Lambda
├── Glue job metrics  — DPU usage, bytes processed, error counts
├── QuickSight        — Self-service operational dashboards
└── PagerDuty         — On-call alerting for pipeline SLA breaches

SECURITY STACK:
├── Lake Formation    — Column-level, row-level data access control
├── KMS               — Encryption at rest for all S3, Redshift, Kinesis
├── VPC               — All compute in private subnets
├── IAM Roles         — Least-privilege per service
├── Secrets Manager   — No hardcoded credentials anywhere
└── Macie             — PII detection in S3 data lake

COST OPTIMIZATION:
├── S3 Intelligent-Tiering      — Auto cold/warm/archive tiering
├── EMR Spot Instances          — 60–80% cost reduction for batch
├── Glue auto-scaling           — Pay only for actual DPU usage
├── Athena partitioning         — Reduce scan costs 90%+
└── Reserved Instances          — Redshift 1-yr reserved = 40% savings
```

---

## Complete Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    E-COMMERCE DATA PLATFORM — FULL ARCHITECTURE            │
│                                                                            │
│  ┌─────────────┐   ┌──────────────┐   ┌─────────────────────────────────┐ │
│  │   SOURCES   │   │  INGESTION   │   │        DATA LAKE (S3)           │ │
│  │             │   │              │   │  ┌─────────┬────────┬─────────┐  │ │
│  │ Clickstream ├──►│ API GW +     ├──►│  │ BRONZE  │ SILVER │  GOLD  │  │ │
│  │ App Events  │   │ Kinesis DS   │   │  │ (Raw)   │(Clean) │ (Biz)  │  │ │
│  │             │   │              │   │  └────┬────┴───┬────┴────┬────┘  │ │
│  │ RDS Orders  ├──►│ AWS DMS(CDC) ├──►│       │        │         │       │ │
│  │             │   │              │   │       ▼        ▼         ▼       │ │
│  │ 3rd Party   ├──►│ AppFlow      ├──►│  ┌─────────────────────────────┐ │ │
│  │ (Shopify)   │   │              │   │  │    Glue Data Catalog        │ │ │
│  └─────────────┘   └──────────────┘   │  └─────────────────────────────┘ │ │
│                                       └─────────────────────────────────┘ │
│                                                   │                        │
│  ┌─────────────────────────────────────────────────▼─────────────────────┐ │
│  │                         PROCESSING LAYER                              │ │
│  │  Real-time: Kinesis Analytics (Flink) ──► DynamoDB / ElastiCache      │ │
│  │  Batch:     AWS Glue / EMR (Spark)   ──► S3 Silver / Gold             │ │
│  │  ML:        SageMaker Pipelines      ──► Feature Store / Endpoints    │ │
│  └────────────────────────────────────┬───────────────────────────────── ┘ │
│                                       │                                    │
│  ┌────────────────────────────────────▼──────────────────────────────────┐ │
│  │                       CONSUMPTION LAYER                               │ │
│  │  Athena ──► Ad-hoc SQL          Redshift ──► Business Reporting       │ │
│  │  Kinesis + DynamoDB ──► Live Dashboards    SageMaker ──► ML APIs      │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────┘
```

---

---

# ❓ Question 2: How Do You Design the Same for 10x Data Growth?

### Context: From 500 GB/day → 5 TB/day, 10M → 100M DAU

---

## 🧠 Mnemonic — **"SHAPES"**

```
S — Sharding & Partitioning    (distribute data more granularly)
H — Horizontal Scaling         (add compute capacity, not just size)
A — Async everywhere           (decouple producers from consumers)
P — Pre-aggregation            (compute at ingest, not at query time)
E — Elasticity                 (auto-scale every component)
S — Storage tiering            (cost-optimize at petabyte scale)
```

---

## Step-by-Step 10x Scale Changes

### Change 1 — Ingestion: Kinesis Shard Scaling

```
Current (10M DAU):   17,000 events/sec → ~50 Kinesis shards
10x (100M DAU):     170,000 events/sec → ~500 Kinesis shards

Actions:
├── Enable Kinesis Auto-scaling (on-demand mode)
│   ✅ Automatic shard splits/merges based on throughput
│   ✅ No manual shard management
│   ❌ On-demand is 3x more expensive per shard-hour than provisioned
│
└── Evaluate MSK (Managed Kafka) at this scale
    ✅ Better throughput control, lower cost at high volume
    ✅ Native Kafka API — portable, not AWS-locked
    ✅ Multi-AZ partitions — higher durability
    ❌ More operational complexity than Kinesis
    ❌ Requires Kafka expertise on the team
```

> **Decision:** Migrate from Kinesis to **MSK (Kafka)** at 100M DAU. The cost and throughput advantages outweigh the operational complexity at this scale.

---

### Change 2 — Processing: Glue → EMR + Glue Hybrid

```
Current:  AWS Glue handles all batch ETL
10x:      Glue cannot efficiently handle 5TB/day jobs within SLA windows

Problem:
├── A 5TB Glue job at G.2X workers = ~120 workers running 2+ hours
├── Cost: 120 workers × 2 hrs × $0.29/DPU-hr ≈ $70/run × 5 runs/day = $350/day
└── $350/day = $127,750/year — just for batch ETL

Solution: EMR Spot Cluster for heavy jobs
├── EMR r6g.8xlarge Spot Instances = ~$0.70/hr (vs $6/hr on-demand)
├── 20-node cluster × 0.70 × 2hrs = $28/run × 5 = $140/day
└── Savings: 60% cost reduction with better performance

Architecture:
├── Glue:  Lightweight Bronze → Silver transforms (< 500GB)
├── EMR:   Heavy Silver → Gold joins and aggregations (5TB)
└── Flink: Unchanged for real-time — scales horizontally
```

**Decision Table:**

| Workload Size | Tool | Reason |
|---|---|---|
| < 100 GB | Glue | Serverless, fast startup |
| 100 GB – 1 TB | Glue with G.4X workers | Managed, acceptable cost |
| 1 TB – 10 TB | EMR Spot | Cost-efficient, full control |
| > 10 TB | EMR + Spark Adaptive Query | Maximum performance |

---

### Change 3 — Storage: Table Format Migration

```
Current:  Parquet files on S3 (append-only)
10x:      Parquet breaks down at 5TB/day due to:
          ├── Small file proliferation from streaming writes
          ├── No efficient upserts (full partition rewrites)
          └── No time travel for debugging bad data

Solution: Migrate to Apache Iceberg on S3

Apache Iceberg advantages at 10x scale:
├── ✅ Partition evolution   — change partitioning without rewriting data
├── ✅ Hidden partitioning   — auto-partitions without user knowing column values
├── ✅ Row-level deletes     — GDPR compliance without full partition rewrite
├── ✅ Time travel           — query yesterday's data for debugging
├── ✅ Concurrent writes     — multiple jobs write simultaneously without corruption
├── ✅ Compaction built-in   — merge small files automatically
└── ❌ Requires Spark 3+, EMR 6+, or Athena v3 (AWS native support available)

Migration Path:
Week 1: New tables in Iceberg format (Gold layer first)
Week 2: Migrate Silver layer, validate with Athena v3
Week 3: Bronze layer migration (most impact on streaming)
Week 4: Decommission legacy Parquet paths
```

---

### Change 4 — Analytics: Athena → Redshift Serverless

```
Current (10M DAU):  Athena handles ad-hoc queries (~20 concurrent users)
10x (100M DAU):     200+ concurrent analyst queries — Athena degrades

Problem with Athena at 10x:
├── No result caching (same query by 50 users = 50 × scan cost)
├── No concurrency guarantees
└── 5TB scans × $5/TB = $25/query (expensive at high concurrency)

Solution: Redshift Serverless for high-concurrency
├── ✅ Scales RPUs (Redshift Processing Units) automatically
├── ✅ Materialized Views for pre-computed aggregates
├── ✅ Result caching — same query by 50 users = 1 execution
├── ✅ Redshift Spectrum — still queries S3 for cold data
└── ❌ ~3x more expensive than Athena for light, infrequent queries

Hybrid Strategy:
├── Athena:            Infrequent, exploratory ad-hoc queries
├── Redshift Serverless: High-concurrency dashboards, standard reports
└── ElastiCache (Redis): Pre-computed KPIs for real-time dashboards
```

---

### Change 5 — ML: Feature Store Scaling

```
Current:  SageMaker Feature Store (managed, simple)
10x:      Feature computation becomes the bottleneck

Problems:
├── 100M users × 50 features = 5 billion feature values/day to compute
├── Online feature store (DynamoDB) reads spike during peak traffic
└── Training job times increase with 10x data

Solutions:
├── Spark-based feature pipeline on EMR (parallel feature computation)
├── Feature materialization during off-peak hours (2AM–6AM)
├── ElastiCache Redis as L1 cache in front of SageMaker Feature Store
│   ✅ Cache hit rate > 90% for popular user features
│   ✅ < 1ms latency vs 10ms from DynamoDB
└── Feature versioning — only recompute changed features (not all 50)
```

---

### Change 6 — Data Governance at 10x Scale

```
At 10x growth, governance becomes critical:
├── More teams = more accidental data access violations
├── More data = higher PII exposure risk
├── More pipelines = harder to track data lineage

Solutions:
├── AWS Lake Formation    — Fine-grained column/row-level access control
├── AWS Macie             — Automated PII detection in new S3 files
├── OpenMetadata / DataHub — Data catalog with lineage tracking
└── Data contracts        — Schema ownership, SLA commitments per table
```

---

### 10x Architecture Delta Summary

| Component | Current (10M DAU) | 10x (100M DAU) |
|---|---|---|
| **Ingestion** | Kinesis Data Streams | MSK (Kafka) |
| **Stream Processing** | Kinesis Analytics (Flink) | Flink on EMR / MSK Flink |
| **Batch ETL** | AWS Glue | Glue + EMR Spot |
| **Table Format** | Parquet | Apache Iceberg |
| **Query Engine** | Athena | Redshift Serverless + Athena |
| **Feature Store** | SageMaker Feature Store | SageMaker + Redis Cache |
| **Governance** | Basic IAM | Lake Formation + OpenMetadata |
| **Cost/day (estimate)** | ~$2,000/day | ~$8,000/day (4x, not 10x) |

> 💡 **Key insight:** Good architecture means cost scales sub-linearly with data growth. 10x data = 4x cost, not 10x cost.

---

---

# ❓ Question 3: A Query Takes 30 Minutes. What Do You Do?

---

## 🧠 Mnemonic — **"DIPS"**

```
D — Diagnose first    (understand before optimizing)
I — Index & Partition (physical data organization)
P — Push-down & Prune (reduce data scanned)
S — Serve from cache  (pre-compute and cache results)
```

---

## Step 1 — D: Diagnose First (Never Guess)

```
Diagnostic Checklist:
├── WHERE is the query running? (Athena, Redshift, Spark, Presto)
├── WHAT does EXPLAIN ANALYZE show? (identify the bottleneck node)
├── HOW MUCH data is being scanned? (full table scan vs partition scan)
├── ARE there data skews? (one partition 100x larger than others)
├── IS it CPU-bound or IO-bound? (different solutions for each)
└── IS the query inherently serial? (some queries can't be parallelized)

Tools by platform:
├── Athena:     Query details page → Data scanned, execution plan
├── Redshift:   STL_EXPLAIN, SVL_QUERY_REPORT, STL_WLM_QUERY
├── Spark:      Spark UI → Stage details, DAG visualization
└── Glue:       CloudWatch metrics → executor idle time, spill-to-disk
```

---

## Step 2 — I: Index, Partition, and Physical Organization

```
PROBLEM: Full table scan on 5TB table
ROOT CAUSE: No partitioning / wrong partitioning strategy

┌──────────────────────────────────────────────────────────────┐
│  BEFORE: Unpartitioned table                                 │
│  SELECT * FROM orders WHERE order_date = '2024-03-15'        │
│  → Scans 5TB of data → 30 minutes                           │
│                                                              │
│  AFTER: Partitioned by order_date                            │
│  → Scans only 1 day's partition (~14GB) → 45 seconds        │
└──────────────────────────────────────────────────────────────┘

Partitioning Strategy for E-Commerce:
├── Primary partition:   order_date (most queries filter by date)
├── Sub-partition:       region or category (if further filtering common)
└── Avoid over-partitioning: Don't partition by user_id (10M partitions!)

Additional physical optimizations:
├── Z-ordering (Delta Lake):   Co-locate customer_id + product_id values
│   → Speeds up JOIN-heavy queries 3–10x
├── Bloom filters:             Skip files that don't contain a value
│   → Useful for high-cardinality filter columns (order_id lookups)
├── File size:                 Target 128MB–1GB per Parquet file
│   → Too small: metadata overhead; Too large: can't parallelize
└── Columnar compression:      Snappy (fast) vs ZSTD (better ratio)
    → ZSTD recommended for cold data, Snappy for hot data
```

---

## Step 3 — P: Push-down Predicates & Prune Data

```
Optimization Techniques:

1. SELECT only needed columns (never SELECT *):
   ┌─────────────────────────────────────────────────┐
   │ BAD:  SELECT * FROM orders WHERE ...            │
   │       → Reads ALL columns from Parquet (30 min) │
   │                                                 │
   │ GOOD: SELECT order_id, amount, customer_id      │
   │       FROM orders WHERE ...                     │
   │       → Reads only 3 columns (4 min)            │
   └─────────────────────────────────────────────────┘

2. Predicate push-down (filter before join):
   ┌─────────────────────────────────────────────────┐
   │ BAD:  JOIN then filter                          │
   │ SELECT * FROM orders o JOIN customers c         │
   │ ON o.customer_id = c.id                         │
   │ WHERE o.order_date = '2024-03-15'               │
   │ → Joins 5TB then filters                        │
   │                                                 │
   │ GOOD: Filter then join (use subquery/CTE)       │
   │ WITH daily_orders AS (                          │
   │   SELECT * FROM orders                          │
   │   WHERE order_date = '2024-03-15'               │  ← filter first
   │ )                                               │
   │ SELECT * FROM daily_orders o                    │
   │ JOIN customers c ON o.customer_id = c.id        │
   └─────────────────────────────────────────────────┘

3. Avoid functions on partition columns:
   ┌─────────────────────────────────────────────────┐
   │ BAD:  WHERE year(order_date) = 2024             │
   │       → Can't use partition pruning             │
   │                                                 │
   │ GOOD: WHERE order_date BETWEEN '2024-01-01'     │
   │                           AND '2024-12-31'      │
   │       → Uses partition pruning                  │
   └─────────────────────────────────────────────────┘

4. Broadcast joins for small tables:
   ┌─────────────────────────────────────────────────┐
   │ When joining a 5TB table with a 10MB lookup:    │
   │ /*+ BROADCAST(small_table) */                   │
   │ → Broadcasts small table to all executors       │
   │ → Eliminates shuffle join (biggest Spark cost)  │
   └─────────────────────────────────────────────────┘
```

---

## Step 4 — S: Serve from Pre-Computed Results

```
For queries that are slow AND frequently repeated:

Strategy 1: Materialized Views (Redshift / Athena)
├── Pre-compute expensive aggregations on a schedule
├── Query hits the materialized view (<1 sec) not the raw table (30 min)
└── Refresh: hourly, daily, or on new data arrival

Strategy 2: Summary Tables / Gold Layer Tables
├── Create gold layer tables specifically for the slow query pattern
├── Run expensive aggregation as a nightly Glue job (cheap at 3AM)
└── Dashboard reads from gold table = instant

Strategy 3: ElastiCache Redis for KPI endpoints
├── For dashboard KPIs queried >100x/min:
│   Write KPI to Redis (TTL 5 min) → all requests hit Redis
└── Query runs once every 5 min; 100+ users see cached results

Strategy 4: Workload Management (Redshift WLM)
├── Identify: is the 30-min query blocking other users?
├── Assign long-running queries to their own WLM queue
└── Short queries don't wait behind 30-min monsters
```

---

## Root Cause → Solution Matrix

| Root Cause | Diagnosis Signal | Solution | Expected Improvement |
|---|---|---|---|
| Full table scan | Data scanned = table size | Add partition filter | 10–100x faster |
| Data skew | One executor takes 10x longer | Salting / repartition | 3–10x faster |
| Wrong join order | Large × Large join | Reorder, broadcast small | 5–20x faster |
| No column pruning | SELECT * everywhere | Select only needed cols | 2–5x faster |
| Serial execution | Single-threaded query | Repartition, parallelize | 5–15x faster |
| Repeated computation | Same query by many users | Materialize / cache | 100x+ faster |
| Small files | Millions of tiny Parquet files | Compact to 128MB+ | 3–10x faster |

---

---

# ❓ Question 4: Pipeline Grows from 200 GB/day → 5 TB/day. What Changes?

---

## 🧠 Mnemonic — **"PIECES"**

```
P — Partition strategy         (redesign for higher cardinality)
I — Ingestion pipeline         (scale the front door)
E — ETL compute                (right-size the engines)
C — Compression & file format  (reduce physical data volume)
E — Execution model            (batch sizing, parallelism)
S — Storage & cost             (tiering, lifecycle, compaction)
```

---

## Step 1 — P: Redesign Partition Strategy

```
Current (200GB/day):
├── Partition by: year=2024/month=03/day=15/
└── Works fine — each partition ≈ 200GB

Problem at 5TB/day:
├── Single day partition = 5TB → queries still scan too much
└── Need sub-day partitioning

New partition scheme:
├── year=2024/month=03/day=15/hour=14/
└── Each partition ≈ 5TB / 24 = ~200GB per hour

For high-cardinality event types:
├── event_type=purchase/year=2024/month=03/day=15/
└── Allows analysts to filter by event type AND date

Iceberg hidden partitioning (best practice):
├── No partition columns exposed to users
├── Iceberg auto-creates optimal partitions based on data distribution
└── Partition strategy can be changed without data rewrites
```

---

## Step 2 — I: Scale the Ingestion Pipeline

```
Current (200GB/day):
├── Peak: ~2.3 MB/sec
├── Kinesis: 10 shards (1 MB/sec each = 10 MB/sec capacity)
└── Glue: 10 workers, hourly batch loads

5TB/day:
├── Peak: ~58 MB/sec average, ~174 MB/sec at 3x peak
├── Required Kinesis shards: 174 MB/sec ÷ 1 MB/sec = 174 shards
└── Or migrate to MSK (Kafka) for better throughput economics

Changes to ingestion:
├── Kinesis: Enable on-demand mode (auto-scales shards)
├── Firehose: Increase buffer to 128 MB / 60 seconds
│   → Fewer, larger S3 files → less metadata overhead
├── Schema Registry: Enforce schema at producer (Glue Schema Registry)
│   → Bad records rejected at source, not discovered 3 hours later
└── Dead Letter Queue: Capture failed events for reprocessing
    → Never lose data due to schema mismatch or downstream failure
```

---

## Step 3 — E: Right-Size ETL Compute

```
Current (200GB/day):
├── Glue: 10 × G.1X workers, 1 hour runtime
└── Cost: 10 DPUs × 1 hr × $0.44 = $4.40/run

5TB/day (25x more data):
├── Naïve scaling: 250 × G.1X workers = $110/run (too expensive)
└── Smart scaling: Optimize first, then scale

Optimization before scaling:

1. Switch from Glue G.1X to G.2X workers:
   ├── G.2X has 2x memory → less spill-to-disk
   └── Jobs that spill can be 5x slower — more memory = faster job

2. Use EMR Spot Instances for large jobs:
   ├── r6g.4xlarge spot ≈ $0.40/hr vs $1.60/hr on-demand
   ├── 20-node cluster × $0.40 × 2 hrs = $16/run
   └── vs Glue 100 workers × $0.44 × 2 hrs = $88/run

3. Pipeline decomposition:
   ├── Don't run one monolithic 5TB job
   ├── Split by event_type or region
   ├── Run 10 parallel jobs × 500GB each
   └── Total time: same 2 hours, but fault-isolated

4. Incremental processing:
   ├── Process only new files since last run (Glue job bookmarks)
   ├── At 5TB/day with hourly runs: each run processes ~200GB
   └── Keeps individual job sizes manageable

   ┌──────────────────────────────────────────────────┐
   │ Monolithic:  1 job × 5TB × 4 hrs = risky        │
   │ Incremental: 24 jobs × 200GB × 20 min = better  │
   └──────────────────────────────────────────────────┘
```

---

## Step 4 — C: Compression & File Format Optimization

```
At 5TB/day, file format choice directly impacts:
├── Storage cost        (compressed Parquet vs raw JSON = 10x savings)
├── Query performance   (columnar reads vs row scans)
└── Downstream costs    (Athena charges per TB scanned)

Format Comparison for 5TB/day:
┌────────────────┬───────────┬──────────────┬────────────────────┐
│ Format         │ Size      │ Scan Speed   │ Best For           │
├────────────────┼───────────┼──────────────┼────────────────────┤
│ JSON (raw)     │ 5TB/day   │ Very slow    │ Bronze only        │
│ CSV            │ 4TB/day   │ Slow         │ Avoid              │
│ Parquet+Snappy │ 800GB/day │ Fast         │ Silver/Gold        │
│ Parquet+ZSTD   │ 500GB/day │ Fast         │ Cold archive       │
│ ORC            │ 600GB/day │ Very fast    │ Hive workloads     │
│ Iceberg+ZSTD   │ 450GB/day │ Fastest      │ Production at scale│
└────────────────┴───────────┴──────────────┴────────────────────┘

Recommendation:
├── Bronze:    Parquet + Snappy (fast write, readable)
├── Silver:    Iceberg + Snappy (ACID, time travel)
├── Gold:      Iceberg + ZSTD  (best compression for cold-ish data)
└── Archive:   S3 Glacier with ZSTD (7-year retention at $0.004/GB/month)
```

---

## Step 5 — E: Execution Model Changes

```
Parallelism tuning for 5TB/day:

1. Spark partition sizing:
   ├── Rule: Each Spark partition should be 128MB–256MB
   ├── 5TB ÷ 200MB = 25,600 Spark partitions optimal
   └── spark.sql.shuffle.partitions = 25600

2. Adaptive Query Execution (AQE) — enable it:
   ├── spark.sql.adaptive.enabled = true
   ├── Automatically re-optimizes at runtime
   ├── Handles data skew without manual salting
   └── Merges small partitions post-shuffle

3. Dynamic Partition Overwrite:
   ├── Only overwrite partitions that have new data
   ├── spark.sql.sources.partitionOverwriteMode = dynamic
   └── 200GB/day job doesn't rewrite all 5TB every run

4. Checkpointing for streaming:
   ├── 5TB/day streams need reliable checkpoints
   ├── Store Flink/Spark checkpoints in S3 (durable)
   └── Set checkpoint interval: every 60 seconds
```

---

## Step 6 — S: Storage Cost Optimization at 5TB/day

```
Storage cost analysis at 5TB/day:
├── Raw:       5TB/day × 365 = 1.8PB/year × $0.023/GB = $41,400/year
├── Compressed: 800GB/day × 365 = 288TB × $0.023/GB = $6,624/year
└── With tiering: Hot (90d) + Warm (1yr) + Glacier (7yr) = $3,200/year

S3 Lifecycle Policy:
├── 0–90 days:   S3 Standard          ($0.023/GB/month)
├── 90–365 days: S3 Standard-IA       ($0.0125/GB/month)
├── 1–7 years:   S3 Glacier Instant   ($0.004/GB/month)
└── 7+ years:    S3 Glacier Deep      ($0.00099/GB/month)

Compaction jobs (critical at 5TB/day):
├── Streaming writes create many small files (problem!)
│   Example: 5TB/day ÷ 1,000 events/file = 5M files/day
├── Run hourly compaction job to merge small files → 128MB targets
└── Without compaction: S3 LIST operations slow, Athena metadata overhead

Table maintenance (Iceberg/Delta):
├── VACUUM:   Remove old versions older than 7 days (free up storage)
├── OPTIMIZE: Compact small files into larger ones
└── ANALYZE:  Refresh table statistics for query planner
```

---

## Complete Change Summary: 200GB → 5TB/day

```
┌─────────────────────────────────────────────────────────────────────┐
│              200 GB/day → 5 TB/day MIGRATION PLAYBOOK               │
├───────────────────┬──────────────────┬──────────────────────────────┤
│ Layer             │ Before           │ After                        │
├───────────────────┼──────────────────┼──────────────────────────────┤
│ Ingestion         │ 10 Kinesis shards│ On-demand Kinesis / MSK      │
│ File format       │ JSON → Parquet   │ Iceberg + ZSTD               │
│ Partitioning      │ Daily            │ Hourly + event_type          │
│ Batch ETL         │ Glue 10 workers  │ EMR Spot + Glue hybrid       │
│ Job model         │ Monolithic daily │ Incremental hourly (25x/day) │
│ Spark tuning      │ Default settings │ AQE + 25K partitions         │
│ Storage lifecycle │ S3 Standard      │ Intelligent-Tiering + Glacier│
│ Compaction        │ None             │ Hourly Iceberg OPTIMIZE      │
│ Monitoring        │ Basic CloudWatch │ CloudWatch + custom metrics  │
└───────────────────┴──────────────────┴──────────────────────────────┘
```

---

---

# 🎯 Most Important System Design Questions & Answers for This Domain

---

## Q5: How Do You Ensure Exactly-Once Processing in Streaming Pipelines?

**Mnemonic: "ICK" — Idempotency + Checkpoints + Kafka offsets**

```
Challenge: Network failures, job restarts can cause duplicate data

Solution stack:
1. Idempotent writes:
   ├── Assign deterministic event IDs at source (UUID v5, not random)
   ├── Use MERGE (upsert) instead of INSERT in target tables
   └── If event_id already exists → skip, don't duplicate

2. Checkpointing (Flink/Spark):
   ├── Save processing state to S3 every 60 seconds
   ├── On restart: resume from last checkpoint
   └── Data between checkpoint and failure is reprocessed (idempotent = safe)

3. Kafka offset management:
   ├── Commit offsets ONLY after successful write to target
   ├── At-least-once delivery + idempotent writes = exactly-once semantics
   └── Never auto-commit offsets (risk of losing data on failure)

4. Transactional writes (Delta Lake / Iceberg):
   ├── Write to staging location first
   ├── Commit atomically (either all or nothing)
   └── Failed jobs leave no partial state
```

---

## Q6: How Do You Handle Late-Arriving Data in Streaming?

**Mnemonic: "WAW" — Watermarks + Allowed Lateness + Window re-computation**

```
Problem: Event happens at 2:00 PM but arrives at 2:15 PM
         → Window for 2:00 PM has already closed

Solution:

1. Watermarks (Flink):
   ├── Define: "accept events up to 10 minutes late"
   ├── Flink holds windows open until watermark advances past window end
   └── Events older than watermark are routed to side output for alerting

2. Allowed lateness window:
   ├── Emit partial result at window close (fast, approximate)
   ├── Re-emit corrected result when late data arrives
   └── Consumer uses latest emission (last-write-wins)

3. Lambda Architecture pattern:
   ├── Speed layer:  Real-time approximate result (Flink)
   ├── Batch layer:  Accurate historical result (Spark, includes late data)
   └── Serving layer: Merge speed + batch (serve batch when ready)

4. Store raw events with original event_time:
   ├── Bronze layer always has true event_time
   ├── Re-run batch jobs over corrected data window
   └── Correct dashboards with accurate numbers next day
```

---

## Q7: How Do You Design for Data Quality at Scale?

**Mnemonic: "VACV" — Validate + Alert + Quarantine + Version**

```
Data Quality Framework:

1. Validate at ingestion (Schema Registry):
   ├── Enforce schema at Kafka producer level
   ├── Reject malformed events before they enter Bronze
   └── Route to DLQ (Dead Letter Queue) for investigation

2. Quality checks at each Medallion layer:
   ├── Bronze → Silver: null checks, type checks, range validation
   ├── Silver → Gold:   referential integrity, business rule validation
   └── Use Great Expectations or AWS Glue Data Quality (DQ rules)
   
   Example DQ rules:
   ├── order_amount BETWEEN 0 AND 100000
   ├── customer_id NOT NULL
   ├── event_timestamp > '2020-01-01'
   └── order_status IN ('pending', 'shipped', 'delivered', 'cancelled')

3. Alert on anomalies:
   ├── Row count drop > 20% vs yesterday → PagerDuty alert
   ├── Null rate increase > 5% → Slack notification
   └── Schema change detected → halt pipeline, alert data owner

4. Quarantine bad records:
   ├── Don't fail the entire pipeline for 0.01% bad records
   ├── Route to quarantine/bad_records/ S3 prefix
   ├── Log rejection reason with record
   └── Review daily, backfill when source is fixed

5. Data versioning:
   ├── Never overwrite — always append or version
   ├── Iceberg time travel: revert to clean state if corruption found
   └── Audit log: who changed what, when, and why
```

---

## Q8: How Do You Manage Schema Evolution Without Breaking Downstream Consumers?

**Mnemonic: "ABCDE" — Additive + Backward-compatible + Contract + Deprecation + Evolution**

```
Rules for safe schema evolution:

ALLOWED (non-breaking):
├── Add new nullable column             ← safe for all consumers
├── Widen data type (INT → BIGINT)     ← safe
└── Add new partition value             ← safe

DANGEROUS (breaking):
├── Remove a column                     ← breaks consumers reading it
├── Rename a column                     ← breaks all queries using old name
├── Narrow a type (BIGINT → INT)       ← data loss risk
└── Change partition column             ← all existing paths break

Strategy:

1. Schema Registry (Kafka / Glue Schema Registry):
   ├── Enforce backward/forward compatibility rules
   ├── Reject schema changes that break consumers
   └── Version every schema change

2. Deprecation cycle:
   ├── New column added alongside old column
   ├── Consumers migrate to new column (30-day window)
   ├── Old column marked deprecated (still present)
   └── Old column removed after 90-day grace period

3. Contracts:
   ├── Each table has an owner and a published schema contract
   ├── Breaking changes require 30-day advance notice + migration guide
   └── Downstream teams sign off before breaking change deploys

4. Iceberg schema evolution:
   ├── Add/rename/delete columns in metadata only (no data rewrite)
   └── Old readers still work with old column IDs
```

---

## Q9: How Do You Optimize Cost for a Data Platform Running 24/7?

**Mnemonic: "STRIPE" — Spot + Tiering + Reserved + Incremental + Partition + Eliminate waste**

```
Cost Optimization Playbook:

S — Spot Instances for EMR batch:        60–80% savings
T — Tiering (S3 Intelligent-Tiering):   40–60% storage savings
R — Reserved capacity (Redshift 1-yr):  40% savings vs on-demand
I — Incremental processing:             Process only new data (90% savings)
P — Partition pruning + Parquet:        90% scan cost reduction (Athena)
E — Eliminate idle resources:
    ├── Terminate Glue dev endpoints when not in use
    ├── Auto-pause Redshift Serverless after 5 min idle
    └── Use Lambda for infrequent triggers, not always-on EC2

Cost Allocation:
├── Tag ALL resources by team/project/environment
├── AWS Cost Explorer dashboards per team
├── Monthly cost review with budget alerts at 80% threshold
└── Showback to product teams for accountability
```

---

## Q10: How Do You Handle Disaster Recovery for a Data Platform?

**Mnemonic: "RAIDR" — Replication + Automation + Isolation + Data backup + RTO/RPO**

```
DR Strategy for Data Platform:

Define targets first:
├── RPO (Recovery Point Objective): How much data loss is acceptable?
│   → Transactions: RPO = 0 (zero loss)
│   → Analytics: RPO = 4 hours (last batch run)
│
└── RTO (Recovery Time Objective): How fast must recovery happen?
    → Real-time pipeline: RTO = 15 minutes
    → Batch pipeline: RTO = 4 hours

Strategy by component:

1. S3 Data Lake:
   ├── Cross-Region Replication → secondary AWS region
   ├── S3 Versioning → recover from accidental deletes
   └── Glacier for long-term backup

2. Kinesis / MSK:
   ├── Multi-AZ by default (within-region HA)
   ├── For cross-region: MSK Replicator (fully managed Kafka replication)
   └── 7-day message retention = 7-day replay window

3. Glue Data Catalog:
   ├── Export catalog to S3 daily (Glue export API)
   ├── Recreate in secondary region from backup
   └── Terraform/CDK for infrastructure as code → fast rebuild

4. Redshift:
   ├── Automated snapshots (every 8 hours by default)
   ├── Cross-Region Snapshot Copy → secondary region
   └── Redshift Serverless: near-instant failover (no cluster to spin up)

5. Runbooks:
   ├── Document every recovery step
   ├── Test DR quarterly (chaos engineering)
   └── Automate recovery with Step Functions where possible
```

---

## 🏆 Scoring Rubric — What FAANG Interviewers Look For

```
┌─────────────────────────────────────────────────────────────────┐
│              FAANG SYSTEM DESIGN SCORING CRITERIA               │
├──────────────────────────┬──────────────────────────────────────┤
│ Criterion                │ What They Want to See                │
├──────────────────────────┼──────────────────────────────────────┤
│ Requirements Clarity     │ Clarify before designing             │
│ Back-of-envelope Math    │ Real numbers, not just "a lot"       │
│ Component Justification  │ WHY this tool, not just WHAT         │
│ Trade-off Awareness      │ Pros/cons for every decision         │
│ Scalability Thinking     │ Design for 10x from the start        │
│ Failure Modes            │ What breaks? How do you recover?     │
│ Cost Awareness           │ Optimize, don't just scale           │
│ Operational Simplicity   │ Fewer moving parts = less incidents  │
│ Security & Compliance    │ PII, encryption, access control      │
│ Communication            │ Structured, clear, whiteboard-ready  │
└──────────────────────────┴──────────────────────────────────────┘
```

---

## 📝 Final Cheat Sheet — All Mnemonics

```
┌─────────────────────────────────────────────────────────────────┐
│                    ALL MNEMONICS AT A GLANCE                    │
├────────────────────┬────────────────────────────────────────────┤
│ Mnemonic           │ Meaning                                    │
├────────────────────┼────────────────────────────────────────────┤
│ SCARED MOST        │ Framework for ANY system design answer     │
│ RICE BOWL          │ E-Commerce data platform design steps      │
│ KAF                │ Ingestion: Kinesis + API GW + Firehose     │
│ BSG                │ Storage: Bronze + Silver + Gold            │
│ GLASS              │ Processing tools selection                 │
│ RAQ                │ Analytics: Redshift + Athena + QuickSight  │
│ FTSD               │ ML: Features + Train + Serve + Drift       │
│ SHAPES             │ 10x scale-out strategy                     │
│ DIPS               │ Slow query fix: Diagnose+Index+Push+Serve  │
│ PIECES             │ 200GB → 5TB pipeline migration             │
│ ICK                │ Exactly-once: Idempotency+Checkpoints+Kafka│
│ WAW                │ Late data: Watermarks+Allowed+Window       │
│ VACV               │ Data quality: Validate+Alert+Quarantine+Ver│
│ ABCDE              │ Schema evolution safely                    │
│ STRIPE             │ Cost optimization                          │
│ RAIDR              │ Disaster recovery                          │
└────────────────────┴────────────────────────────────────────────┘
```

---

*Prepared by: Senior AWS Solutions Architect | FAANG-scale Data Engineering*  
*Last updated: March 2026 | Stack: AWS, Apache Spark, Kafka, Flink, Iceberg, Delta Lake, SageMaker*
