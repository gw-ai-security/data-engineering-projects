# ADR-001: CloudOps Analytical Use Case

- **Status:** Accepted
- **Date:** 2026-07-06
- **Decision scope:** MVP analytical use case

## Context

Cloud platform teams often need to analyze data that originates from separate financial, resource inventory, and operational contexts.

Cloud cost data describes what was charged. Resource inventory data describes which Lambda workloads exist and provides resource identity, configuration, and governance attributes. Operational metrics describe how those workloads are used and provide selected reliability signals.

When these data contexts are analyzed independently, cost changes have limited interpretability. An increase in cost can be measured, but cost data alone does not show whether workload usage increased at a comparable rate. For example, a substantial increase in cost has a different analytical meaning when invocation volume grows at a similar rate than when invocation volume changes only slightly.

The same limitation applies to governance and reliability findings. Missing governance attributes or elevated error rates can indicate that a workload requires investigation, but these observations do not establish a causal root cause.

The MVP therefore requires a bounded analytical use case that associates cost, resource inventory, and operational metrics at resource level. In this project, correlation means cross-source resource-level association rather than calculation of a statistical correlation coefficient.

## Decision

CloudOps Insight Lake will be designed as an analytical data platform for AWS Lambda workloads.

The MVP will combine resource-level cost data, Lambda resource inventory, and selected operational metrics in order to identify:

- cost and usage divergence,
- tag governance violations,
- reliability review candidates, and
- cross-source data integrity problems.

The platform will use a canonical resource identity so that records from separate source contexts can be associated with the same logical Lambda resource.

The analytical output will identify **review candidates**. It will not claim to diagnose causal root causes automatically.

The MVP is intentionally analytical and batch-oriented. It is not designed as a real-time incident detection or alerting system.

## Alternatives Considered

### 1. Cost-only analytics

The platform could analyze only cloud cost development.

This would reduce source integration and transformation complexity. However, cost growth could not be contextualized with workload usage. The platform could identify that spending increased but could not determine whether usage increased at a comparable rate.

**Rejected for the MVP because it does not support the central cost-versus-usage analysis.**

### 2. Generic multi-service CloudOps platform

The initial platform could include Lambda, EC2, RDS, and other AWS services.

This would provide broader service coverage. However, operational usage semantics differ materially between AWS services. Lambda metrics such as `Invocations`, `Duration`, `Errors`, and `Throttles` cannot be generalized directly to EC2 or RDS workloads through a single generic usage model.

**Rejected for the MVP because the additional semantic complexity would weaken the bounded analytical model before the core cross-source correlation approach has been validated.**

### 3. Real-time monitoring system

The platform could be designed for immediate incident detection and operational alerting.

This would support low-latency operational response. However, the defined project question focuses on analytical review of cost development, workload usage, governance, and reliability patterns. The efficiency policy compares complete consecutive calendar months and therefore does not require real-time event processing.

**Rejected because real-time monitoring solves a different primary problem than the defined MVP.**

## Consequences

### Positive

- Cost changes can be interpreted together with workload usage instead of being reviewed in isolation.
- Resource inventory provides identity and governance context for cost and metric records.
- Selected reliability signals can identify Lambda workloads that require further technical investigation.
- A bounded Lambda-first use case keeps operational metric semantics explicit and consistent.
- The analytical rules remain deterministic and explainable.
- The decision creates a clear requirement for canonical resource identity and cross-source join integrity.

### Negative

- The platform can identify suspicious patterns and review candidates but cannot prove causal root causes from the available data.
- The batch-oriented analytical model cannot detect or alert on short-lived incidents in real time.
- Lambda-specific usage semantics cannot be transferred directly to EC2, RDS, or other AWS services.
- Extending the platform to additional services will require service-specific metric models and explicit decisions about how cross-service analysis should be performed.

## Resulting Architectural Requirements

This decision creates the following requirements for later architecture decisions:

1. Cost, inventory, and operational data must remain distinguishable as separate source contexts.
2. Records referring to the same logical Lambda resource must be joinable through a canonical resource identity.
3. Generic cost facts and Lambda-specific operational metrics must retain separate data semantics.
4. Analytical classifications must be described as review candidates rather than causal diagnoses.
5. The MVP does not require real-time streaming or multi-service workload modeling.
