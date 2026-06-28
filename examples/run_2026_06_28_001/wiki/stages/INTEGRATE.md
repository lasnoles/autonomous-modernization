---
type: stage
title: "INTEGRATE — environment stand-up & E2E verification"
description: "Stood up a healthy Podman topology and verified the migration: E2E 38/38, SLOs PASS (p95 182 ms, error 0.00%)."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Stage::INTEGRATE
tokens: 120400
tags: [stage, integrate, podman, e2e, slo]
links: [index.md, EXECUTE.md, ../environment/environment.md, ../e2e/e2e-report.md, ../gaps/GAP-007.md, ../log.md, ../tokens.md]
---

# Stage: INTEGRATE

Stands up a runnable, reproducible system topology (Podman) and verifies the
migrated service end-to-end from production-derived traces.

## What it did

- Provisioned the topology and confirmed every service healthy.
- Replayed E2E scenarios built from production traces.
- Measured SLOs against thresholds.

## Topology (CANON)

`billing-svc` + `postgres:16` + `rabbitmq:3` +
**`mock-payment-provider`** (placeholder for GAP-007) +
observability (`otel` / `prometheus` / `grafana` / `loki`) — **all healthy**.

The `mock-payment-provider` stands in for the unknown external payment provider;
the charge flow's external dependency is mocked, not live — see
[`../gaps/GAP-007.md`](../gaps/GAP-007.md).

## Key numbers (CANON)

| Metric | Value | Threshold | Result |
|--------|-------|-----------|--------|
| E2E scenarios | **38 / 38 pass** | all pass | ✅ |
| p95 latency | **182 ms** | < 250 ms | ✅ |
| Error rate | **0.00%** | < 0.5% | ✅ |

Overall: **PASS**.

## Tokens

**120.4k** — see [`../tokens.md`](../tokens.md).

## Links

- Environment topology + monitoring: [`../environment/environment.md`](../environment/environment.md)
- Full E2E report: [`../e2e/e2e-report.md`](../e2e/e2e-report.md)
- Placeholder external (mocked here): [`../gaps/GAP-007.md`](../gaps/GAP-007.md)
- Acceptance scoreboard: [`../key-results/index.md`](../key-results/index.md)
- Previous stage: [EXECUTE](EXECUTE.md)
