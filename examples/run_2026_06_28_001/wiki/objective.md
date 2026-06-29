---
type: run-objective
title: "Objective — billing-jakarta-payments-2026q3"
description: "Migrate billing-svc to Java 21 + Spring Boot 3.x and extract Payments behind a strangler-fig seam, with no breaking change to /api/v1/**."
status: COMPLETE
runId: run_2026_06_28_001
owner: min_than_zaw@msi-global.com.sg
environment: staged
gid: run::run_2026_06_28_001::objective
tags: [objective, java-21, spring-boot-3, strangler-fig, payments]
links:
  - index.md
  - key-results/index.md
  - stages/PLAN.md
  - change-units/CU-021.md
  - gaps/GAP-007.md
---

# Objective

Migrate **`billing-svc`** to **Java 21 + Spring Boot 3.x** and extract the
**Payments** capability behind a **strangler-fig seam**.

This run is the source of the Target Spec Intent; acceptance is measured by the
four key results in [key-results/index.md](key-results/index.md).

## Scope

| In scope | Out of scope |
|----------|--------------|
| `billing-svc` (primary): Java 8 + Spring Boot 1.5 → Java 21 + Spring Boot 3.3 | Other downstream services |
| `web-app` (consumer): `ts-morph` call-site updates to follow the seam | UI/UX redesign |
| Payments capability extraction behind a strangler-fig seam | Dunning / Invoicing seam extraction (deferred, future waves) |
| Security + deprecated-API remediation in scope | Database schema redesign |

**Repos**

- **billing-svc** — primary. Java 8 + Spring Boot 1.5 → Java 21 + Spring Boot 3.3.
- **web-app** — consumer. Call-site updates via `ts-morph` to track the seam.

## Constraints

1. **No breaking change to `/api/v1/**`** — public contract is frozen; behavioral
   equivalence is proven against production traces (see
   [e2e/e2e-report.md](e2e/e2e-report.md)).
2. **Max 1 seam per wave** — only the Payments seam is extracted this run
   ([CU-021](change-units/CU-021.md)); Dunning and Invoicing candidates are left
   for future waves.
3. **No data migration** — no schema or data changes; Postgres stays as-is.

## Strategy

Strangler-fig seam for Payments: an **interface-facade** plus a **feature flag
`payments.v2` (default-off)** giving an **O(1) rollback** by flipping the flag,
with a guaranteed v1 fallback path. Planned and sequenced across 4 waves in
[stages/PLAN.md](stages/PLAN.md); the seam itself is [CU-021](change-units/CU-021.md).

## Known dependency / risk

The external payment-provider API is unknown, captured as
[GAP-007](gaps/GAP-007.md) — a typed placeholder `InterfaceContract "POST /charge"`
(confidence 0.30). It is **open but non-blocking**: the seam ships with the v2
path default-off and a Podman mock standing in for the provider.

## Outcome

✅ **COMPLETE** — all four key results MET, certification PASS. See the live front
door at [index.md](index.md) and the scoreboard at
[key-results/index.md](key-results/index.md).
