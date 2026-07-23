# Firehose Architecture Patterns & Anti-Patterns

## Overview
Reference guide for evaluating whether a Firehose streaming workflow follows proven architecture patterns or exhibits anti-patterns that indicate design issues.

## Common Architecture Patterns

### Pattern 1: Streaming ETL to Data Lake
```
Producers → Firehose → [Lambda Transform] → [Format Conversion: Parquet] → S3 (partitioned)
                                                                              ↓
                                                                         Athena / Spark
```

**When to use:** Building a queryable data lake from streaming data.

**Best practices:**
- Enable dynamic partitioning by time and/or business key
- Enable Parquet/ORC format conversion (saves 60-90% on query costs)
- Buffer size ≥ 64 MiB (required for format conversion)
- Buffer interval 300-900s (larger files = better query performance)
- Use Glue Data Catalog for schema management
- Enable compression (Snappy auto-enabled with Parquet)

**Anti-patterns:**
- ❌ Storing raw JSON without format conversion (expensive to query)
- ❌ Small buffer sizes creating thousands of tiny S3 objects
- ❌ No partitioning (full table scan on every query)
- ❌ Using Lambda for simple field extraction (use JQ instead)

---

### Pattern 2: Real-Time Event Streaming
```
Producers → Firehose (zero buffer) → HTTP Endpoint / Splunk / OpenSearch
                                          ↓
                                   Real-time dashboards / Alerts
```

**When to use:** Low-latency delivery to analytics or monitoring destinations.

**Best practices:**
- Zero or minimal buffer interval (0-60s)
- Small buffer size (1-5 MiB)
- GZIP content encoding for HTTP endpoints
- Retry duration configured for transient failures
- Monitor DataFreshness closely (SLA-critical)
- Backup bucket for failed deliveries

**Anti-patterns:**
- ❌ Large buffers with real-time SLA (unnecessary latency)
- ❌ No retry configuration (single failure = data loss)
- ❌ No DataFreshness alarm (stalls go undetected)
- ❌ Zero buffer on S3 backup destination (not supported)

---

### Pattern 3: Log Aggregation Pipeline
```
CloudWatch Logs → Subscription Filter → Firehose → S3 (compressed, partitioned by date)
                                                      ↓
                                                 Athena / CloudWatch Logs Insights
```

**When to use:** Centralizing logs from multiple sources into S3 for analysis.

**Best practices:**
- Buffer interval 300-900s (maximize file size)
- Buffer size 64-128 MiB (reduce S3 object count)
- Enable GZIP compression (logs compress 80-90%)
- Partition by date (year/month/day/hour) for efficient querying
- Enable CloudWatch Logs decompression if source is compressed

**Anti-patterns:**
- ❌ Buffer < 60s (creates excessive small files)
- ❌ No compression (wastes storage for highly compressible text)
- ❌ Flat S3 prefix (no partitioning → slow queries)
- ❌ No lifecycle policy on S3 destination (unbounded growth)

---

### Pattern 4: Multi-Destination Fan-Out
```
Producers → KDS → Firehose Stream A → S3 (archive)
                → Firehose Stream B → OpenSearch (search/alerts)
                → Firehose Stream C → Redshift (analytics)
```

**When to use:** Same data needs to reach multiple destinations with different configurations.

**Best practices:**
- Use Kinesis Data Streams as shared source (fan-out without duplicate PutRecord)
- Each Firehose stream tuned independently (buffer, transform, format)
- Monitor KinesisMillisBehindLatest on each stream
- Separate IAM roles per stream (least privilege)

**Anti-patterns:**
- ❌ Single stream trying to serve multiple use cases (conflicting buffer needs)
- ❌ Producers doing fan-out themselves (duplicate put calls, higher cost)
- ❌ Same Lambda transform for all destinations (one size doesn't fit all)
- ❌ No monitoring per-stream (one stream lagging goes undetected)

---

### Pattern 5: IoT / High-Volume Ingestion
```
IoT Devices → IoT Core Rules → Firehose → [Dynamic Partitioning by device_id] → S3
                                                                                    ↓
                                                                              Per-device analytics
```

**When to use:** High-cardinality data from many sources needing per-entity partitioning.

**Best practices:**
- Dynamic partitioning with appropriate key granularity
- Monitor PartitionCount (stay well under 500 limit)
- Consider partition key that limits active partition count (e.g., hour + device_region, not device_id)
- Use JQ for key extraction over Lambda (lower latency, lower cost)
- Plan for partition count scaling if device fleet grows

**Anti-patterns:**
- ❌ Partition key with unlimited cardinality (device_id with millions of devices → exceeds 500)
- ❌ No PartitionCountExceeded alarm (records silently go to error bucket)
- ❌ Hot partitions (one key gets 90% of traffic)
- ❌ Post-hoc partitioning with Glue/EMR when dynamic partitioning would work

---

### Pattern 6: Cross-Account / Cross-Region Delivery
```
Account A → Firehose → Cross-account S3 bucket (Account B)
                     → Cross-account OpenSearch (Account B)
```

**When to use:** Centralizing data from multiple accounts into a shared data lake or search cluster.

**Best practices:**
- Destination bucket policy explicitly grants Firehose role from source account
- Use resource-based policies (not just IAM role)
- Encryption key accessible from source account (KMS key policy)
- Monitor delivery in source account (metrics are published there)

**Anti-patterns:**
- ❌ Overly permissive bucket policy (`Principal: *`)
- ❌ No encryption on cross-account transfer
- ❌ No monitoring in source account (can't see delivery failures)

## Quick Assessment Matrix

Use this to quickly classify a streaming workflow:

| Question | Answer → Pattern |
|----------|-----------------|
| Is the destination S3 for analytics queries? | Pattern 1: Streaming ETL |
| Is sub-minute freshness required? | Pattern 2: Real-Time Event |
| Is the source CloudWatch Logs? | Pattern 3: Log Aggregation |
| Does data need to go to 2+ destinations? | Pattern 4: Fan-Out |
| Is ingestion from thousands of IoT devices? | Pattern 5: IoT Ingestion |
| Does data cross AWS account boundaries? | Pattern 6: Cross-Account |

## Universal Anti-Patterns (Apply to All)

| Anti-Pattern | Why It's Bad | Fix |
|--------------|-------------|-----|
| No backup bucket | Data loss on destination failure | Always configure backup S3 |
| Error logging disabled | Can't diagnose delivery failures | Enable CloudWatch Logs |
| Default buffer (never tuned) | Suboptimal for every use case | Tune per the pattern matrix |
| Monolithic Lambda transform | Single point of failure, hard to debug | Keep transforms simple, specific |
| No delimiters in records | Downstream consumers can't parse | Add `\n` between records |
| Same IAM role for everything | Blast radius too large | Separate roles per stream |
| No CloudWatch alarms | Failures go undetected for hours | Minimum: success, freshness, throttle |
| Ignoring FailedPutCount | Silent data loss at ingestion | Always check and retry failed records |
