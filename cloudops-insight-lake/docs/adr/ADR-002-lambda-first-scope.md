# ADR-002: Lambda-First MVP Scope

- **Status:** Accepted
- **Date:** 2026-07-14
- **Decision scope:** MVP service scope and extensibility boundary

## Context

ADR-001 defines CloudOps Insight Lake as an analytical platform that associates cost, resource inventory, and operational data at resource level. The remaining scope question is whether the MVP should support several AWS services immediately or implement one service-specific analytical path completely first.

AWS services do not share a uniform operational usage model.

For AWS Lambda, workload usage can be described through service-native measures such as:

- `Invocations`
- `Errors`
- `Duration`
- `Throttles`

An invocation represents a clearly defined Lambda workload event. EC2, RDS, S3, and other AWS services use materially different operational semantics. EC2 usage may be interpreted through runtime, CPU, memory, or network activity. RDS usage may be interpreted through connections, queries, I/O, storage, and database load. These measures are not directly equivalent to Lambda invocations.

A single generic operational fact table for multiple services would therefore either:

- contain many service-specific nullable columns,
- reduce different metrics to an ambiguous generic usage measure,
- require extensive service-dependent conditional logic, or
- use a generic key-value metric model that weakens explicit data grain and analytical semantics.

The MVP must remain technically bounded, testable, cost-controlled, and explainable. It should demonstrate a complete end-to-end data engineering path rather than broad but shallow service coverage.

## Decision

CloudOps Insight Lake will use a **Lambda-first MVP scope**.

The first complete analytical path will support AWS Lambda workloads only.

The following components remain service-independent where their semantics are genuinely reusable:

- S3 Raw and Curated layers
- canonical `resource_key`
- `dim_resource`
- `fact_resource_cost_daily`
- source contract and data quality patterns
- incremental processing and manifest state
- quarantine handling
- pipeline audit and lineage
- Glue Data Catalog
- Athena Workgroup
- IAM, observability, and cost-control patterns

The following components remain explicitly Lambda-specific:

- Lambda Inventory Collector
- Lambda source adapter
- CloudWatch Lambda Metrics Collector
- Lambda metric normalization and aggregation
- `fact_lambda_metrics_daily`
- Lambda reliability rules
- cost-versus-invocation efficiency analysis

The curated model will not introduce one generic operational fact table containing service-specific nullable columns.

Additional AWS services may be added later only through an explicit extension decision. Each new service must define:

1. its resource discovery method,
2. its canonical identity mapping,
3. its operational metric semantics,
4. the grain of its service-specific fact model,
5. its quality rules,
6. its efficiency or reliability interpretation, and
7. its expected cost and operational complexity.

Supporting another service does not imply that Lambda metrics or analytical rules can be reused unchanged.

## Alternatives Considered

### 1. Generic multi-service MVP

The MVP could support Lambda, EC2, RDS, and other AWS services from the beginning.

This would provide broader AWS service coverage and could make the platform appear more comprehensive. However, the operational metrics of these services have different meanings, units, collection mechanisms, and aggregation rules.

A multi-service MVP would require several service-specific collectors, adapters, fact models, quality rules, and analytical policies before the core cross-source association approach has been validated.

**Rejected because it would increase scope and failure surface while weakening semantic clarity and implementation depth.**

### 2. Single generic operational metrics table

The platform could store all service metrics in one generic table, for example through columns such as `metric_name`, `metric_value`, and `metric_unit`.

This long-form model can be useful for raw metric storage or observability systems. It is less suitable as the primary curated analytical model for this MVP because business queries would need to reconstruct service-specific meaning repeatedly.

Alternatively, a wide table with columns for Lambda, EC2, and RDS metrics would contain many non-applicable `NULL` values and would not have a consistent cross-service row meaning.

**Rejected as the primary curated model because it obscures grain, types, aggregation behavior, and service-specific analytical semantics.**

### 3. Fully Lambda-specific platform without reusable components

The entire architecture could be designed only for Lambda, including resource identity, cost storage, audit, and pipeline control.

This would simplify the immediate implementation but would unnecessarily couple reusable data platform capabilities to one AWS service.

**Rejected because canonical identity, cost facts, data quality, incremental processing, audit, IAM, and catalog patterns can remain service-independent without forcing operational metrics into a generic model.**

## Consequences

### Positive

- The MVP has a clear and defensible service boundary.
- Lambda operational metrics retain explicit meaning and units.
- Every curated table can have a precise grain.
- Data quality and analytical rules can be deterministic and testable.
- The project can demonstrate a complete vertical data engineering path.
- Implementation effort and AWS cost remain controlled.
- Generic platform components remain reusable for later service extensions.
- Future services can be added through explicit adapters and service-specific fact models rather than by weakening the existing Lambda model.

### Negative

- The MVP does not provide cross-service operational analytics.
- EC2, RDS, S3, and other AWS workloads are not analyzed.
- Portfolio breadth is lower than in a multi-service demonstration.
- Adding another service later requires additional contracts, collectors, transformations, tests, and potentially new fact tables.
- There is no universal cross-service workload unit for direct operational efficiency comparison.

## Resulting Architectural Requirements

This decision creates the following requirements:

1. The MVP inventory and operational collectors support AWS Lambda only.
2. `dim_resource` and `fact_resource_cost_daily` remain generic where their data semantics permit it.
3. Lambda operational data is stored in `fact_lambda_metrics_daily`.
4. The grain of `fact_lambda_metrics_daily` is one daily operational metrics record per Lambda resource.
5. Lambda metrics must not be represented as generic service-independent usage measures.
6. Service-specific columns must not be added to a shared operational fact table solely to simulate multi-service support.
7. New AWS services require explicit source contracts, adapters, metric semantics, fact-model grain, quality rules, and an architecture decision.
8. EC2, RDS, S3 workload analytics, and cross-service operational comparison remain outside the MVP scope.
9. Exam breadth for excluded services is handled through separate labs and service-decision exercises rather than by expanding the core MVP.
