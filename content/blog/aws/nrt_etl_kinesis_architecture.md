# Near-Real-Time ETL Architecture with Kinesis Data Streams

## Architecture Overview

A near-real-time ETL pipeline on AWS uses Kinesis Data Streams as the central ingestion bus, fanning out to three processing paths (hot, warm, cold) that all land in a Bronze → Silver → Gold S3 data lake. Every layer is independently scalable and observable.

---

## Pipeline Layers

### 1. Sources

| Source Type | Ingestion Mechanism | Partition Key Strategy |
|---|---|---|
| RDBMS (MySQL, Postgres, Oracle) | AWS DMS → Kinesis target endpoint | Primary key (ensures per-row ordering) |
| Application microservices | `PutRecords` SDK (batch 500 / 5 MB) | Domain entity ID (e.g. `order_id`) |
| IoT / Kafka | IoT Core rules action / MSK Connect | Device ID / topic-key passthrough |
| Clickstream / server logs | Kinesis Agent (file tail) | Session ID / host hash |

---

### 2. Ingestion — Kinesis Data Streams

- **Shards**: unit of throughput — 1 MB/s write, 2 MB/s read each
- **Partition keys**: MD5-hashed to a 128-bit shard range; hot-shard avoidance requires high-cardinality keys
- **Enhanced Fan-Out (EFO)**: each consumer application gets a dedicated 2 MB/s per shard via HTTP/2 push — essential when Databricks, Lambda, and Firehose all read the same stream
- **Retention**: 24h default, configurable up to 365 days — enables full pipeline replay
- **Sequence numbers**: monotonically increasing per shard — use for precise consumer checkpointing
- **Delivery guarantee**: at-least-once (consumers must deduplicate on `sequence_number` or `event_id`)

**Shard sizing formula:**
```
write_shards = ceil( (records/sec × avg_record_bytes) / 1,048,576 )
read_shards  = write_shards          # with EFO enabled
             = ceil( consumers × 2MB/s / 2,097,152 )   # without EFO
final_shards = max(write_shards, read_shards) × 1.2    # 20% headroom
```

---

### 3. Processing Paths (Fan-Out)

All three paths subscribe independently to the same KDS stream.

#### Hot Path — AWS Lambda
- **Latency target**: < 1 second end-to-end
- **Configuration**: Event Source Mapping with `ParallelizationFactor` 1–10 per shard
- **Reliability**: `BisectBatchOnFunctionError = true`, `MaximumRetryAttempts = 2`, SQS DLQ
- **Outputs**: DynamoDB (real-time state), SNS/EventBridge (alerts), ElastiCache (session store)
- **Use cases**: fraud signals, real-time session writes, push notifications, live inventory

#### Warm Path — Kinesis Data Analytics (Managed Apache Flink)
- **Latency target**: 1–30 seconds
- **Consumer type**: EFO subscription (dedicated throughput, no shared read limit)
- **State backend**: RocksDB with incremental checkpoints to S3
- **Operators**: tumbling windows (1 min), sliding windows (5 min), watermarks for late events
- **Output sink**: `StreamingFileSink` → S3 Silver zone (Parquet + Snappy)
- **Use cases**: windowed aggregations, CEP fraud rules, anomaly detection, enriched event streams

#### Cold Path — Kinesis Data Firehose
- **Latency target**: 60 seconds – 5 minutes (buffer hint: 128 MB or 300s)
- **Format conversion**: JSON → Parquet using Glue schema (inline, no separate ETL job)
- **Dynamic partitioning**: `!{partitionKeyFromQuery:event_type}` for field-based S3 prefixes
- **Inline Lambda transform**: field normalization, PII masking before landing
- **Output**: S3 Bronze zone
- **Use cases**: raw archival, replay buffer, compliance retention

---

### 4. S3 Data Lake — Bronze / Silver / Gold Zones

#### Bronze (Raw Zone)
- **Written by**: Kinesis Firehose
- **Format**: JSON or Avro (as-is from stream, no transformation)
- **Partitioning**: `s3://bucket/bronze/{source}/{year}/{month}/{day}/{hour}/`
- **Schema enforcement**: none — raw landing zone
- **Latency to availability**: 1–5 minutes
- **Retention**: long-term (S3 Intelligent Tiering, lifecycle to Glacier after 90d)

#### Silver (Curated Zone)
- **Written by**: Glue ETL job (Bronze → Silver)
- **Format**: Parquet + Snappy
- **Partitioning**: `s3://bucket/silver/{domain}/{entity}/{year}/{month}/{day}/`
- **Transformations applied**:
  - Schema projection (drop internal metadata fields)
  - Deduplication on `(event_id, event_ts)`
  - Type coercion (timestamps, decimals, enums)
  - Null handling per column policy
- **Job bookmarks**: enabled (incremental processing, no re-read of processed prefixes)
- **Catalog update**: Glue Crawler runs after each job to register new partitions

#### Gold (Enriched / Analytics Zone)
- **Written by**: Databricks Auto Loader (or EMR Spark)
- **Format**: Delta Lake (ACID, time travel, schema evolution)
- **Partitioning**: optimized per query pattern; Z-order on `(event_type, customer_id)`
- **Transformations applied**:
  - Surrogate key generation
  - SCD Type 2 dimension merges (`MERGE INTO`)
  - Business-level aggregations (daily/hourly fact tables)
  - PII tokenization / masking
- **Latency to availability**: 5–15 minutes from event source
- **Consumers**: Athena, Redshift Spectrum, Databricks SQL, QuickSight

---

### 5. Transform & Catalog

#### AWS Glue ETL (Bronze → Silver)
```python
# Core pattern — Bronze to Silver Glue job
glueContext.create_dynamic_frame_from_catalog(
    database="raw_db",
    table_name="bronze_events",
    transformation_ctx="bronze_source"
)
# Apply schema, dedup, type coerce
# Write with job bookmark
glueContext.write_dynamic_frame_from_options(
    frame=silver_frame,
    connection_type="s3",
    format="parquet",
    connection_options={"path": "s3://bucket/silver/events/"},
    transformation_ctx="silver_sink"
)
```
- Worker type: G.1X (4 vCPU, 16 GB) — scale DPUs with daily volume
- Trigger: EventBridge scheduled rule or S3 event notification
- Monitoring: CloudWatch metric `glue.driver.aggregate.numFailedTasks`

#### Databricks Auto Loader (Silver → Gold)
```python
# Trigger: availableNow (batch micro-job, not continuous)
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "parquet")
    .option("cloudFiles.schemaLocation", "/checkpoints/silver_schema")
    .load("s3://bucket/silver/events/"))

(df.writeStream
    .trigger(availableNow=True)
    .format("delta")
    .option("checkpointLocation", "/checkpoints/gold_events")
    .option("mergeSchema", "true")
    .outputMode("append")
    .table("analytics_db.gold_events"))
```

#### Glue Data Catalog + Lake Formation
- **One catalog, three databases**: `raw_db` (Bronze), `curated_db` (Silver), `analytics_db` (Gold)
- **Glue Crawlers**: auto-discover partitions and update table schemas after each ETL run
- **Lake Formation**: column-level grants — PII columns accessible only to privileged roles; masked views for BI roles
- **Shared by**: Athena, Redshift Spectrum, Databricks (external Hive metastore or Unity Catalog), EMR Spark

---

### 6. Consumption Layer

| Query Engine | Zone Queried | Typical Use Case |
|---|---|---|
| Amazon Athena | Silver, Gold | Ad-hoc SQL, data exploration, data quality checks |
| Redshift Spectrum | Gold | Join warehouse native tables with lake external tables |
| Databricks SQL | Gold (Delta) | High-concurrency BI, photon-accelerated aggregations |
| QuickSight (SPICE) | Gold via Athena | Executive dashboards, scheduled refresh every 15 min |

---

### 7. Orchestration

#### AWS Step Functions (recommended)
```
GlueStartJobRun (Bronze→Silver)
    → GlueCrawlerStart
    → EMR/DatabricksJobRun (Silver→Gold)
    → AthenaStartQueryExecution (data quality validation)
    → SNS.Publish (success / failure notification)
```

#### MWAA (Airflow) — alternative for complex DAG dependencies
- Use when pipeline has cross-domain dependencies or external system triggers
- Sensor operators for S3 prefix readiness before triggering downstream jobs

---

### 8. Observability & Alerting

| Metric | Threshold | Action |
|---|---|---|
| `GetRecords.IteratorAgeMilliseconds` | > 60,000 ms | Consumer falling behind — scale Lambda parallelism or add shards |
| `WriteProvisionedThroughputExceeded` | > 0 | Hot shard — review partition key cardinality or split shards |
| `Firehose.DeliveryToS3.DataFreshness` | > 900s | Firehose buffer stall — check Lambda transform errors |
| `Glue.driver.aggregate.numFailedTasks` | > 0 | ETL job failure — inspect CloudWatch Logs, check DQ rules |
| `KDA.millisBehindLatest` | > 30,000 ms | Flink job falling behind — scale KPU count |
| Databricks job run status | FAILED | Auto Loader failure — check stream read checkpoint, shard throttle |

All alarms publish to an SNS topic → PagerDuty or Slack webhook.

---

### 9. Databricks-Specific: Direct KDS Read (Bypass Firehose)

For use cases where end-to-end latency to Gold must be under 5 minutes, Databricks can read KDS directly using the `spark-sql-kinesis` connector:

```python
kinesisDF = (spark.readStream
    .format("aws-kinesis")
    .option("kinesis.region", "us-east-1")
    .option("kinesis.streamName", "prod-events-stream")
    .option("kinesis.consumerType", "EFO")  # dedicated 2 MB/s per shard
    .option("kinesis.efo.consumerName", "databricks-gold-consumer")
    .option("kinesis.startingPosition", "LATEST")
    .option("kinesis.checkpointLocation", "s3://checkpoints/kinesis/")
    .load())
```

**Key configuration decisions**:
- Use `EFO` consumer type — avoids `ReadProvisionedThroughputExceeded` when Lambda and Firehose also read the stream
- Checkpoint shard sequence numbers to S3 (not DBFS) for DR resilience
- Use `TRIM_HORIZON` for backfill / replay runs, `LATEST` for ongoing streaming
- Set `trigger(processingTime="30 seconds")` for near-real-time micro-batch; `trigger(availableNow=True)` for scheduled batch-style runs

---

### 10. Latency Summary

| Path | Components | Typical End-to-End Latency |
|---|---|---|
| Hot | KDS → Lambda → DynamoDB | < 1 second |
| Warm | KDS → KDA Flink → S3 Silver | 1–30 seconds |
| Cold (Firehose) | KDS → Firehose → S3 Bronze | 1–5 minutes |
| Bronze → Silver | S3 Bronze → Glue ETL → S3 Silver | 5–15 minutes |
| Silver → Gold | S3 Silver → Databricks/EMR → Delta Gold | 5–20 minutes |
| **Total (source → Gold)** | | **~15–30 minutes** |

> For sub-5-minute Gold latency: use Databricks direct KDS read with EFO, bypassing Bronze/Silver intermediate zones.
