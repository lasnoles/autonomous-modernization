---
type: entry-point
title: "@Scheduled dunning run"
description: "Scheduled entry point on billing-svc that sweeps overdue invoices; roots the dunning-sweep flow."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Endpoint::cron:dunning-run
tags: [entry-point, scheduled, dunning]
links: [index.md, ../flows/flow-dunning-sweep.md, ../stages/stage-INDEX.md]
---

# Endpoint: `@Scheduled` dunning run

A scheduled (cron) entry point that periodically sweeps overdue invoices and
drives the dunning process.

## Properties

| Prop | Value |
|------|-------|
| `gid` | `billing-svc::Endpoint::cron:dunning-run` |
| `entryClass` | scheduled |
| `protocol` | cron |
| `schedule` | `@Scheduled(cron = "0 0 2 * * *")` (daily 02:00) |
| `handler` | `DunningJob.runDunningSweep()` |
| `synthetic` | false |

## Handler

`HANDLED_BY` → `DunningJob.runDunningSweep` → `DunningService` → reads overdue
invoices and emits dunning notices.

## Flow

Roots the **dunning-sweep** flow:
[`../flows/flow-dunning-sweep.md`](../flows/flow-dunning-sweep.md). That flow
reads the `INVOICE` table and emits to the `invoices` Kafka topic.

## Links

- Inventory: [`index.md`](index.md)
- Flow: [`../flows/flow-dunning-sweep.md`](../flows/flow-dunning-sweep.md)
- Indexed by: [`../stages/stage-INDEX.md`](../stages/stage-INDEX.md)
