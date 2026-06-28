---
type: flow
title: "Flow — dunning-sweep"
description: "Scheduled flow rooted at the @Scheduled dunning job: DunningService reads INVOICE and emits the invoices Kafka topic."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Flow::dunning-sweep
tags: [flow, scheduled, dunning, kafka]
links: [index.md, ../entry-points/scheduled-dunning-run.md, ../stages/RECOVER.md]
---

# Flow: dunning-sweep

The recurring collections path: find overdue invoices and emit dunning notices.

## Properties

| Prop | Value |
|------|-------|
| `gid` | `billing-svc::Flow::dunning-sweep` |
| `kind` | scheduled |
| `rootEntryPoint` | `billing-svc::Endpoint::cron:dunning-run` |
| `methodSet` | (transitive call set from the job method) |
| `crossRepo` | false |
| `coverage` | ~0.86 (run avg) |
| `synthetic` | false |

## `ENTERS_AT`

- Root: the `@Scheduled` dunning job —
  [`../entry-points/scheduled-dunning-run.md`](../entry-points/scheduled-dunning-run.md).

## `FLOWS_THROUGH` (method path)

```
@Scheduled dunning job
  → DunningJob.runDunningSweep()
    → DunningService.sweepOverdue()
      → InvoiceRepository.findOverdue()               # reads INVOICE
      → DunningService.buildNotices(invoices)
      → KafkaTemplate.send("invoices", notice)        # emits invoices topic
```

## `TOUCHES`

| Target | Kind | Access |
|--------|------|--------|
| `INVOICE` | Table | READS |
| `invoices` | Kafka topic (ExternalSystem) | emits / `DEPENDS_ON_EXTERNAL` |

This flow does **not** touch the placeholder payment-provider — its only external
dependency is the broker it publishes to.

## Links

- Root entry point: [`../entry-points/scheduled-dunning-run.md`](../entry-points/scheduled-dunning-run.md)
- Dunning seam candidate (extractability 0.55) was identified in RECOVER but
  **deferred** this run (max one seam per wave) — no ChangeUnit. See
  [`../stages/RECOVER.md`](../stages/RECOVER.md).
- All flows: [`index.md`](index.md)
