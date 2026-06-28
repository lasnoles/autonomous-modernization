---
type: decision
title: "Decision — CU-014 auto-apply"
description: "Gate decision for CU-014 (javax→jakarta): risk 0.31 ≤ autoApplyCeiling 0.40, kind not in requireHumanFor → auto-apply, no human gate."
status: APPLIED
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Decision::d-CU-014-auto
tokens: 0
tags: [decision, auto-apply, api-migration]
links: [index.md, ../change-units/CU-014.md, ../log.md]
---

# Decision — CU-014 → **auto-apply**

Gate decision for [CU-014 — javax→jakarta](../change-units/CU-014.md).

## Inputs

| | |
|---|---|
| Risk score | **0.31** |
| Kind | `api-migration` |
| autoApplyCeiling | 0.40 |
| blockAbove | 0.80 |
| requireHumanFor | `[seam-extraction, security-fix]` |

## Gate math

```
1. kind ∈ requireHumanFor ?   api-migration ∈ {seam-extraction, security-fix} → NO
2. risk ≥ blockAbove ?         0.31 ≥ 0.80                                     → NO
3. risk > autoApplyCeiling ?   0.31 > 0.40                                     → NO
4. ⇒ risk ≤ autoApplyCeiling   0.31 ≤ 0.40                                     → AUTO-APPLY
```

## Decision

**AUTO-APPLY** — no human gate.

`0.31 ≤ 0.40` places the change in the auto-apply band, and `api-migration` is
not in `requireHumanFor`. The deterministic OpenRewrite recipe with proven
behavioral equivalence carries no factor that would push it into the human-gated
band.

## Rationale

A namespace-only recipe migration with behavioral equivalence proven by the
existing suite (regenerated where coverage was missing) and no PRESERVES-rule
risk. There is no upside to a human gate here; the gate's purpose is to surface
*judgment* calls, and this is mechanical.

## Outcome

`11:02:55` — VALIDATED → RISK_SCORED → **APPLIED** (auto). See the
[run log](../log.md) and [CU-014](../change-units/CU-014.md).
