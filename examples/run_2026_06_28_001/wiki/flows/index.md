---
type: flow
title: "Flows — coverage of every entry point"
description: "All 41 traced flows for billing-svc; avg coverage 0.86; every one of the 37 entry points has ≥1 flow."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
tags: [flows, coverage, discover]
links: [index.md, ../entry-points/index.md, ../stages/DISCOVER.md, ../log.md]
---

# Flows

A **Flow** is an execution slice rooted at one entry point — the transitive call
graph from that entry through methods into the data and external systems it
touches (`architecture/entry-point-discovery.md` §3).

## Summary (CANON)

| Metric | Value |
|--------|-------|
| Flows traced | **41** |
| Avg coverage | **0.86** |
| Entry points covered | **37 / 37** — every entry point has ≥1 flow |
| Cross-repo flows | continued into `web-app` where contracts cross the boundary |

Completeness invariant satisfied: no entry point has zero flows. Some entry
points yield several flows (branches, error paths), which is why 37 entry points
produce 41 flows.

## Example flows

| Flow | Root entry point | Kind | Touches |
|------|------------------|------|---------|
| [flow-charge-invoice](flow-charge-invoice.md) | `POST /api/v1/payments` | request | `INVOICE`, `PAYMENT`, payment-provider (placeholder) |
| [flow-dunning-sweep](flow-dunning-sweep.md) | `@Scheduled` dunning job | scheduled | `INVOICE` (read), `invoices` Kafka topic (emit) |

## Links

- Entry-point inventory: [`../entry-points/index.md`](../entry-points/index.md)
- Traced by stage: [`../stages/DISCOVER.md`](../stages/DISCOVER.md)
- Placeholder external on the charge flow: [`../gaps/GAP-007.md`](../gaps/GAP-007.md)
