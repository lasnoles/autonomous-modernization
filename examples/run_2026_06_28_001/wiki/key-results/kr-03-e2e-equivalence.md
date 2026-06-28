---
type: key-result
title: "KR-03 — E2E equivalence on production traces"
description: "All 38 end-to-end scenarios replayed from production traces pass behavioral equivalence. MET."
status: MET
runId: run_2026_06_28_001
repo: billing-svc
gid: run::run_2026_06_28_001::kr::KR-03
tags: [key-result, e2e, equivalence, prod-traces]
links:
  - index.md
  - ../e2e/e2e-report.md
  - ../stages/INTEGRATE.md
  - ../environment/environment.md
  - ../flows/index.md
---

# KR-03 — E2E equivalence on production traces

**Status: ✅ MET**

## Acceptance criterion

The migrated `billing-svc` is **behaviorally equivalent** to the pre-migration
service, proven by replaying **production traces** end-to-end with no breaking
change to `/api/v1/**`.

## Result

| Metric | Target | Actual |
|--------|--------|-------:|
| Equivalence scenarios passing | 38 / 38 | **38 / 38** ✅ |
| p95 latency | < 250 ms | **182 ms** ✅ |
| Error rate | < 0.5% | **0.00%** ✅ |
| Certification | PASS | **PASS** ✅ |

All 38 scenarios derived from production traces pass equivalence against the
migrated service running in the Podman topology; SLOs met with margin.

## Evidence

| Evidence | Link |
|----------|------|
| Full E2E report (38/38, SLOs, certification) | [e2e/e2e-report.md](../e2e/e2e-report.md) |
| INTEGRATE stage — topology + certification PASS | [stages/INTEGRATE.md](../stages/INTEGRATE.md) |
| Podman topology under test | [environment/environment.md](../environment/environment.md) |
| Traced flows (41, avg coverage 0.86) backing the scenarios | [flows/index.md](../flows/index.md) |

## Notes

- Equivalence is run against the **staged** environment with the
  `mock-payment-provider` standing in for the unknown external provider
  ([GAP-007](../gaps/GAP-007.md)).
- Depends on KR-01 (the artifact must build and run) — see
  [KR-01](kr-01-build-green.md).
