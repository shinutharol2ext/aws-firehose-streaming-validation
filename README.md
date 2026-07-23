# Validate and Evaluate Amazon Data Firehose Streaming Workflows

A [Kiro Power](https://kiro.dev/docs/powers/) that gives your AI agent expert knowledge for validating and evaluating Amazon Data Firehose delivery streams end-to-end.


## What it does

When you mention Firehose, delivery streams, or streaming pipelines, this power activates and provides:

- **Workflow Evaluation** — Score your pipeline across 8 dimensions (throughput, latency, cost, data quality, resilience, scalability, security, observability) with a structured report template
- **Architecture Pattern Matching** — Identify which of 6 canonical patterns your pipeline follows and surface anti-patterns
- **Delivery Stream Health** — Validate ingestion metrics, buffering config, throttling, and recommended alarms
- **Transformation Validation** — Verify Lambda response format, error handling, and performance
- **Destination Reachability** — Check IAM, connectivity, and delivery success per destination type (S3, Redshift, OpenSearch, Snowflake, Splunk, HTTP)
- **End-to-End Data Flow** — Full pipeline validation with latency measurement, record reconciliation, and troubleshooting decision tree

## Install

### From Kiro IDE
1. Open the Powers panel
2. Click **Add Custom Power** → **Import power from GitHub**
3. Enter: `https://github.com/stharolz/aws-firehose-streaming-validation`

### From folder
1. Clone this repo
2. Open Kiro → Powers panel → **Add Custom Power** → **Import power from a folder**
3. Select the cloned directory

## Usage

Once installed, mention any Firehose-related topic in your Kiro chat:

- "Evaluate my Firehose streaming workflow"
- "Validate the delivery stream health"
- "Is my Firehose pipeline following best practices?"
- "Check my Firehose transformation function"
- "Score my data pipeline architecture"

The power activates automatically based on keywords.

## Structure

```
aws-firehose-streaming-validation/
├── POWER.md                              # Power metadata, onboarding, steering map
├── LICENSE                               # MIT License
├── PRIVACY.md                            # Privacy policy
├── README.md                             # This file
└── steering/
    ├── workflow-evaluation.md            # 8-dimension scoring framework
    ├── architecture-patterns.md          # 6 patterns + anti-patterns reference
    ├── delivery-stream-health.md         # Ingestion, buffering, throttling, alarms
    ├── transformation-validation.md      # Lambda response format, errors, testing
    ├── destination-reachability.md       # Per-destination validation (6 types)
    └── end-to-end-data-flow.md           # Full pipeline validation + decision tree
```

## License

[MIT](LICENSE)

## Support

[Open an issue](https://github.com/stharolz/aws-firehose-streaming-validation/issues)
