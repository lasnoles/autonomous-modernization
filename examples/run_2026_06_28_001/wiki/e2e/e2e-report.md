---
type: e2e-scenario
title: "E2E verification — 38/38 behavioral equivalence on production traces"
description: "Whole-system end-to-end verification for INTEGRATE: 38 scenarios built from production traces, 38/38 pass behavioral equivalence, SLOs met, all exit criteria green. Certification PASS."
status: PASS
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::E2E::billing-e2e
tags: [e2e-scenario, integrate, behavioral-equivalence, prod-traces, slo, certification, cross-repo]
links: [../index.md, ../log.md, ../stages/INTEGRATE.md, ../environment/environment.md, ../change-units/CU-021.md, ../gaps/GAP-007.md, ../key-results/kr-03-e2e-equivalence.md, ../key-results/kr-04-payments-seam.md, ../tokens.md]
---

# E2E verification — billing-svc

> Owned by the **integration-verifier** role in [INTEGRATE](../stages/INTEGRATE.md).
> The suite is built from **real production traffic**, not synthetic happy paths,
> and drives the *whole stood-up system* (service + dependents + mocks) through
> real user journeys on the [Podman topology](../environment/environment.md).

| | |
|---|---|
| **Result** | ✅ **PASS** — certification PASS |
| **Scenarios** | **38** (built from production traces) |
| **Behavioral equivalence** | **38 / 38** pass vs. baseline |
| **SLOs** | all within Target-Spec bounds |
| **Topology** | [environment/environment.md](../environment/environment.md) — all health checks green |
| **Blocking gaps** | none ([GAP-007](../gaps/GAP-007.md) non-blocking) |

## How the suite was built

1. The `replay` skill captured production request/response traces (HTTP,
   messages, DB I/O) as cassettes, with determinism seams (clock/random/UUID)
   controlled.
2. The tester turned **38 representative production traces** into end-to-end
   scenarios that drive the whole stood-up system.
3. Each scenario asserts **behavioral equivalence** vs. the recorded baseline
   outputs **and** checks monitoring signals (no new error classes, SLOs met).
4. Scenarios map to `Capability`/`Endpoint` nodes (`VERIFIES` edges) so business
   coverage is explicit.

## Behavioral equivalence

| | |
|---|---|
| Scenarios from prod traces | **38** |
| Passing equivalence | **38** |
| Failing | **0** |
| New error classes observed | **0** |

**38 / 38 pass behavioral equivalence** — the migrated Java 21 / Spring Boot 3.x
system produces outputs equivalent to the Java 8 baseline across every replayed
production journey.

## SLOs (read from monitoring)

Measured against the [observability stack](../environment/environment.md)
(Prometheus scraping `billing-svc`, Grafana `billing-e2e` dashboards):

| SLO | Measured | Target | Status |
|-----|----------|--------|--------|
| p95 latency | **182 ms** | < 250 ms | ✅ met |
| Error rate | **0.00%** | < 0.5% | ✅ met |
| Invoice throughput | **within 1% of baseline** | within ±5% of baseline | ✅ met |

## Payments seam — shadow verification

The Payments capability ([CU-021](../change-units/CU-021.md)) was verified in
**shadow** against the **placeholder** `mock-payment-provider` with the feature
flag **default-off**. Shadow traffic exercised the new seam path end-to-end and
matched the v1 path; the live path stays on v1. The placeholder satisfies
[GAP-007](../gaps/GAP-007.md) — a labeled mock, not a dangling wire — so the
integration is exercised without claiming the unknown real provider is resolved.

## Cross-repo journey

Because the graph spans repos, INTEGRATE composed both migrated services in one
Podman topology and replayed the cross-service journey:

```
web-app  ──►  billing-svc  /api/v1/invoices
```

Verified end-to-end — proving the *system*, not just each service, still works
after modernization.

## Exit-criteria checklist

| Criterion | Status |
|-----------|--------|
| Full topology comes up healthy under Podman (all health checks green) | ✅ topologyHealthy |
| E2E scenarios from prod traces pass behavioral equivalence | ✅ e2eEquivalenceGreen (38/38) |
| Monitoring shows no new error classes; SLOs within target bounds | ✅ slosMet |
| Every external dependency satisfied by a real backend or labeled placeholder mock | ✅ noDanglingIntegration (payment-provider → labeled mock) |
| Remaining gaps are non-blocking | ✅ noBlockingGaps ([GAP-007](../gaps/GAP-007.md) non-blocking) |

**Certification: PASS** — the run reaches COMPLETE with E2E green.

## Links

- The topology this ran on → [environment/environment.md](../environment/environment.md)
- KR this satisfies → [key-results/kr-03-e2e-equivalence.md](../key-results/kr-03-e2e-equivalence.md)
- Payments-seam KR → [key-results/kr-04-payments-seam.md](../key-results/kr-04-payments-seam.md)
- Placeholder gap → [gaps/GAP-007.md](../gaps/GAP-007.md)
- Stage → [stages/INTEGRATE.md](../stages/INTEGRATE.md) · Log → [log.md](../log.md) · Tokens → [tokens.md](../tokens.md)
- Front door → [index.md](../index.md)
