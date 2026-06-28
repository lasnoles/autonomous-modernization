---
type: stage
title: "PLAN — ChangeUnit planning & wave ordering"
description: "Produced 14 ChangeUnits across 4 waves with risk gates: autoApplyCeiling 0.40 / blockAbove 0.80 / requireHumanFor seam-extraction & security-fix."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Stage::PLAN
tokens: 91200
tags: [stage, plan, change-units, waves, risk-gates]
links: [index.md, DEBT.md, EXECUTE.md, ../change-units/index.md, ../decisions/index.md, ../log.md, ../tokens.md]
---

# Stage: PLAN

Turns recovered architecture, capabilities, and debt into an ordered plan of
atomic `ChangeUnit`s grouped into `Wave`s, with explicit risk gates.

## What it did

- Sized 14 ChangeUnits using flow blast-radius.
- Ordered them into 4 waves (dependency-respecting, parallelizable where safe).
- Configured the risk gate policy that EXECUTE enforces.

## Key numbers (CANON)

| Metric | Value |
|--------|-------|
| ChangeUnits | **14** |
| Waves | **4** |

Risk gate policy:

| Gate | Value | Meaning |
|------|-------|---------|
| `autoApplyCeiling` | 0.40 | risk ≤ 0.40 → auto-apply |
| `blockAbove` | 0.80 | risk > 0.80 → hard block |
| `requireHumanFor` | `[seam-extraction, security-fix]` | always pause for human approval |

`requireHumanFor` is why the Payments seam-extraction unit (CU-021) and the
Log4j security fix (CU-031) route through human approval regardless of their
numeric risk score.

## Approach (execution playbooks)

PLAN also selected an **execution playbook** per ChangeUnit and emitted the
**approach proposal** for the gate (`architecture/execution-playbooks.md`):

| Scope | Playbook | Trigger |
|-------|----------|---------|
| Payments (CU-021) | `seam-extraction` | objective extracts Payments; extractability 0.78 |
| default (all other CUs) | `strangler-fig` | default |

No `language-port` (target stays Java), no `characterization-first` (coverage
adequate), no `dependency-untangle`. With `approachGate: review`, the non-default
`seam-extraction` selection surfaced a one-line approach proposal at `10:25`,
**approved by min_than**, then EXECUTE proceeded. See
[`../decisions/index.md`](../decisions/index.md).

## Tokens

**91.2k** — see [`../tokens.md`](../tokens.md).

## Links

- All ChangeUnits + status/risk: [`../change-units/index.md`](../change-units/index.md)
- Seam-extraction unit: [`../change-units/CU-021.md`](../change-units/CU-021.md)
- Security-fix unit: [`../change-units/CU-031.md`](../change-units/CU-031.md)
- Gate decisions log: [`../decisions/index.md`](../decisions/index.md)
- Previous stage: [DEBT](DEBT.md) · Next stage: [EXECUTE](EXECUTE.md)
