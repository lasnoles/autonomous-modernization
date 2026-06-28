---
type: entry-point
title: "Entry-point inventory — no single front door"
description: "The complete entry-point inventory for billing-svc: 37 non-synthetic surfaces across 8 classes, plus 14 synthetic fixtures excluded."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
tags: [entry-points, inventory, coverage]
links: [index.md, ../stages/stage-INDEX.md, ../flows/index.md, ../log.md]
---

# Entry-point inventory

Legacy systems rarely have a single `main()`. `billing-svc` is entered through
**37** distinct non-synthetic surfaces. Discovery enumerates *every* one — there
is no assumed front door (`architecture/entry-point-discovery.md` §1).

Every entry point has **≥1 traced flow** (completeness invariant) — see
[`../flows/index.md`](../flows/index.md).

## By entry class (CANON)

| `entryClass` | Count | Example detection |
|--------------|-------|-------------------|
| http | **18** | `@RestController` / `@RequestMapping` |
| messaging | **6** | `@KafkaListener`, `@RabbitListener` |
| scheduled | **4** | `@Scheduled`, Quartz `Job` |
| startup | **3** | `@PostConstruct`, `ApplicationReadyEvent` |
| webhook | **2** | inbound callback routes |
| batch | **2** | Spring Batch `Job` / `CommandLineRunner` |
| cli/main | **1** | `public static void main` |
| library | **1** | `isPublicApi` exported API consumed by `web-app` |
| **Total** | **37** | |

### Excluded

| Tag | Count | Treatment |
|-----|-------|-----------|
| synthetic (test fixtures) | **14** | detected but tagged `synthetic`; excluded from production flows, retained for coverage measurement |

## Example entry points

| Endpoint | Class | Protocol | Flow |
|----------|-------|----------|------|
| [POST /api/v1/invoices](http-post-invoices.md) | http | http | [flow-charge-invoice](../flows/flow-charge-invoice.md) |
| [payment-events listener](kafka-payment-events.md) | messaging | kafka | [flow-charge-invoice](../flows/flow-charge-invoice.md) |
| [scheduled dunning run](scheduled-dunning-run.md) | scheduled | cron | [flow-dunning-sweep](../flows/flow-dunning-sweep.md) |

## Links

- Built by stage: [`../stages/stage-INDEX.md`](../stages/stage-INDEX.md)
- Flows traced from these entry points: [`../flows/index.md`](../flows/index.md)
