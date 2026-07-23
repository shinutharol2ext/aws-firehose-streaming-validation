# Destination Reachability Validation

## Overview
Validate that Amazon Data Firehose can successfully deliver data to the configured destination. Covers IAM permissions, network connectivity, destination configuration, and delivery metrics per destination type.

## Universal Checks (All Destinations)

### 1. IAM Role Validation

Firehose uses an IAM role to access destination resources. Verify:

```bash

aws firehose describe-delivery-stream --delivery-stream-name <STREAM_NAME> \
  --query 'DeliveryStreamDescription.Destinations[0].*.RoleARN'
```

**Required trust policy:**
```json
{
  "Effect": "Allow",
  "Principal": { "Service": "firehose.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

**Common IAM issues:**
- Role doesn't have `firehose.amazonaws.com` in trust policy
- Role missing permissions for destination (S3 PutObject, OpenSearch bulk index, etc.)
- Role missing KMS decrypt/encrypt permissions (if encryption enabled)
- Role missing Lambda invoke permissions (if transformation enabled)
- Role was auto-created with placeholders that were never updated

### 2. Delivery Success Metrics

Check the delivery success metric for your destination type:

| Destination | Success Metric | Failure Indicator |
|-------------|---------------|-------------------|
| S3 | `DeliveryToS3.Success` | Sum = 0 over 5 min |
| Redshift | `DeliveryToRedshift.Success` | Sum = 0 over 5 min |
| OpenSearch | `DeliveryToAmazonOpenSearchService.Success` | Sum = 0 over 5 min |
| OpenSearch Serverless | `DeliveryToAmazonOpenSearchServerless.Success` | Sum = 0 over 5 min |
| Snowflake | `DeliveryToSnowflake.Success` | Sum = 0 over 5 min |
| Splunk | `DeliveryToSplunk.Success` | Sum = 0 over 5 min |
| HTTP Endpoint | `DeliveryToHttpEndpoint.Success` | Minimum = 0 (delivery failures) |

**Important:** Small drops in average success metrics do NOT indicate data loss. Firehose retries delivery errors and does not move forward until records are either delivered to the destination or backed up to S3.

### 3. Data Freshness (Latency)

The `DataFreshness` metric shows the age of the oldest record in Firehose:

**Alert when:** DataFreshness > (buffer interval × 2) — indicates delivery is stalled.

| Destination | Metric |
|-------------|--------|
| S3 | `DeliveryToS3.DataFreshness` |
| Redshift | `DeliveryToRedshift.DataFreshness` |
| OpenSearch | `DeliveryToAmazonOpenSearchService.DataFreshness` |
| Snowflake | `DeliveryToSnowflake.DataFreshness` |
| Splunk | `DeliveryToSplunk.DataFreshness` |
| HTTP Endpoint | `DeliveryToHttpEndpoint.DataFreshness` |

## Destination-Specific Validation

### Amazon S3

**Configuration checks:**
- Bucket exists and is in the expected region
- Bucket policy allows `s3:PutObject`, `s3:AbortMultipartUpload`, `s3:GetBucketLocation`, `s3:ListBucket`, `s3:ListBucketMultipartUploads`
- S3 prefix pattern is correct (especially with dynamic partitioning)
- If using KMS encryption on the bucket, the Firehose role has `kms:GenerateDataKey` and `kms:Decrypt`

**Validation commands:**
```bash
# Check bucket accessibility
aws s3 ls s3://<BUCKET_NAME>/<PREFIX>/ --region <REGION>

# Verify recent objects were delivered
aws s3 ls s3://<BUCKET_NAME>/<PREFIX>/ --recursive | tail -5
```

### Amazon Redshift

**Configuration checks:**
- Redshift cluster is available and accepting connections
- COPY command has correct table name, columns, and data format
- Intermediate S3 bucket is accessible (Firehose stages data in S3 before COPY)
- Redshift user credentials are valid
- Security group allows inbound from Firehose IPs
- `DeliveryToRedshift.DataFreshness` — includes S3 staging + COPY time

### OpenSearch Service / Serverless

**Configuration checks:**
- Domain/collection is active and healthy
- Index exists or auto-creation is enabled
- Index mapping is compatible with incoming data schema
- Cluster has sufficient storage and compute capacity

**OpenSearch-specific metrics:**
- `DeliveryToAmazonOpenSearchService.AuthFailure` — 1 = auth/permissions issue
- `DeliveryToAmazonOpenSearchService.DeliveryRejected` — 1 = delivery failure (mapping conflict, cluster full)

**Common issues:**
- Auth failure: Verify cluster access policy includes the Firehose role ARN
- Delivery rejected: Check index mapping conflicts or cluster storage pressure

### Snowflake

**Configuration checks:**
- Snowflake account URL is correct
- Private key or key pair authentication is valid
- Database, schema, and table exist
- Table columns match incoming data structure

**Snowflake-specific metrics:**
- `DeliveryToSnowflake.DataCommitLatency` — time for data to be committed after insert
- Increasing commit latency = Snowflake processing slower than ingestion rate

### Splunk

**Configuration checks:**
- HEC (HTTP Event Collector) endpoint URL is correct
- HEC token is valid and not expired
- HEC endpoint is reachable from Firehose
- Index specified in HEC token matches intended destination

**Splunk-specific metrics:**
- `DeliveryToSplunk.DataAckLatency` — time for Splunk to acknowledge
- Increasing trend = Splunk indexers are falling behind

### HTTP Endpoint (Custom)

**Configuration checks:**
- Endpoint URL is reachable and responding
- Authentication credentials (API key, access key) are valid
- Endpoint returns expected HTTP response codes (2xx = success)
- Content encoding matches (GZIP vs none)
- Retry duration is configured (default: 60 seconds)

## Backup Bucket Validation

For ALL destinations (except S3-primary), validate the backup configuration:

- **Failed-only backup**: Undeliverable records go to backup S3 bucket
- **All-records backup**: Every record also goes to backup S3

**Check backup is working:**
- `BackupToS3.Success` > 0 when delivery failures occur
- `BackupToS3.DataFreshness` — age of oldest backup record

**If backup is failing too:** Check the backup S3 bucket permissions independently.

## Error Log Investigation

Enable CloudWatch Logs error logging and check:
```
Log group: /aws/kinesisfirehose/<STREAM_NAME>
```

Common error patterns by destination:
- S3: `Access Denied`, `NoSuchBucket`, `KMS.AccessDeniedException`
- Redshift: `COPY command failed`, `Connection refused`
- OpenSearch: `403 Forbidden`, `index_not_found_exception`, `mapper_parsing_exception`
- HTTP: `Connection timeout`, `429 Too Many Requests`, `5xx responses`
