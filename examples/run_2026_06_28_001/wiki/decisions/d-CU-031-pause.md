---
type: decision
title: "Decision — CU-031 pause (security-fix) → approved"
description: "Gate decision for CU-031 (Log4j CVE fix): kind security-fix ∈ requireHumanFor AND risk 0.61 in pause band → PAUSED 10:55; owner approved 13:05; applied."
status: APPLIED
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Decision::d-CU-031-pause
tokens: 0
tags: [decision, pause, human-gate, security-fix, log4j]
links: [index.md, ../change-units/CU-031.md, ../log.md]
---

# Decision — CU-031 → **PAUSE** (human gate) → approved

Gate decision for [CU-031 — Log4j CVE-2021-44228 → 2.17](../change-units/CU-031.md).

## Inputs

| | |
|---|---|
| Risk score | **0.61** |
| Kind | `security-fix` |
| autoApplyCeiling | 0.40 |
| blockAbove | 0.80 |
| requireHumanFor | `[seam-extraction, security-fix]` |

## Gate math

```
1. kind ∈ requireHumanFor ?   security-fix ∈ {seam-extraction, security-fix} → YES → PAUSE
   (short-circuits to PAUSE regardless of risk)

   — risk is also evaluated and independently lands in the pause band: —
2. risk ≥ blockAbove ?         0.61 ≥ 0.80   → NO
3. risk > autoApplyCeiling ?   0.61 > 0.40   → YES  ⇒ 0.40 < 0.61 < 0.80 → PAUSE band
```

**Two independent triggers** require a human gate: the kind is in
`requireHumanFor`, *and* the risk (0.61) is in the pause band.

## Decision

**PAUSE — `requireHumanFor: security-fix`.** Human approval required before apply.

```yaml
decision: pause
requireHumanFor: security-fix
pausedAt: 2026-06-28T10:55:00Z
```

## Rationale

Security fixes are always human-gated by policy — a security change is exactly
where a silent regression is most costly, so even an "obviously correct"
remediation (CVE-2021-44228 / Log4Shell → Log4j 2.17) gets a human eye. The risk
score of 0.61 independently lands in the pause band, reflecting the service-wide
blast radius of a logging dependency.

## Human approval

```yaml
approval:
  approvedAt: 2026-06-28T13:05:00Z
  approver: owner            # billing-svc repo owner
  approverRole: owner
  note: "Log4j 2.17.x closes CVE-2021-44228; logging smoke test green; appenders/levels
         unchanged. Approved to apply."
```

## Outcome

- `10:55:00` — RISK_SCORED → **PAUSED** (`requireHumanFor: security-fix`)
- `13:05:00` — **APPROVED** by **owner**
- **APPLIED** thereafter (wave 1)

See the [run log](../log.md) and [CU-031](../change-units/CU-031.md).
