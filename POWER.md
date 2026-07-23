---
name: "aws-firehose-streaming-validation"
displayName: "Validate and Evaluate Amazon Data Firehose Streaming Workflows"
description: "Evaluate and validate Amazon Data Firehose streaming workflows end-to-end: architecture assessment, throughput design, cost optimization, data quality, transformation, destination delivery, and operational health scoring"
keywords: ["firehose", "kinesis firehose", "data firehose", "delivery stream", "streaming", "data pipeline", "ingestion", "buffering", "transformation", "kinesis", "real-time data", "delivery validation", "streaming ETL", "data lake", "parquet", "dynamic partitioning"]
author: "stharolz"
---


# Validate and Evaluate Amazon Data Firehose Streaming Workflows

This power provides comprehensive evaluation and validation for Amazon Data Firehose delivery streams — from architecture assessment and workflow scoring through to operational health checks.

## Onboarding

### Step 1: Verify AWS CLI access
Ensure you have AWS CLI configured with appropriate permissions to describe Firehose streams:
- Verify with: `aws firehose list-delivery-streams`
- Required IAM permissions: `firehose:Describe*`, `firehose:List*`, `cloudwatch:GetMetricData`, `cloudwatch:GetMetricStatistics`, `logs:GetLogEvents`, `logs:FilterLogEvents`

### Step 2: Identify the delivery stream
Before validating, identify:
- **Stream name** — the Firehose delivery stream to evaluate
- **Source type** — Direct PUT, Kinesis Data Streams, or MSK
- **Destination type** — S3, Redshift, OpenSearch, Snowflake, Splunk, or HTTP Endpoint
- **Whether data transformation is enabled** (Lambda function)
- **Whether dynamic partitioning is enabled**
- **Whether format conversion is enabled** (Parquet/ORC)
- **Business SLA** — required data freshness and acceptable data loss

### Step 3: Determine evaluation scope
Choose the appropriate workflow:
- **Quick health check** → delivery-stream-health.md
- **Full architecture evaluation** → workflow-evaluation.md
- **Pattern identification** → architecture-patterns.md

## When to Load Steering Files

- Running a full streaming workflow evaluation with scoring → `workflow-evaluation.md`
- Identifying architecture patterns and anti-patterns → `architecture-patterns.md`
- Checking delivery stream health, ingestion rates, or buffering → `delivery-stream-health.md`
- Validating Lambda data transformation function → `transformation-validation.md`
- Verifying destination connectivity and delivery success → `destination-reachability.md`
- Running full pipeline validation end-to-end → `end-to-end-data-flow.md`

## License and support

This power is provided under the MIT License (MIT).
- [Privacy Policy](https://github.com/stharolz/aws-firehose-streaming-validation/blob/main/PRIVACY.md)
- [Support](https://github.com/stharolz/aws-firehose-streaming-validation/issues)
