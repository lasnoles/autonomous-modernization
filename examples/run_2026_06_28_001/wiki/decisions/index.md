---
type: decision
title: "Gate decisions — run_2026_06_28_001"
description: "Every autonomy-gate decision for the run: per-CU risk, the ceiling math, the auto/pause/block outcome, and rationale."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
tags: [decisions, gates, autonomy, index]
links: [d-CU-014-auto.md, d-CU-021-pause.md, d-CU-031-pause.md, ../index.md, ../change-units/index.md, ../log.md]
---

# Gate decisions

Every ChangeUnit passes through the autonomy gate after it is risk-scored. The
gate configuration for this run:

```yaml
gates:
  risk:
    autoApplyCeiling: 0.40
    blockAbove: 0.80
  review:
    requireHumanFor: [seam-extraction, security-fix]
```

> **Decision rule**
> - `risk ≤ 0.40` → **auto-apply**
> - `0.40 < risk < 0.80` → **pause** (human approval required)
> - `risk ≥ 0.80` → **block**
> - `kind ∈ requireHumanFor` (`seam-extraction`, `security-fix`) → **pause regardless of risk**

## All decisions

| CU | Kind | Risk | Ceiling math | Decision | Rationale |
|----|------|-----:|--------------|----------|-----------|
| CU-001 | dependency-upgrade | 0.12 | `0.12 ≤ 0.40` | auto | below ceiling |
| [CU-002](../change-units/CU-002.md) | dependency-decouple | 0.18 | `0.18 ≤ 0.40` | auto | below ceiling |
| CU-003 | dead-code-removal | 0.09 | `0.09 ≤ 0.40` | auto | below ceiling |
| [**CU-031**](d-CU-031-pause.md) | security-fix | 0.61 | `0.40 < 0.61 < 0.80` **and** kind ∈ requireHumanFor | **pause** | security-fix always human-gated; risk also in pause band |
| [**CU-014**](d-CU-014-auto.md) | api-migration | 0.31 | `0.31 ≤ 0.40` | auto | below ceiling; kind not gated |
| CU-015 | language-level | 0.27 | `0.27 ≤ 0.40` | auto | below ceiling |
| CU-016 | test-backfill | 0.19 | `0.19 ≤ 0.40` | auto | below ceiling |
| CU-018 | framework-upgrade | 0.38 | `0.38 ≤ 0.40` | auto | below ceiling (narrowest auto margin) |
| CU-019 | config-migration | 0.14 | `0.14 ≤ 0.40` | auto | below ceiling |
| CU-020 | api-migration | 0.33 | `0.33 ≤ 0.40` | auto | below ceiling |
| [**CU-021**](d-CU-021-pause.md) | seam-extraction | 0.52 | `0.40 < 0.52 < 0.80` **and** kind ∈ requireHumanFor | **pause** | seam-extraction always human-gated; risk also in pause band (incl. +0.15 from [GAP-007](../gaps/GAP-007.md)) |
| CU-022 | pattern-refactor | 0.22 | `0.22 ≤ 0.40` | auto | below ceiling |
| CU-027 | test-backfill | 0.24 | `0.24 ≤ 0.40` | auto | below ceiling |
| CU-030 | config-migration | 0.11 | `0.11 ≤ 0.40` | auto | below ceiling |

**Outcome:** 12 auto-applied, **2 paused** (then approved by the owner at
`13:05`), 0 blocked. No CU reached the `blockAbove (0.80)` threshold.

## Paused decisions (detailed)

- [**d-CU-031-pause**](d-CU-031-pause.md) — Log4j security-fix; paused `10:55`, approved `13:05`.
- [**d-CU-021-pause**](d-CU-021-pause.md) — payments seam-extraction; paused `11:39`, approved `13:05`, applied `13:50`.

## Auto decisions (detailed example)

- [**d-CU-014-auto**](d-CU-014-auto.md) — javax→jakarta; the exact auto-apply math.

## See also

- [ChangeUnits index](../change-units/index.md) · [run log](../log.md) · [run index](../index.md)
