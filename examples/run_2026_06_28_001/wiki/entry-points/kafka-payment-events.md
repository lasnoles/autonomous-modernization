---
type: entry-point
title: "payment-events — Kafka consumer"
description: "Messaging entry point on billing-svc consuming the payment-events topic; participates in the charge-invoice flow."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Endpoint::kafka:payment-events
tags: [entry-point, messaging, kafka, payments]
links: [index.md, ../flows/flow-charge-invoice.md, ../stages/stage-INDEX.md]
---

# Endpoint: `payment-events` (Kafka consumer)

A messaging entry point that consumes payment lifecycle events and reconciles
them against invoices.

## Properties

| Prop | Value |
|------|-------|
| `gid` | `billing-svc::Endpoint::kafka:payment-events` |
| `entryClass` | messaging |
| `protocol` | kafka |
| `topic` | `payment-events` |
| `handler` | `PaymentEventListener.onPaymentEvent(PaymentEvent)` (`@KafkaListener`) |
| `synthetic` | false |

## Handler

`HANDLED_BY` → `PaymentEventListener.onPaymentEvent` → `PaymentReconciler` →
updates `PAYMENT` / `INVOICE` state.

## Flow

Participates in the **charge-invoice** flow (the asynchronous settlement leg):
[`../flows/flow-charge-invoice.md`](../flows/flow-charge-invoice.md). Like the
synchronous leg, it depends on the placeholder payment provider
([`../gaps/GAP-007.md`](../gaps/GAP-007.md)).

## Links

- Inventory: [`index.md`](index.md)
- Flow: [`../flows/flow-charge-invoice.md`](../flows/flow-charge-invoice.md)
- Indexed by: [`../stages/stage-INDEX.md`](../stages/stage-INDEX.md)
