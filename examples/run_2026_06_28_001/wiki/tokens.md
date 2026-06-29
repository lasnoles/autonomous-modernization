---
type: tokens
title: "Token usage rollup — run_2026_06_28_001"
description: "Per-stage token rollup and run totals; EXECUTE breaks down across change-units and roles."
status: COMPLETE
runId: run_2026_06_28_001
gid: run::run_2026_06_28_001::tokens
model: claude-opus-4-8
tokens: 1423900
tags: [tokens, cost, rollup]
links:
  - index.md
  - stages/index.md
  - stages/EXECUTE.md
  - change-units/index.md
  - log.md
---

# Token usage — `run_2026_06_28_001`

> Live token rollup, folded from the event stream (one attribution per LLM call,
> keyed by repo · stage · changeUnit · role). Model **`claude-opus-4-8`**.

## Run totals

| Metric | Value |
|--------|-------|
| **Total tokens** | **1,423,900** |
| Input | 1,131,200 |
| Output | 292,700 |
| **Estimated cost** | **~$22.40** |

## Per-stage rollup

| Stage | Tokens | Notes |
|-------|-------:|-------|
| [INDEX](stages/stage-INDEX.md) | 48,100 | 37 entry points, 1,204 nodes |
| [RECOVER](stages/RECOVER.md) | 31,400 | 9 components, 3 candidate seams |
| [DISCOVER](stages/DISCOVER.md) | 27,300 | 6 capabilities, 41 rules |
| [DEBT](stages/DEBT.md) | 19,000 | 58 debt items, 3 CVEs |
| [PLAN](stages/PLAN.md) | 91,200 | 14 ChangeUnits, 4 waves |
| [EXECUTE](stages/EXECUTE.md) | 1,071,500 | 14 CUs across 4 waves (breakdown below) |
| [INTEGRATE](stages/INTEGRATE.md) | 120,400 | Podman topology, E2E 38/38, SLOs |
| historian / overhead | 15,000 | bundle authoring, orchestration |
| **Total** | **1,423,900** | in 1,131,200 / out 292,700 |

> INDEX is the largest non-EXECUTE stage; **EXECUTE dominates at 1,071.5k** — it
> is the only stage that breaks down across change-units and roles.

## EXECUTE — breakdown across change-units

EXECUTE attributes tokens to each ChangeUnit (and within a CU, across roles such
as transform, validate, risk-score). The named CUs below are the load-bearing
ones; the remaining wave-1/2/4 CUs make up the balance to the 1,071.5k total.

| CU | Wave | What | Risk | Decision | Tokens |
|----|------|------|------|----------|-------:|
| [CU-002](change-units/CU-002.md) | 1 | decouple-payments-deps | 0.18 | auto | 21,000 |
| [CU-031](change-units/CU-031.md) | 1 | security-fix Log4j → 2.17 | 0.61 | human-approved | 34,500 |
| [CU-014](change-units/CU-014.md) | 2 | javax → jakarta | 0.31 | auto | 47,200 |
| [CU-018](change-units/CU-018.md) | 2 | UpgradeSpringBoot_3_x | 0.38 | auto | 88,100 |
| [CU-021](change-units/CU-021.md) | 3 | payments seam-extraction | 0.52 | human-approved | 63,400 |
| [CU-022](change-units/CU-022.md) | 3 | field → constructor injection | 0.22 | auto | 18,900 |
| _other CUs (8)_ | 1–4 | remaining transforms (auto, risk < 0.40) | — | auto | 798,400 |
| **EXECUTE total** | | | | | **1,071,500** |

## Notes

- Token counters are event-sourced and resume-safe: the rollup spans the whole
  run across any restarts (`run-state-observability.md` §3).
- Per-stage **token budgets** can gate a runaway stage; none tripped this run.
- Chronological per-transition token deltas are in [log.md](log.md).
