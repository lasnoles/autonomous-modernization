---
type: stage
title: "Pipeline — run_2026_06_28_001"
description: "The nine-stage modernization pipeline (INTAKE→COMPLETE) with per-stage status, key output, and token spend."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
tags: [pipeline, stages, overview]
links: [../index.md, ../log.md, ../tokens.md, ../key-results/index.md]
---

# Pipeline — `run_2026_06_28_001`

Run **billing-jakarta-payments-2026q3** · environment `staged` ·
started `2026-06-28T10:00:00Z` · completed `2026-06-28T14:20:00Z` · **COMPLETE**.

Objective: migrate `billing-svc` from Java 8 / Spring Boot 1.5 to Java 21 /
Spring Boot 3.x, and extract **Payments** behind a strangler-fig seam.

This page is progressive disclosure: each stage links down to its own page.
Read [`../log.md`](../log.md) for the chronological state-machine transitions and
[`../tokens.md`](../tokens.md) for the full token rollup.

## Stages

| # | Stage | Status | Key output | Tokens |
|---|-------|--------|------------|--------|
| 1 | INTAKE | ✅ done | Toolchain pinned (JDK 21, OpenRewrite 2.x); repos `billing-svc` (+ `web-app` consumer) acquired | 0 |
| 2 | [INDEX](stage-INDEX.md) | ✅ done | 1,204 graph nodes; **37** non-synthetic entry points (+14 synthetic excluded) | 48.1k |
| 3 | [RECOVER](RECOVER.md) | ✅ done | 9 components; seams Payments 0.78 / Dunning 0.55 / Invoicing 0.40; 2 layering violations | 31.4k |
| 4 | [DISCOVER](DISCOVER.md) | ✅ done | 6 capabilities; 41 business rules; **41 flows** (avg coverage 0.86) | 27.3k |
| 5 | [DEBT](DEBT.md) | ✅ done | 58 debt items; 3 CVEs (incl. Log4j CVE-2021-44228); hotspot `InvoiceService` | 19.0k |
| 6 | [PLAN](PLAN.md) | ✅ done | 14 ChangeUnits in 4 waves; risk gates configured | 91.2k |
| 7 | [EXECUTE](EXECUTE.md) | ✅ done | All 14 CUs **APPLIED** (CU-031 & CU-021 via human approval) | 1,071.5k |
| 8 | [INTEGRATE](INTEGRATE.md) | ✅ done | Podman topology healthy; E2E 38/38; SLOs PASS (p95 182 ms, err 0.00%) | 120.4k |
| 9 | COMPLETE | ✅ done | Run sealed; OKF bundle finalized; key results met | — |

**Total tokens:** 1,423,900 — see [`../tokens.md`](../tokens.md).

## Where to go next

- The proof of "no single front door": [`../entry-points/index.md`](../entry-points/index.md)
- All traced flows + coverage: [`../flows/index.md`](../flows/index.md)
- What changed: [`../change-units/index.md`](../change-units/index.md)
- Open questions / placeholders: [`../gaps/index.md`](../gaps/index.md)
- Gate decisions: [`../decisions/index.md`](../decisions/index.md)
- Acceptance scoreboard: [`../key-results/index.md`](../key-results/index.md)
