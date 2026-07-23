# Delivery Stream Health Validation

## Overview
Validate the health of an Amazon Data Firehose delivery stream by checking ingestion metrics, buffering configuration, throttling, and overall stream status.

## Validation Steps

### 1. Stream Configuration Check
Describe the delivery stream and verify:
```bash

aws firehose describe-delivery-stream --delivery-stream-name <STREAM_NAME>
```

Validate:
- **Stream status**: Must be `ACTIVE` (not `CREATING`, `DELETING`, or `CREATING_FAILED`)
- **Source type**: Confirm source matches expected (DirectPut, KinesisStreamAsSource, MSKAsSource)
- **Encryption**: Verify server-side encryption is enabled (`KeyType: AWS_OWNED_CMK` or `CUSTOMER_MANAGED_CMK`)
- **Error logging**: CloudWatch error logging MUST be enabled (strongly recommended per AWS docs)

### 2. Buffering Configuration Validation

**Buffer defaults:**
- Size: 5 MiB
- Interval: 5 minutes (300 seconds)

**Key rules:**
- Buffer size is measured in MBs, interval in seconds
- The FIRST buffer condition satisfied triggers delivery (size OR time)
- Zero buffering (interval = 0) is available for application destinations but NOT for:
  - S3 backup destination
  - Dynamic partitioning
- When buffer interval < 60 seconds, Firehose uses multi-part upload for S3
- Buffer hints are applied at shard/partition level; dynamic partitioning buffer hints are at stream/topic level

**Validation checks:**
- Is buffer interval appropriate for the use case? (real-time = low, batch = higher)
- Is buffer size aligned with destination ingestion capacity?
- For HTTP endpoints: lower buffer values = faster delivery but higher cost
- For S3: higher buffer values = fewer, larger objects (better for query engines like Athena)

### 3. Ingestion Metrics (CloudWatch)

Check these metrics in the `AWS/Firehose` namespace:

| Metric | What to check | Alert threshold |
|--------|---------------|-----------------|
| `IncomingBytes` | Data flowing in | Zero = no data reaching Firehose |
| `IncomingRecords` | Records flowing in | Zero = upstream problem |
| `IncomingPutRequests` | API calls succeeding | Sudden drops = producer issue |
| `ThrottledRecords` | Throttling occurring | Any non-zero value needs investigation |
| `BytesPerSecondLimit` | Current throughput limit | Compare to IncomingBytes rate |
| `RecordsPerSecondLimit` | Current records/sec limit | Compare to IncomingRecords rate |

**Important:** CloudWatch metrics are aggregated over 1-minute intervals. Short microbursts (few seconds) may not appear in IncomingBytes/IncomingRecords but WILL show in ThrottledRecords.

### 4. Throttling Investigation

If `ThrottledRecords > 0`:
1. Check for **microbursts** — sudden spikes exceeding per-second capacity
2. Verify producer is using **exponential backoff and retry**
3. Check record sizes and batch sizes being sent
4. Consider requesting a **throughput quota increase** via AWS Support
5. Note: Direct PUT streams auto-scale throughput when sustained throttling occurs

### 5. Source-Specific Checks

**If source is Kinesis Data Streams:**
- Check `DataReadFromKinesisStream.Bytes` and `.Records` — confirm data is being read
- Check `KinesisMillisBehindLatest` — high values mean Firehose is falling behind
- Check `ThrottledGetRecords` and `ThrottledGetShardIterator` — KDS throttling

**If source is MSK:**
- Check `DataReadFromSource.Records` and `.Bytes`
- Check `KafkaOffsetLag` — difference between what's available and what's read
- Check `DataReadFromSource.Backpressured` — indicates reading is delayed
- Check `FailedValidation.Records` — records failing schema validation

### 6. Recommended CloudWatch Alarms

Create alarms for:
- `IncomingBytes` = 0 for 5 minutes (no data flowing)
- `ThrottledRecords` > 0 (capacity issue)
- `DeliveryTo<Destination>.DataFreshness` > buffer interval × 2 (delivery stalled)
- `DeliveryTo<Destination>.Success` = 0 for 5 minutes (delivery failures)
- `BytesPerSecondLimit` approaching IncomingBytes rate (nearing capacity)

## Common Issues

| Symptom | Likely Cause | Resolution |
|---------|--------------|------------|
| IncomingRecords = 0 | Producer not sending data | Check producer app, verify PutRecord/PutRecordBatch calls |
| ThrottledRecords spikes | Microbursts | Implement exponential backoff, smooth producer rate |
| High KinesisMillisBehindLatest | Firehose can't keep up with KDS | Increase Firehose throughput or reduce KDS shard count |
| Stream status not ACTIVE | Configuration issue | Check recent API calls for errors |
