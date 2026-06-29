---
type: stage
title: "RECOVER — architecture recovery"
description: "Recovered 9 logical components and scored strangler-fig seams; Payments is the most extractable (0.78)."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Stage::RECOVER
tokens: 31400
tags: [stage, recover, architecture, seams, strangler-fig]
links: [index.md, stage-INDEX.md, DISCOVER.md, ../flows/index.md, ../log.md, ../tokens.md]
---

# Stage: RECOVER

Clusters the L0 graph into logical components (L1) and scores candidate
strangler-fig seams — the boundaries along which functionality can be extracted.

## What it did

- Clustered classes into 9 recovered `Component` nodes by cohesion/coupling.
- Scored `Seam` extractability for the extraction objective (Payments).
- Detected layering violations (dependencies crossing layers the wrong way).

## Key numbers (CANON)

| Metric | Value |
|--------|-------|
| Recovered components | **9** |
| Layering violations | 2 |

Seam extractability (0–1):

| Seam | Extractability |
|------|----------------|
| **Payments** | **0.78** |
| Dunning | 0.55 |
| Invoicing | 0.40 |

Payments scores highest, which directly supports the objective to extract
Payments behind a seam. This drove the seam-extraction ChangeUnit in PLAN —
see [`../change-units/CU-021.md`](../change-units/CU-021.md).

## Tokens

**31.4k** — see [`../tokens.md`](../tokens.md).

## Links

- Charge flow that crosses the Payments seam: [`../flows/flow-charge-invoice.md`](../flows/flow-charge-invoice.md)
- Capabilities derived next: [DISCOVER](DISCOVER.md)
- Previous stage: [INDEX](stage-INDEX.md)
