---
type: stage
title: "EXECUTE — applying the 14 ChangeUnits"
description: "All 14 ChangeUnits APPLIED; CU-031 & CU-021 (seam-extraction) applied via human approval at 13:05. Roles developer/devops/tester ran per CU."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Stage::EXECUTE
tokens: 1071500
tags: [stage, execute, change-units, human-approval]
links: [index.md, PLAN.md, INTEGRATE.md, ../change-units/index.md, ../decisions/index.md, ../log.md, ../tokens.md]
---

# Stage: EXECUTE

Applies each `ChangeUnit` under its risk gate: validate → risk-score → apply
(auto, or after human approval where required). Roles **developer / devops /
tester** run per CU.

## What it did

- Executed all 14 ChangeUnits across the 4 planned waves.
- Auto-applied units at or below `autoApplyCeiling` (0.40).
- Paused the gated units for human approval, then applied on sign-off.
- Re-validated behavioral equivalence after each apply.

## Key numbers (CANON)

| Metric | Value |
|--------|-------|
| ChangeUnits applied | **14 / 14** |
| Applied via human approval | CU-031, CU-021 (seam-extraction) at **13:05** |

The two seam-extraction units hit `requireHumanFor: [seam-extraction]` and were
APPLIED only after explicit human approval — see the decisions log. This is the
single most token-intensive stage of the run.

## Tokens

**1,071.5k** — by far the largest stage; see [`../tokens.md`](../tokens.md).

## Links

- All ChangeUnits: [`../change-units/index.md`](../change-units/index.md)
- Human-approved seam units: [`../change-units/CU-031.md`](../change-units/CU-031.md),
  [`../change-units/CU-021.md`](../change-units/CU-021.md)
- Jakarta migration: [`../change-units/CU-014.md`](../change-units/CU-014.md) ·
  Log4j fix: [`../change-units/CU-031.md`](../change-units/CU-031.md)
- Gate decisions: [`../decisions/index.md`](../decisions/index.md)
- Previous stage: [PLAN](PLAN.md) · Next stage: [INTEGRATE](INTEGRATE.md)
