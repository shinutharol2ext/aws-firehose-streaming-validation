# Streaming Workflow Evaluation

## Overview
Evaluate the overall architecture and design quality of an Amazon Data Firehose streaming workflow. Goes beyond health checks to assess whether the pipeline is optimally designed for the use case — covering throughput capacity, cost efficiency, data quality, and scalability.

## Evaluation Framework

Score each dimension 1-5 and provide recommendations:


| Dimension | What to Evaluate |
|-----------|-----------------|
| Throughput Design | Is the pipeline sized for peak load + growth? |
| Latency Optimization | Is buffering aligned with freshness requirements? |
| Cost Efficiency | Is the pipeline minimizing unnecessary costs? |
| Data Quality | Are transformations, formats, and partitioning correct? |
| Resilience | Can the pipeline handle failures without data loss? |
| Scalability | Can the pipeline grow without redesign? |
| Security Posture | Is data protected in transit and at rest? |
| Observability | Can operators detect and diagnose issues? |

## Dimension 1: Throughput Design

### Quotas & Limits Reference

| Limit | Value | Notes |
|-------|-------|-------|
| Record size (Direct PUT) | 1,000 KB (before Base64) | Cannot be changed |
| PutRecordBatch max records | 500 per request | Cannot be changed |
| PutRecordBatch max payload | 4 MB per request | Cannot be changed |
| Record size (MSK source, no Lambda) | 10 MB max | |
| Record size (MSK source, with Lambda) | 6 MB max | Lambda caps at 6 MB; >6 MB goes to error bucket |
| Dynamic partitioning active partitions | 500 (default) | Can be increased |
| Multi-record de-aggregation | Max 500 records | >500 succeeds but de-aggregation breaks |

### Evaluation Questions
1. What is the peak ingestion rate (MB/s, records/s)?
2. Is `BytesPerSecondLimit` sufficiently above peak? (headroom > 20%)
3. Is `RecordsPerSecondLimit` sufficiently above peak?
4. Are producers using PutRecordBatch (not individual PutRecord) for throughput?
5. Are producers implementing exponential backoff for throttled responses?
6. Is PutRecordBatch response `FailedPutCount` being checked and retried?

### Red Flags
- ThrottledRecords consistently > 0
- IncomingBytes close to BytesPerSecondLimit (< 20% headroom)
- Single PutRecord calls instead of batching
- No retry logic in producer code
- Record sizes near 1 MB limit (fragile to growth)

## Dimension 2: Latency Optimization

### Buffering Decision Matrix

| Use Case | Buffer Interval | Buffer Size | Rationale |
|----------|-----------------|-------------|-----------|
| Real-time analytics | 0 seconds | 1 MiB | Minimum latency (application destinations only) |
| Near-real-time dashboards | 60 seconds | 1-5 MiB | Balance cost vs. freshness |
| Log aggregation | 300 seconds | 64-128 MiB | Larger files, fewer S3 objects |
| Data lake ETL | 300-900 seconds | 128 MiB | Maximize object size for Athena/Spark |
| Format conversion (Parquet/ORC) | 300+ seconds | 64-128 MiB (min 64 MiB required) | Must be ≥ 64 MiB for format conversion |

### Evaluation Questions
1. What is the required data freshness SLA?
2. Does DataFreshness metric match the SLA?
3. Are buffer hints aligned with the use case (see matrix above)?
4. Is transformation buffer adding unnecessary latency?
5. For zero-buffering: is this an application destination? (not S3 backup, not dynamic partitioning)

### Red Flags
- DataFreshness >> buffer interval (delivery is stalling)
- Real-time use case with 5-min buffer (default not tuned)
- Zero buffering on destinations that don't support it
- Buffer < 60s for S3 → multi-part uploads → many small files

## Dimension 3: Cost Efficiency

### Cost Drivers
1. **Data ingestion** — charged per GB ingested (first 500 TB: $0.029/GB in us-east-1)
2. **Format conversion** — additional $0.018/GB for Parquet/ORC conversion
3. **Dynamic partitioning** — $0.020/GB for JQ processing
4. **VPC delivery** — additional charges for VPC-enabled delivery
5. **Lambda transformation** — Lambda invocation + duration costs
6. **Destination costs** — S3 PUT requests, Redshift COPY, OpenSearch indexing

### Evaluation Questions
1. Is compression enabled? (GZIP, Snappy, or ZIP reduce storage + transfer costs)
2. Are there unnecessary Lambda invocations? (can JQ or format conversion replace Lambda?)
3. Is dynamic partitioning creating too many small files? (downstream query cost)
4. Is format conversion to Parquet/ORC used for analytics workloads? (saves 60-90% on S3 queries)
5. Are buffer sizes large enough to avoid excessive S3 PutObject calls?
6. Is the stream over-provisioned for actual throughput?

### Cost Optimization Checklist
- [ ] Enable GZIP/Snappy compression for S3 destinations
- [ ] Use Parquet/ORC format conversion instead of raw JSON for analytics
- [ ] Replace simple Lambda transforms with built-in JQ processing where possible
- [ ] Increase buffer size to reduce S3 PUT request count
- [ ] Use dynamic partitioning instead of post-processing ETL for partitioned data
- [ ] Remove unused/idle delivery streams

### Red Flags
- JSON stored in S3 for Athena queries (should be Parquet)
- Lambda doing simple field extraction (JQ can do this natively)
- Buffer = 1 MiB for batch analytics (creates too many small S3 files)
- Compression disabled for S3 destination
- Multiple streams where one with dynamic partitioning would suffice

## Dimension 4: Data Quality

### Format Conversion Validation
When Parquet/ORC format conversion is enabled:
- Buffer size MUST be ≥ 64 MiB (default changes to 128 MiB)
- Only S3 destinations support format conversion
- Only JSON input → Parquet or ORC output supported
- Requires Glue Data Catalog table/schema definition
- Input parsers: OpenX JSON or Hive JSON

### Dynamic Partitioning Validation
- Partition keys extracted correctly (JQ expressions or Lambda metadata)
- S3 prefix expressions using `!{partitionKey:key_name}` syntax
- Active partition count well below 500 limit
- PartitionCountExceeded metric = 0 (otherwise records go to error bucket)
- Partitions distribute evenly (avoid hot partitions)

### Data Delimiter Check
For Direct PUT → S3 without format conversion:
- Records MUST include delimiters (e.g., `\n` newline) for downstream consumers to parse
- Without delimiters, S3 objects contain concatenated blobs that are unparseable

### Evaluation Questions
1. Is the schema definition (Glue catalog) up to date with the data?
2. Are partition keys producing even distribution?
3. Is format conversion schema matching actual incoming data?
4. Are records delimited for downstream parsability?
5. Are `FailedConversion.Records` or `FailedConversion.Bytes` > 0?

## Dimension 5: Resilience

### Firehose Built-in Resilience
- Firehose retries failed deliveries automatically
- Data in-flight is stored across multiple AZs
- On persistent destination failure → data goes to backup S3 bucket

### Evaluation Questions
1. Is backup S3 bucket configured and accessible?
2. Is backup mode appropriate? (all records vs. failed only)
3. For Lambda: are retries configured? (default 3, max 9)
4. For Lambda: is the function idempotent? (retries = duplicate processing risk)
5. Is there a dead-letter/error path for unprocessable records?
6. What happens if the destination is down for hours? (backup bucket fills up)
7. For KDS source: what happens if the stream is resharded? (Firehose adapts automatically)

### Red Flags
- No backup bucket configured
- Backup bucket in same account/region as primary (shared blast radius)
- Lambda not idempotent but retries > 0
- No alarm on BackupToS3.Records > 0 (failures happening silently)
- No alarm on DataFreshness exceeding SLA

## Dimension 6: Scalability

### Firehose Auto-Scaling Behavior
- Direct PUT streams auto-scale throughput when sustained throttling occurs
- No manual scaling needed for most use cases
- For KDS/MSK sources: throughput limited by source shard/partition count

### Evaluation Questions
1. Is throughput quota sufficient for projected 12-month growth?
2. If using KDS source: does shard count support future volume?
3. If using MSK source: does partition count support future volume?
4. Will dynamic partitioning stay under 500 active partitions at peak?
5. Will Lambda concurrency support peak transformation load?
6. Will the destination handle increased throughput? (OpenSearch cluster, Redshift slots)

### Red Flags
- At >80% of BytesPerSecondLimit with growth expected
- Dynamic partitioning approaching 500 partition limit
- Lambda at reserved concurrency limit
- Destination not sized for projected growth (e.g., small OpenSearch cluster)

## Dimension 7: Security Posture

### Evaluation Questions
1. Is server-side encryption enabled? (SSE-KMS recommended over SSE-S3)
2. Is the KMS key customer-managed? (allows key rotation, granular access)
3. Is the IAM role following least privilege? (only required actions on specific resources)
4. Are VPC endpoints used if Firehose writes to VPC-only destinations?
5. Is CloudTrail logging API calls for audit?
6. Are tags applied for cost allocation and access control?

### Red Flags
- No encryption enabled
- IAM role with `firehose:*` or `s3:*` (overly broad)
- Same IAM role used across multiple unrelated streams
- No resource-based policies on destination (S3 bucket policy, OS domain policy)

## Dimension 8: Observability

### Minimum Viable Observability
Every Firehose stream should have:
1. **CloudWatch error logging enabled** (strongly recommended per AWS docs)
2. **Alarms on delivery success** — `DeliveryTo<Dest>.Success` = 0
3. **Alarms on data freshness** — DataFreshness > SLA
4. **Alarms on throttling** — ThrottledRecords > 0
5. **Dashboard** — IncomingRecords, DataFreshness, Success metrics

### Advanced Observability
- Alarm on incoming = 0 (producer failure detection)
- Alarm on BackupToS3.Records > 0 (silent failures)
- Alarm on BytesPerSecondLimit approaching IncomingBytes
- For Lambda: alarm on ExecuteProcessing.Duration increasing
- For dynamic partitioning: alarm on PartitionCountExceeded
- Cross-account CloudWatch dashboard for multi-stream visibility

### Red Flags
- Error logging disabled
- No CloudWatch alarms at all
- Only monitoring ingestion, not delivery success
- No DataFreshness alarm (delivery can stall silently)

## Evaluation Report Template

```markdown
# Firehose Workflow Evaluation: [Stream Name]

## Summary
| Dimension | Score (1-5) | Status |
|-----------|-------------|--------|
| Throughput Design | X/5 | 🟢/🟡/🔴 |
| Latency Optimization | X/5 | 🟢/🟡/🔴 |
| Cost Efficiency | X/5 | 🟢/🟡/🔴 |
| Data Quality | X/5 | 🟢/🟡/🔴 |
| Resilience | X/5 | 🟢/🟡/🔴 |
| Scalability | X/5 | 🟢/🟡/🔴 |
| Security Posture | X/5 | 🟢/🟡/🔴 |
| Observability | X/5 | 🟢/🟡/🔴 |
| **Overall** | **X/40** | |

## Top Recommendations (Priority Order)
1. [Critical] ...
2. [High] ...
3. [Medium] ...

## Detailed Findings
### [Dimension]
- Finding: ...
- Evidence: ...
- Recommendation: ...
- Effort: Low/Medium/High
```
