# End-to-End Data Flow Validation

## Overview
Comprehensive validation of the entire Firehose pipeline from source ingestion through to destination delivery. Use this workflow to confirm data integrity, measure end-to-end latency, and validate the complete pipeline is functioning correctly.

## End-to-End Validation Workflow

### Phase 1: Source Ingestion Verification

**Confirm data is entering the stream:**

| Source Type | What to Check | Key Metric |
|-------------|---------------|------------|
| Direct PUT | PutRecord/PutRecordBatch API calls succeeding | `IncomingRecords`, `IncomingPutRequests` |
| Kinesis Data Streams | Firehose reading from all shards | `DataReadFromKinesisStream.Records` |
| MSK | Firehose consuming from all partitions | `DataReadFromSource.Records` |

**Validation:**
- `IncomingRecords > 0` — data is arriving
- `IncomingBytes > 0` — data has content
- `ThrottledRecords = 0` — no capacity issues

If `IncomingRecords = 0` with Direct PUT: confirm PutRecord/PutRecordBatch calls are executing correctly in the producer application.

### Phase 2: Transformation Verification (if enabled)

**Confirm transformation is processing:**
- `ExecuteProcessing.Success = 1` — Lambda invocations succeeding
- `SucceedProcessing.Records > 0` — records being transformed

**Compare input to output:**
- `IncomingRecords` (input) vs `SucceedProcessing.Records` (transformed)
- Any gap = records with `ProcessingFailed` or `Dropped` status
- Check backup S3 for failed records (if backup enabled)

**Latency contribution:**
- Transformation buffer time (default 60 seconds) + Lambda execution time

### Phase 3: Destination Delivery Verification

**Confirm data is being delivered:**
- `DeliveryTo<Destination>.Success > 0` — deliveries succeeding
- `DeliveryTo<Destination>.Records > 0` — records reaching destination
- `DeliveryTo<Destination>.Bytes > 0` — data reaching destination

**If delivery metrics are zero but ingestion is positive:**
1. Check `DataFreshness` — is it growing? (delivery stalled)
2. Check CloudWatch Logs for error messages
3. Check IAM role permissions
4. Check destination accessibility

### Phase 4: Data Integrity Verification

**Record count reconciliation:**
```
Expected flow:
IncomingRecords → (minus Dropped) → DeliveryTo<Destination>.Records + BackupToS3.Records
```

| Scenario | Expected Relationship |
|----------|----------------------|
| No transformation | IncomingRecords ≈ Delivery.Records (within buffer window) |
| With transformation, no drops | IncomingRecords ≈ SucceedProcessing.Records ≈ Delivery.Records |
| With transformation + drops | IncomingRecords > SucceedProcessing.Records (Dropped excluded from failures) |
| With failures | IncomingRecords = Delivery.Records + BackupToS3.Records (if backup enabled) |

**Data content verification:**
1. Send a known test record with a unique identifier
2. Wait for buffer interval to elapse
3. Check destination for the test record
4. Verify data content matches (accounting for any transformation)

### Phase 5: Latency Measurement

**End-to-end latency components:**

```
Total Latency = Source Buffer + Transform Buffer + Transform Execution + Destination Buffer + Delivery Time
```

| Component | Metric / Setting | Typical Range |
|-----------|-----------------|---------------|
| Source buffer (KDS/MSK) | N/A (Firehose reads continuously) | 0–few seconds |
| Transform buffer | Transform BufferIntervalInSeconds | 0–900s (default 60s) |
| Transform execution | `ExecuteProcessing.Duration` | 100ms–5min |
| Destination buffer | BufferIntervalInSeconds | 0–900s (default 300s) |
| Delivery time | `DataFreshness` - buffer settings | Varies by destination |

**DataFreshness interpretation:**
- DataFreshness = age of the OLDEST record still in Firehose
- Normal: DataFreshness ≈ buffer interval (records delivered on schedule)
- Problem: DataFreshness >> buffer interval (delivery stalled or failing)

### Phase 6: Dynamic Partitioning Validation (if enabled)

**Metrics to check:**
- `PartitionCount` — active partitions being processed (max 500 default)
- `PartitionCountExceeded` — 1 = you've hit the partition limit
- `PerPartitionThroughput` — bytes/sec per partition
- `DeliveryToS3.ObjectCount` — objects delivered (should reflect partitions)
- `JQProcessing.Duration` — time for JQ expression evaluation

**Common issues:**
- Partition count exceeded → records go to error bucket
- High JQ duration → complex partitioning expressions
- Uneven partition throughput → data skew across partition keys

## Complete Validation Checklist

### Pre-Flight
- [ ] Stream status is ACTIVE
- [ ] Error logging is enabled (CloudWatch Logs)
- [ ] IAM role has correct trust policy and permissions
- [ ] Server-side encryption is configured
- [ ] Backup bucket is configured and accessible

### Ingestion
- [ ] IncomingRecords > 0
- [ ] IncomingBytes > 0
- [ ] ThrottledRecords = 0
- [ ] Source-specific metrics healthy (KDS lag, MSK offset lag)

### Transformation (if enabled)
- [ ] Lambda function is healthy (no errors in Lambda logs)
- [ ] ExecuteProcessing.Success = 1
- [ ] SucceedProcessing.Records tracking IncomingRecords
- [ ] No recordId mismatches in CloudWatch Logs
- [ ] Lambda timeout > processing time for max batch

### Delivery
- [ ] DeliveryTo<Destination>.Success > 0
- [ ] DataFreshness within expected range (≈ buffer interval)
- [ ] No auth failures or delivery rejections
- [ ] Backup bucket receiving failed records (if applicable)

### Alarms
- [ ] IncomingRecords = 0 alarm (no data flowing)
- [ ] ThrottledRecords > 0 alarm (capacity issues)
- [ ] DataFreshness > threshold alarm (delivery stalled)
- [ ] DeliverySuccess = 0 alarm (delivery failures)
- [ ] Lambda errors alarm (if transformation enabled)

## Troubleshooting Decision Tree

```
Is IncomingRecords > 0?
├── NO → Problem is upstream (producer not sending data)
│         Check: PutRecord API calls, KDS shard health, MSK topic
│
└── YES → Is DeliveryTo<Dest>.Success > 0?
          ├── NO → Is transformation enabled?
          │         ├── YES → Is ExecuteProcessing.Success = 1?
          │         │         ├── NO → Lambda is failing. Check Lambda logs.
          │         │         └── YES → Destination is unreachable. Check IAM + network.
          │         └── NO → Destination is unreachable. Check IAM + network.
          │
          └── YES → Is DataFreshness growing?
                    ├── YES → Delivery is slower than ingestion. Scale destination.
                    └── NO → Pipeline is healthy. ✓
```

## Test Record Injection

To validate end-to-end with a test record:

```bash
# Generate a test record
TEST_DATA=$(echo -n '{"test_id":"validation-'$(date +%s)'","message":"end-to-end-test"}' | base64)

# Put test record
aws firehose put-record \
  --delivery-stream-name <STREAM_NAME> \
  --record Data="$TEST_DATA"

# Wait for buffer interval + processing time
echo "Waiting for delivery (buffer interval + processing)..."
sleep <BUFFER_INTERVAL_SECONDS + 30>

# Check destination for the test record
# (destination-specific command here)
```

**Cleanup:** Remove test records from destination after validation to avoid polluting production data.
