---
type: decision
title: "Decision — CU-021 pause (seam-extraction) → approved"
description: "Gate decision for CU-021 (payments seam): kind seam-extraction ∈ requireHumanFor AND risk 0.52 in pause band → PAUSED 11:39; owner approved 13:05; applied 13:50."
status: APPLIED
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Decision::d-CU-021-pause
tokens: 0
tags: [decision, pause, human-gate, seam-extraction, payments]
links: [index.md, ../change-units/CU-021.md, ../gaps/GAP-007.md, ../key-results/kr-04-payments-seam.md, ../log.md]
---

# Decision — CU-021 → **PAUSE** (human gate) → approved

Gate decision for [CU-021 — payments seam extraction](../change-units/CU-021.md).

## Inputs

| | |
|---|---|
| Risk score | **0.52** (base 0.37 **+ 0.15** from [GAP-007](../gaps/GAP-007.md)) |
| Kind | `seam-extraction` |
| autoApplyCeiling | 0.40 |
| blockAbove | 0.80 |
| requireHumanFor | `[seam-extraction, security-fix]` |

## Gate math

```
1. kind ∈ requireHumanFor ?   seam-extraction ∈ {seam-extraction, security-fix} → YES → PAUSE
   (short-circuits to PAUSE regardless of risk)

   — risk is also evaluated and independently lands in the pause band: —
2. risk ≥ blockAbove ?         0.52 ≥ 0.80   → NO
3. risk > autoApplyCeiling ?   0.52 > 0.40   → YES  ⇒ 0.40 < 0.52 < 0.80 → PAUSE band
```

**Two independent triggers** both require a human gate: the kind is in
`requireHumanFor`, *and* the risk (0.52) is in the pause band.

## Decision

**PAUSE — `requireHumanFor: seam-extraction`.** Human approval required before
apply.

```yaml
decision: pause
requireHumanFor: seam-extraction
pausedAt: 2026-06-28T11:39:10Z
```

## Rationale

Seam extraction on the highest-criticality component (Payments), built against a
**placeholder** contract ([GAP-007](../gaps/GAP-007.md), confidence 0.30) whose
gapRisk (+0.15) is exactly what pushes the score into the pause band. Equivalence
is golden-replay *against the placeholder*, not the real provider
(`equivalenceConfidence 0.65`). This is precisely the kind of judgment call the
human gate exists for — even though the apply is safe (flag default-off, O(1)
rollback).

## Human approval

```yaml
approval:
  approvedAt: 2026-06-28T13:05:00Z
  approver: owner            # billing-svc repo owner
  approverRole: owner
  note: "Seam is default-off (payments.v2=off); O(1) flag rollback; golden replay green
         against mock-payment-provider. Approved to apply; cutover gated separately."
```

## Outcome

- `11:39:10` — RISK_SCORED → **PAUSED** (`requireHumanFor: seam-extraction`)
- `13:05:00` — **APPROVED** by **owner**
- `13:50:00` — **APPLIED** (feature flag `payments.v2` default-off)

See the [run log](../log.md), [CU-021](../change-units/CU-021.md), and the key
result [kr-04-payments-seam](../key-results/kr-04-payments-seam.md).
