---
type: stage
title: "DISCOVER — business discovery & flow exploration"
description: "Traced 41 flows from all 37 entry points (avg coverage 0.86) and lifted 6 capabilities and 41 business rules."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Stage::DISCOVER
tokens: 27300
tags: [stage, discover, flows, capabilities, business-rules]
links: [index.md, stage-INDEX.md, DEBT.md, ../flows/index.md, ../entry-points/index.md, ../log.md, ../tokens.md]
---

# Stage: DISCOVER

Explores **all reachable flows** from the entry-point inventory and lifts each
flow into the business capability and rules it realizes
(`architecture/entry-point-discovery.md` §3).

## What it did

- Rooted a `Flow` traversal at every one of the 37 entry points and walked
  `CALLS*` to build each reachable method set.
- Recorded data touchpoints (`READS`/`WRITES`) and external touchpoints
  (`DEPENDS_ON_EXTERNAL`, including the placeholder payment-provider).
- Lifted flows into `Capability` and `BusinessRule` nodes.

## Key numbers (CANON)

| Metric | Value |
|--------|-------|
| Flows traced | **41** |
| Avg flow coverage | **0.86** |
| Entry points covered | 37 / 37 (every entry point has ≥1 flow) |
| Capabilities | **6** |
| Business rules | **41** |

Capabilities by criticality:

| Capability | Criticality |
|------------|-------------|
| Invoicing | high |
| Payments | high |
| Dunning | med |
| Customer | med |
| Reporting | low |
| Notifications | low |

## Tokens

**27.3k** — see [`../tokens.md`](../tokens.md).

## Links

- All flows + coverage: [`../flows/index.md`](../flows/index.md)
- Example flows: [`../flows/flow-charge-invoice.md`](../flows/flow-charge-invoice.md),
  [`../flows/flow-dunning-sweep.md`](../flows/flow-dunning-sweep.md)
- Placeholder external surfaced here: [`../gaps/GAP-007.md`](../gaps/GAP-007.md)
- Previous stage: [INDEX](stage-INDEX.md) · Next stage: [DEBT](DEBT.md)
