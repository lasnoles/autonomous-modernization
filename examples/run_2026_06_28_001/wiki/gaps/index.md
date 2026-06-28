---
type: gap
title: "Gaps & Assumptions register — run_2026_06_28_001"
description: "Every unresolved unknown and every typed placeholder for this run. No gap is invisible; placeholders are never skipped."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
tags: [gaps, assumptions, placeholders, index]
links: [GAP-007.md, ../index.md, ../change-units/CU-021.md, ../environment/environment.md, ../e2e/e2e-report.md, ../log.md]
---

# Gaps & Assumptions register

Autonomy means **resolving our own open questions** — honestly. When the
resolution ladder (human gate → spec → reality/code → contracts → reasoned
inference) yields no answer, the unknown is **classified and recorded**, never
ignored. Every unresolved unknown and every placeholder lives here as a `Gap`.

> **Invariant — never silently skip an integration.** Missing info becomes a
> **typed placeholder + a Gap**, not an omission. A repo with an unknown
> dependency still ships a buildable, mockable, testable integration surface.
> Every placeholder has a Gap; every Gap has a `whatWouldResolveIt`.

## Register

| Gap | Category | Subject | Status | Confidence | Blocking | raisesRiskBy | whatWouldResolveIt |
|-----|----------|---------|--------|-----------:|----------|-------------:|--------------------|
| [GAP-007](GAP-007.md) | unknown-interface | PaymentProvider external API | placeholdered (OPEN) | 0.30 | **no** | +0.15 | Provider OpenAPI spec, **or** one captured prod request/response pair |

One gap was registered this run. It is **open** but **non-blocking**: rather than
stalling, the system built the payments integration against a **typed
placeholder** and let it raise the risk of the dependent ChangeUnit
([CU-021](../change-units/CU-021.md)) so the gate would treat it with appropriate
caution.

## How gaps feed the run

- **Placeholders are never skipped.** GAP-007's placeholder `InterfaceContract`
  is fully wired: the seam ([CU-021](../change-units/CU-021.md)) is built to it,
  the `mock-payment-provider` container ([environment](../environment/environment.md))
  serves it, and the [E2E suite](../e2e/e2e-report.md) exercises it.
- **Gaps raise risk.** GAP-007 `raisesRiskBy 0.15`, which is what tips CU-021
  from 0.37 to **0.52** — keeping the change in the human-gated band.
- **Gaps are replaced, not forgotten.** When the provider's real contract
  arrives, GAP-007 is resolved, the placeholder superseded, and everything built
  on it is re-validated.

## See also

- [GAP-007](GAP-007.md) — full detail
- [CU-021](../change-units/CU-021.md) — the ChangeUnit that depends on this gap
- [environment](../environment/environment.md) — the mock that runtime-backs the placeholder
