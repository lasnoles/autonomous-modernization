---
type: key-result
title: "KR-04 — Payments behind seam with v1 fallback + O(1) rollback"
description: "Payments capability extracted behind a strangler-fig seam with feature flag payments.v2 (default-off) and O(1) flip-flag rollback. MET; GAP-007 open non-blocking."
status: MET
runId: run_2026_06_28_001
repo: billing-svc
gid: run::run_2026_06_28_001::kr::KR-04
tags: [key-result, strangler-fig, payments, seam, feature-flag, rollback]
links:
  - index.md
  - ../change-units/CU-021.md
  - ../change-units/CU-002.md
  - ../decisions/d-CU-021-pause.md
  - ../gaps/GAP-007.md
  - ../stages/RECOVER.md
---

# KR-04 — Payments behind seam with v1 fallback + O(1) rollback

**Status: ✅ MET** (with [GAP-007](../gaps/GAP-007.md) open, **non-blocking**)

## Acceptance criterion

The **Payments** capability is extracted behind a **strangler-fig seam** with a
guaranteed **v1 fallback** and an **O(1) rollback**.

## Result

Payments is extracted via an **interface-facade** plus a **feature flag
`payments.v2` (default-off)**. The v1 path remains the live default, so the
fallback is the running behavior; switching to v2 (or back) is a single flag
flip — **O(1) rollback**. RECOVER had ranked Payments as the strongest seam
candidate (cohesion 0.78), consistent with the "max 1 seam per wave" constraint.

| Property | Value |
|----------|-------|
| Seam mechanism | interface-facade |
| Feature flag | `payments.v2` — **default-off** |
| v1 fallback | live default path |
| Rollback | **O(1)** — flip `payments.v2` flag |
| Seam candidate rank (RECOVER) | Payments **0.78** (highest of 3) |

## Evidence

| Evidence | Link |
|----------|------|
| Payments seam-extraction ChangeUnit | [CU-021](../change-units/CU-021.md) |
| Prerequisite: decouple payments deps | [CU-002](../change-units/CU-002.md) |
| Human approval of the seam-extraction gate (approved 13:05, APPLIED 13:50) | [d-CU-021-pause.md](../decisions/d-CU-021-pause.md) |
| Seam candidates ranked at RECOVER | [stages/RECOVER.md](../stages/RECOVER.md) |

## Open gap (non-blocking)

[GAP-007](../gaps/GAP-007.md) — the external payment-provider API is unknown,
captured as a typed placeholder `InterfaceContract "POST /charge"` (confidence
0.30), backed by the Podman mock `mock-payment-provider`. It raises risk by
+0.15 and is a dependency of [CU-021](../change-units/CU-021.md), but it is
**OPEN and non-blocking**: the seam ships with `payments.v2` default-off, so the
unknown provider is never on the live path. **It does not gate KR-04.**

What would resolve it: the provider's OpenAPI spec, **or** one captured
production request/response pair. See [GAP-007](../gaps/GAP-007.md).
