---
type: key-result
title: "KR-01 — Build green on Java 21 / Spring Boot 3"
description: "billing-svc compiles and tests green on JDK 21 and Spring Boot 3.3 after migration. MET."
status: MET
runId: run_2026_06_28_001
repo: billing-svc
gid: run::run_2026_06_28_001::kr::KR-01
tags: [key-result, build, java-21, spring-boot-3]
links:
  - index.md
  - ../change-units/CU-014.md
  - ../change-units/CU-018.md
  - ../change-units/CU-022.md
  - ../stages/EXECUTE.md
  - ../stages/INTEGRATE.md
  - ../e2e/e2e-report.md
---

# KR-01 — Build green on Java 21 / Spring Boot 3

**Status: ✅ MET**

## Acceptance criterion

`billing-svc` compiles and its test suite passes on **Java 21** with **Spring
Boot 3.3**, replacing Java 8 + Spring Boot 1.5.

## Result

Clean build and green test suite on the target toolchain. The migration was
delivered by the wave-2/wave-3 change-units that move the codebase onto the new
runtime, then confirmed by the INTEGRATE stage running the built artifact in the
Podman topology.

## Evidence

| Evidence | Link |
|----------|------|
| `javax → jakarta` namespace migration (compile prerequisite for SB 3) | [CU-014](../change-units/CU-014.md) |
| Spring Boot 3.x upgrade (framework + starters) | [CU-018](../change-units/CU-018.md) |
| Field → constructor injection (SB 3 idiomatic wiring) | [CU-022](../change-units/CU-022.md) |
| EXECUTE — all 14 CUs APPLIED | [stages/EXECUTE.md](../stages/EXECUTE.md) |
| INTEGRATE — built artifact runs healthy in Podman topology | [stages/INTEGRATE.md](../stages/INTEGRATE.md) |
| Runtime behavior verified equivalent | [e2e/e2e-report.md](../e2e/e2e-report.md) |

## Notes

- Toolchain pinned at INTAKE: **JDK 21**, OpenRewrite 2.x.
- Build-green is a precondition for KR-03 (the artifact must run before
  equivalence can be tested) — see [KR-03](kr-03-e2e-equivalence.md).
