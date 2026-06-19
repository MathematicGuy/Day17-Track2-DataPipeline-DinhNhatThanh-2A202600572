# Design Document: Credit Scoring & Fraud Detection Feature Pipeline (Vietnamese E-commerce)

This document outlines the architecture and tradeoffs for a real-time and batch-integrated feature pipeline designed for credit scoring and fraud detection in a high-throughput Vietnamese e-commerce and digital wallet ecosystem.

---

## 1. Problem Statement & Constraints

The system supports a digital checkout ecosystem (similar to Tiki/Shopee with ZaloPay) processing millions of transactions daily. 

### Key Constraints:
- **Low Latency**: Real-time fraud detection must run inline with transactions and evaluate in under **50ms** during checkout.
- **Scale**: Must handle 10 million daily active users (DAU), peaking at **15,000 transactions per second (TPS)** during mega-sale events (e.g., 11/11, 12/12).
- **Data Compliance**: Strictly adhere to Vietnam's **Personal Data Protection Decree (Decree 13/2023/ND-CP - PDPL)**. This requires user consent mapping, data localization within Vietnam, and satisfying the "Right to be Forgotten" (physical deletion of user data within 72 hours of request).
- **Training-Serving Skew & Leakage**: Credit scoring models are highly sensitive to features like "number of transactions in the last 24 hours" and "current balance". Standard joins leak future transaction details into training datasets, resulting in catastrophic model failure in production.

---

## 2. Architecture Sketch

The following ASCII diagram illustrates the hybrid Kappa-like architecture:

```
                                +-------------------+
                                | Transaction Event |
                                +---------+---------+
                                          |
                     +--------------------+--------------------+
                     | (Real-time Stream)                      | (Batch Ingestion)
                     v                                         v
             +-------+-------+                         +-------+-------+
             |   Redpanda    |                         |  DB Replicas  |
             +-------+-------+                         +-------+-------+
                     |                                         |
                     v (Low Latency < 5ms)                     v (Daily ELT)
             +-------+-------+                         +-------+-------+
             | Apache Flink  |                         |  DuckDB / dbt |
             +-------+-------+                         +-------+-------+
                     |                                         |
                     v (Feature Write)                         v (ASOF Temp Joins)
             +-------+-------+                         +-------+-------+
             | Feast Online  |                         | Iceberg Lake  |
             |    (Redis)    |                         | (S3-Compat)   |
             +-------+-------+                         +-------+-------+
                     |                                         |
                     v (< 20ms Lookup)                         v (Offline Train)
             +-------+-------+                         +-------+-------+
             | Model Serving |                         | Model Train   |
             |  (Fraud API)  |<------------------------|  (Day 22 SFT) |
             +---------------+                         +---------------+
```

---

## 3. Core Decisions & Tradeoffs

### Decision 1: Batch vs. Streaming Feature Ingestion (Freshness vs. Cost)
* **Decision**: Hybrid approach (Kappa Architecture). Fast-moving behavioral indicators (e.g., "number of failed checkouts in the last 5 minutes") are computed via **Apache Flink** streaming. Slow-moving aggregations (e.g., "average transaction size over the last 90 days") are computed via **nightly batch SQL** on our Apache Iceberg lakehouse.
* **Tradeoff**: Streaming provides real-time freshness which is essential to stop immediate fraud attacks (e.g., credential stuffing or card testing loops). However, streaming is computationally expensive and hard to backfill. Batch is much cheaper and highly optimized but has a 24-hour lag. We balance this by maintaining real-time variables only for critical short-term risk metrics and offloading historical statistics to batch.

### Decision 2: Prevention of Future Leakage (ASOF Point-in-Time Joins)
* **Decision**: Utilize DuckDB/SQL's **`ASOF JOIN`** in the offline training pipeline, matching every historical transaction event with the exact state of features as they existed *at or before* that event's timestamp.
* **Tradeoff**: Naive "Latest State" Joins vs. Temporal ASOF Joins. Naive joins are simple to write but lead to severe future leakage (e.g. training data knows the user's credit score dropped *after* they committed fraud). While ASOF joins require maintaining a complete, versioned history log of all feature updates (which increases storage requirements by roughly 6x), it guarantees 100% training-serving parity.

### Decision 3: PDPL (Decree 13/2023/ND-CP) Deletion Compliance
* **Decision**: Implement a **Soft-Delete Metadata Layer** combined with **Tiered Asynchronous Purges**. Upon a user's deletion request, we immediately write a tombstone record to our Feast Online Store (Redis) and Iceberg tables, instantly masking the user from the inference engine. Physically, an offline compaction job runs weekly to rewrite S3 Parquet files and purge deleted user records from disk.
* **Tradeoff**: Instant Disk Erasure vs. Asynchronous Purges. Instant physical erasure of historical files causes massive write amplification and risks locking our data lake during peak TPS. Asynchronous purge complies with the 72-hour window allowed under PDPL, minimizing operational costs while remaining 100% compliant.

### Decision 4: Failure Semantics and Idempotent Backfills
* **Decision**: All batch pipelines write to **partitioned Apache Iceberg tables** keyed by transaction date. In the event of a pipeline crash, we perform a partition overwrite. Downstream consumers use deterministic hash keys `MD5(user_id || event_timestamp)` as unique event IDs to prevent duplicate record processing.
* **Tradeoff**: Partition Overwrite vs. Append-and-Dedup. Append-and-dedup is lightweight on writes but causes heavy reads and deduplication latency downstream. Partition overwrite is slower on failure recovery but guarantees that running a backfill multiple times is 100% idempotent with no duplicate rows.

### Decision 5: Infrastructure Localization and Bandwidth (Vietnamese Context)
* **Decision**: Deploy the real-time serving cluster (Model Service, Redpanda, Redis) in **local Vietnamese Data Centers** (VNPT / Viettel IDC) rather than global cloud regions (e.g., AWS Singapore `ap-southeast-1`).
* **Tradeoff**: Local IDC vs. Global Cloud. AWS Singapore offers superior managed infrastructure but incurs a **30-40ms round-trip network latency** over international fiber, making it impossible to meet our 50ms checkout constraint. Additionally, Vietnamese Decree 53/2022/ND-CP mandates storing customer transactional history within domestic borders. We trade off cloud convenience for legal compliance and low latency.

---

## 5. Rejected Alternative: Pure Lambda Architecture

We explicitly rejected a **traditional Lambda architecture** where identical features are defined in two separate codebases (e.g., Scala code for streaming Spark, and SQL code for batch dbt). 

### Reason for Rejection:
Maintaining two separate codebases for the same logic inevitably leads to **training-serving feature drift**. If a developer updates the windowing logic or timezone offset (e.g., handling UTC vs. GMT+7 in Vietnam) in the batch codebase but forgets to port it to the streaming codebase, the features generated during training will not match the features generated during real-time inference. This discrepancy silently degrades model performance. Instead, we use a single feature registry (Feast) and SQL-centric definitions that are run across both engines.
