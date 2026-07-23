# Data Transformation Validation

## Overview
Validate Lambda-based data transformations in Amazon Data Firehose delivery streams. Data transformation is optional but when enabled, must follow strict response format requirements.

## How Transformation Works

1. Firehose buffers incoming data (default: up to 1 minute, up to 3 MiB)
2. Firehose invokes the Lambda function with a batch of records
3. Lambda processes records and returns them with required fields
4. Firehose delivers transformed data to the destination
5. On Lambda failure, Firehose retries (default: 3 times)

## Validation Steps

### 1. Lambda Function Response Format

Every record returned from Lambda MUST include these three fields — otherwise Firehose rejects the records and treats them as transformation failures:

| Field | Type | Description |
|-------|------|-------------|
| `recordId` | string | Must EXACTLY match the recordId passed in by Firehose. Any mismatch = transformation failure |
| `result` | string | Must be one of: `"Ok"`, `"Dropped"`, `"ProcessingFailed"` |
| `data` | string | The transformed data payload, **Base64-encoded** |
| `metadata` | object | (Optional) Used for dynamic partitioning |

**Result status meanings:**
- `"Ok"` — Record transformed successfully
- `"Dropped"` — Record intentionally skipped (counts as success in metrics)
- `"ProcessingFailed"` — Record could not be processed (counts as failure)

**Critical:** Firehose counts records with `"Ok"` and `"Dropped"` as successfully processed. Only `"ProcessingFailed"` records count toward failure metrics.

### 2. Lambda Configuration Checks

```bash
aws lambda get-function-configuration --function-name <FUNCTION_NAME>
```

Validate:
- **Timeout**: Must be sufficient for batch processing. If Lambda duration exceeds the timeout, the entire batch fails
- **Memory**: Adequate for the transformation logic and batch size
- **Concurrency**: Not hitting reserved concurrency limits
- **Runtime**: Not using a deprecated runtime

### 3. Transformation Buffer Configuration

The transformation buffer is separate from the destination buffer:

| Setting | Default | Range | Notes |
|---------|---------|-------|-------|
| Buffer interval | 60 seconds | 0–900 seconds | Time before invoking Lambda |
| Buffer size | 3 MiB | 0.2–3 MiB | Data size before invoking Lambda |
| Retries | 3 | 0–9 | Retry count on Lambda failure |

**Key considerations:**
- Larger buffers = fewer Lambda invocations = lower cost
- Smaller buffers = lower latency but more invocations
- Total processing time (buffering + Lambda execution) adds to end-to-end latency

### 4. CloudWatch Metrics for Transformation

| Metric | What it means |
|--------|---------------|
| `ExecuteProcessing.Duration` | Time spent in Lambda (ms) — monitor for increases |
| `ExecuteProcessing.Success` | 1 = Lambda invocation succeeded, 0 = failed |
| `SucceedProcessing.Records` | Records with `Ok` or `Dropped` result |
| `SucceedProcessing.Bytes` | Bytes from successfully processed records |

### 5. CloudWatch Logs for Transformation Errors

When error logging is enabled, check the Firehose log group for transformation errors:
```
/aws/kinesisfirehose/<STREAM_NAME>
```

Common error patterns:
- `Lambda invocation timed out` — Increase Lambda timeout
- `Lambda function throttled` — Increase concurrency or use provisioned concurrency
- `Response validation failed` — Missing required fields in Lambda response
- `Record ID mismatch` — Lambda returned different recordId than input

### 6. Common Transformation Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| All records show ProcessingFailed | Lambda error/crash | Check Lambda CloudWatch logs |
| Intermittent failures | Lambda throttling | Increase reserved concurrency |
| High latency | Lambda cold starts + large batches | Reduce buffer size or use provisioned concurrency |
| recordId mismatch error | Bug in Lambda code | Ensure recordId passthrough is exact |
| Payload too large after transform | Transformed data exceeds record size limit | Reduce output size or split records |
| Base64 encoding errors | Incorrect encoding in Lambda | Verify Base64 encoding of `data` field |

### 7. Lambda Function Validation Checklist

- [ ] Function returns ALL records (even if dropping some with `"Dropped"`)
- [ ] `recordId` is passed through unchanged for every record
- [ ] `result` is one of the three valid values (case-sensitive)
- [ ] `data` field is properly Base64-encoded
- [ ] Function handles empty batches gracefully
- [ ] Function timeout > expected processing time for max batch size
- [ ] Error handling doesn't swallow exceptions silently
- [ ] Function has appropriate IAM permissions for any external calls

### 8. Testing Transformation

Use the Firehose test invocation feature:
```bash
aws firehose test-delivery-stream-input-transformation \
  --delivery-stream-name <STREAM_NAME> \
  --records Data="<BASE64_ENCODED_SAMPLE_RECORD>"
```

Verify:
- Response contains all required fields
- recordId matches input
- Data is valid Base64
- Result is "Ok" for normal records
