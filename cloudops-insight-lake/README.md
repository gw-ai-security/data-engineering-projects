# CloudOps Insight Lake

**Serverless Cloud Cost & Operational Efficiency Data Platform on AWS**

> A serverless AWS data engineering project that correlates cloud cost data, Lambda resource metadata, and operational metrics to identify cost-usage divergence, governance issues, and reliability review candidates.

## Project Status

**Current phase: Phase 0 — Architecture and Scope Freeze**

CloudOps Insight Lake is currently under active development.

The repository is built incrementally with a focus on technical understanding, reproducibility, data quality, cost control, and documented architecture decisions.

Planned implementation areas include:

- AWS Data Exports / CUR 2.0 ingestion
- AWS Lambda resource inventory collection
- Amazon CloudWatch metrics collection
- Amazon S3 Raw and Curated data layers
- AWS Glue Python Shell ETL
- Incremental object processing
- Data quality validation
- Apache Parquet datasets
- AWS Glue Data Catalog
- Amazon Athena analytics
- Pipeline observability
- IAM least-privilege controls
- Infrastructure as Code

---

## Problem Statement

Cloud platform teams often need to analyze information that originates from separate AWS data sources.

Cloud cost data describes **what was charged**.

Resource inventory describes **which workloads exist and how they are configured**.

Operational metrics describe **how workloads are actually used and how reliably they operate**.

Analyzing these datasets independently makes it difficult to determine whether increasing cloud costs are explained by increasing workload usage or whether a workload requires further efficiency, governance, or reliability review.

CloudOps Insight Lake addresses the following analytical question:

> **Which AWS Lambda workloads show a notable divergence between cost development and actual usage, and where do data governance or reliability issues exist?**

The project intentionally focuses on **AWS Lambda workloads** for the MVP.

This bounded scope allows operational measures such as `Invocations`, `Errors`, `Duration`, and `Throttles` to retain clear and consistent semantics.

---

## Project Objectives

The platform is designed to support four core analytical areas.

### Cost Analysis

Identify Lambda resources with the highest monthly cost and analyze cost development over time.

### Operational Efficiency

Compare cost growth with actual workload usage.

A workload may become an efficiency review candidate when cost growth significantly exceeds invocation growth.

The initial thresholds used by the project are configurable demonstration heuristics and are **not presented as AWS or FinOps industry standards**.

### Tag Governance

Identify resources that violate the minimum tagging policy.

Required governance attributes:

- `Environment`
- `Owner`
- `CostCenter`

### Reliability Review

Identify Lambda workloads with operational patterns such as elevated error rates that require further technical investigation.

The MVP uses deterministic rule-based classifications.

**Machine learning anomaly detection is explicitly outside the MVP scope.**

---

## Planned Architecture

```text
AWS Data Exports / CUR 2.0
        |
        | AWS-managed S3 delivery
        v
raw/cost/cur2/

AWS Lambda Inventory Collector
        |
        | ListFunctions + service-native tag retrieval
        v
raw/inventory/

Amazon CloudWatch Collector
        |
        | GetMetricData
        | Invocations
        | Errors
        | Duration
        | Throttles
        v
raw/metrics/

Synthetic Benchmark Generator
        |
        v
raw/synthetic/

        Raw Sources
             |
             v
    AWS Glue Python Shell
             |
      Source Adapters
             |
      Data Quality Gate
             |
        Normalization
             |
         Aggregation
             |
    Incremental Processing
             |
             v
        S3 Curated Layer
             |
             v
      AWS Glue Data Catalog
             |
             v
       Amazon Athena
             |
             v
 vw_lambda_efficiency_daily
```

---

## Data Sources

### AWS Data Exports / CUR 2.0

Cost and usage data is delivered directly to Amazon S3 using AWS-managed Data Exports delivery.

The project does **not** implement a Lambda-based CUR collector.

```text
AWS Data Exports
→ S3
→ raw/cost/cur2/
```

This keeps AWS-managed delivery separate from custom collector logic.

---

### Lambda Resource Inventory

The inventory collector discovers Lambda functions using the service-native Lambda APIs.

```text
ListFunctions
→ resource metadata
→ tag retrieval
→ raw/inventory/
```

The project does not rely exclusively on the Resource Groups Tagging API for complete Lambda discovery because completely untagged resources must remain visible to the inventory process.

---

### Amazon CloudWatch Metrics

The CloudWatch collector retrieves Lambda operational metrics using `GetMetricData`.

Metrics:

- `Invocations`
- `Errors`
- `Duration`
- `Throttles`

Planned collector characteristics:

- daily collection
- hourly metric period
- batched `MetricDataQueries`
- `NextToken` pagination
- retry handling
- exponential backoff
- API call and datapoint audit metrics

---

### Synthetic Benchmark

The project includes deterministic synthetic datasets for local pipeline development and quality testing.

Two source modes are strictly separated:

```text
SOURCE_MODE=aws
SOURCE_MODE=synthetic
```

Synthetic and real AWS records must not be silently mixed.

Every record contains:

```text
source_mode
```

Synthetic scenario records additionally contain:

```text
scenario_id
```

---

## Synthetic Ground-Truth Scenarios

The synthetic benchmark intentionally injects known data and analytics conditions.

| Scenario | Condition | Expected Classification |
|---|---|---|
| `S01` | Missing `Owner` tag | `TAG_GOVERNANCE_WARNING` |
| `S02` | Missing `CostCenter` | `TAG_GOVERNANCE_WARNING` |
| `S03` | Cost growth without comparable usage growth | `EFFICIENCY_REVIEW_CANDIDATE` |
| `S04` | Metric references unknown Lambda resource | `JOIN_INTEGRITY_FAILURE` |
| `S05` | Elevated Lambda error rate | `RELIABILITY_REVIEW_CANDIDATE` |

The generator also produces:

```text
synthetic_expected_results.json
```

This scenario oracle defines the expected classifications for deliberately manipulated resources.

Pipeline outputs are tested against this ground truth.

---

## Canonical Resource Identity

Data from cost, inventory, and operational metric sources must refer to the same logical resource.

The curated model therefore uses a canonical technical key:

```text
account_id
+
region
+
service_name
+
native_resource_id
```

Example:

```text
123456789012#eu-central-1#AWSLambda#billing-processor
```

Stored as:

```text
resource_key
```

The original AWS resource ARN is retained separately as `resource_arn`.

This explicit identity model allows source-specific identifiers to be normalized before curated datasets are joined.

---

## Curated Data Model

The MVP intentionally separates generic cost measures from Lambda-specific operational measures.

A single generic fact table containing service-specific nullable metric columns is not used.

### `dim_resource`

Resource identity and descriptive attributes.

```text
resource_key
resource_arn
account_id
region
service_name
resource_type
native_resource_id
environment
owner
cost_center
memory_mb
runtime
first_seen_at
last_seen_at
schema_version
```

### `fact_resource_cost_daily`

**Grain:** one daily cost record for a resource and cost usage classification.

```text
usage_date
resource_key
product_code
usage_type
usage_quantity
usage_unit
daily_cost_usd
currency
source_mode
schema_version
```

### `fact_lambda_metrics_daily`

**Grain:** one daily operational metrics record per Lambda resource.

```text
metric_date
resource_key
invocation_count
error_count
error_rate_pct
avg_duration_ms
throttle_count
source_mode
schema_version
```

### `pipeline_run_audit`

Pipeline-level operational and data quality evidence.

```text
run_id
pipeline_name
started_at
ended_at
status
objects_discovered
objects_processed
objects_skipped
records_ingested
records_rejected
records_curated
join_coverage_pct
run_duration_ms
source_lag_hours
error_type
error_message
```

An Athena view will combine curated cost and Lambda metric data for operational efficiency analysis:

```text
vw_lambda_efficiency_daily
```

---

## Efficiency Policy

Cost and usage development is compared using:

```text
complete calendar month
versus
directly preceding complete calendar month
```

The project intentionally does not use rolling 30-day windows for this analysis.

Requirements:

- two consecutive complete calendar months
- `prev_cost > 0`
- `prev_invocations > 0`
- configurable minimum baseline cost
- configurable minimum baseline invocation count

Initial demonstration thresholds:

```text
COST_GROWTH_THRESHOLD = 0.20
USAGE_GROWTH_THRESHOLD = 0.05
```

These values are portfolio demonstration heuristics.

They are not presented as standardized FinOps thresholds.

Monthly comparisons must verify calendar continuity explicitly because SQL `LAG()` returns the previous available row and does not guarantee the directly preceding calendar month.

---

## Incremental Processing

A source object is not considered permanently processed based only on its S3 key.

The pipeline tracks manifest state using:

```text
source_key
etag
last_modified
processed_at
pipeline_version
```

Processing identity:

```text
source_key + etag
```

Expected behavior:

```text
new object
→ PROCESS

same source_key + same etag
→ SKIP

same source_key + changed etag
→ REPROCESS
```

This design allows changed source objects to be detected and processed again.

---

## Data Quality

The pipeline includes explicit data quality validation rather than treating successful execution as proof of correct data.

Planned quality checks include:

- required-field completeness
- duplicate `resource_key` detection
- orphan Lambda metric detection
- join coverage measurement
- source schema validation
- scenario oracle validation

Core quality queries:

```text
Orphan Metrics
Join Coverage
Duplicate Resource Keys
```

A pipeline run is not considered successful solely because the ETL job completes without an exception.

---

## Pipeline Observability

The project distinguishes between two observability layers.

### AWS Runtime Observability

Infrastructure and AWS service execution behavior.

Examples:

- Lambda `Invocations`
- Lambda `Errors`
- Lambda `Duration`
- Lambda `Throttles`
- CloudWatch Logs

### Data Pipeline Observability

Data processing correctness and pipeline behavior.

Examples:

- `records_ingested`
- `records_rejected`
- `records_curated`
- `join_coverage_pct`
- `objects_processed`
- `objects_skipped`
- `source_lag_hours`
- `run_duration_ms`

Pipeline-level evidence is stored in:

```text
pipeline_run_audit
```

---

## S3 Data Lake Layout

Planned logical prefixes:

```text
raw/cost/
raw/inventory/
raw/metrics/
raw/synthetic/

curated/dim_resource/
curated/fact_resource_cost_daily/
curated/fact_lambda_metrics_daily/
curated/pipeline_run_audit/

manifests/
athena-results/
```

The Raw layer preserves source-oriented data.

The Curated layer contains normalized and analytics-oriented Parquet datasets.

---

## Partitioning Strategy

The MVP uses:

```text
year
month
```

Example:

```text
curated/fact_resource_cost_daily/
└── year=2026/
    └── month=07/
```

`service_name` is deliberately not used as a partition key in the Lambda-only MVP.

The partitioning decision will be evaluated using Athena query measurements.

Benchmark queries:

```text
Q1 Full Year Scan
Q2 Single Month Filter
Q3 Service Filter
Q4 Month + Resource Filter
```

Measured evidence:

- scanned bytes
- execution time
- result rows

The objective is to evaluate the partitioning decision using query behavior rather than assuming that more partitions automatically improve performance.

---

## Core Business Queries

### Q1 — Monthly Cost by Resource

> Which Lambda functions generate the highest cost?

### Q2 — Tag Governance

> Which Lambda resources violate the minimum tagging policy?

### Q3 — Cost per 1,000 Invocations

> What is the cost per 1,000 workload units?

### Q4 — Cost Growth versus Usage Growth

> Which Lambda resources show notable month-over-month divergence between cost growth and workload usage growth?

---

## Security Principles

The project applies least-privilege access between pipeline components.

Planned IAM roles:

```text
collector-role
etl-role
athena-query-role
```

Required negative IAM tests include:

| Test | Expected Result |
|---|---|
| Collector writes Raw | PASS |
| Collector writes Curated | `AccessDenied` |
| ETL reads Raw | PASS |
| ETL writes Curated | PASS |
| Athena reads Curated | PASS |
| Athena writes Raw | `AccessDenied` |

Additional S3 controls:

- Block Public Access
- SSE-S3
- TLS enforcement
- S3 Object Ownership
- lifecycle controls

Security behavior must be demonstrated through negative permission tests rather than inferred only from IAM policy documents.

---

## Cost Control

CloudOps Insight Lake is intentionally designed as a cost-controlled portfolio project.

Key principles:

- serverless AWS services where technically justified
- no persistent analytics cluster
- AWS resources created only when required by the active implementation phase
- synthetic local development before AWS pipeline deployment
- Athena scanned-byte measurement
- S3 lifecycle controls
- AWS Budgets
- explicit project cost model

The project does not claim to simulate Big Data scale.

Synthetic benchmark file size and record volume are measured and documented objectively.

---

## Schema Evolution

Curated datasets use explicit schema versions.

Version model:

```text
MAJOR.MINOR
```

Example of a non-breaking additive change:

```text
v1.0
→
v1.1
```

Breaking changes use a new major curated prefix:

```text
curated/v1/
curated/v2/
```

The planned migration pattern is:

```text
deploy v2 schema
→ historical backfill
→ reconcile row counts
→ execute quality rules
→ reconcile aggregates
→ validate Athena v2
→ switch analytics view
→ temporarily retain v1
```

This follows a backfill-and-cutover strategy.

---

## Repository Structure

```text
cloudops-insight-lake/
├── .gitignore
├── LICENSE
├── README.md
├── pyproject.toml
├── docs/
│   └── project_evidence_map.md
├── src/
│   └── cloudops_insight_lake/
│       └── __init__.py
└── tests/
    └── .gitkeep
```

This is the current Phase 0 repository structure. Future implementation areas described elsewhere in this README are planned and are not yet implemented as Python package modules.

The repository structure will evolve only when a concrete implementation requirement exists.

Empty technology-specific directories are intentionally avoided.

---

## Development Roadmap

| Phase | Focus |
|---|---|
| 0 | Architecture and Scope Freeze |
| 1 | Data Contracts and Synthetic Benchmark |
| 2 | Local Data Engineering Core |
| 3 | S3 Data Lake and Security |
| 4 | AWS Collectors |
| 5 | Glue ETL and Incremental State |
| 6 | Glue Catalog and Athena |
| 7 | Pipeline Observability and Operations |
| 8 | Minimal Infrastructure as Code |
| 9 | Glue PySpark / Job Bookmark Lab |
| 10 | Portfolio and Certification Evidence |

The project is implemented in small, testable development units.

Each major phase requires technical evidence before being considered complete.

---

## Out of Scope

The following technologies and use cases are intentionally excluded from the MVP:

- EC2 workload analytics
- RDS workload analytics
- S3 workload analytics
- multi-cloud analytics
- forecasting
- machine learning anomaly detection
- Amazon QuickSight
- Amazon Redshift
- Amazon Kinesis
- AWS Lake Formation
- Apache Iceberg
- AWS Step Functions

These are not excluded because they lack technical value.

They are excluded because the current MVP does not contain a demonstrated requirement that justifies their additional complexity and failure surface.

---

## Engineering Principles

This project follows several explicit engineering principles:

1. **Business problem before AWS service selection**
2. **Explicit data grain before aggregation**
3. **Source-specific adapters before canonical models**
4. **Deterministic data quality rules**
5. **Synthetic and AWS source modes remain separated**
6. **Incremental processing must detect changed source objects**
7. **Successful execution is not equivalent to correct data**
8. **Least privilege must be demonstrated with negative tests**
9. **Partitioning decisions require query evidence**
10. **Portfolio claims must be supported by repository artifacts and test results**

---

## Learning and Certification Context

CloudOps Insight Lake is developed as both a practical data engineering project and a technical learning environment.

The project covers concepts relevant to professional AWS data engineering work and AWS Data Engineer Associate preparation, including:

- data ingestion
- Amazon S3 data lakes
- data contracts
- data quality
- incremental processing
- Parquet
- partitioning
- AWS Glue
- AWS Glue Data Catalog
- Amazon Athena
- IAM
- cost optimization
- pipeline observability
- Infrastructure as Code
- Glue Spark Job Bookmarks

A dedicated evidence map will connect each claimed competency to concrete implementation artifacts, tests, and technical decisions.

```text
docs/project_evidence_map.md
```

---

## Author

Developed as a Data Engineering portfolio project focused on AWS cloud data platforms, operational analytics, data governance, and cost-aware serverless architecture.
