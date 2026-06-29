---
type: entry-point
title: "POST /api/v1/invoices — create invoice"
description: "HTTP entry point on billing-svc that creates an invoice and roots the charge-invoice flow."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Endpoint::POST /api/v1/invoices
tags: [entry-point, http, invoicing]
links: [index.md, ../flows/flow-charge-invoice.md, ../stages/stage-INDEX.md]
---

# Endpoint: `POST /api/v1/invoices`

The primary HTTP surface for creating an invoice and initiating its charge.

## Properties

| Prop | Value |
|------|-------|
| `gid` | `billing-svc::Endpoint::POST /api/v1/invoices` |
| `entryClass` | http |
| `protocol` | http |
| `verb` | POST |
| `path` | `/api/v1/invoices` |
| `mediaTypes` | `application/json` |
| `handler` | `InvoiceController.createInvoice(CreateInvoiceRequest)` |
| `synthetic` | false |

## Handler

`HANDLED_BY` → `InvoiceController.createInvoice` → delegates to
`InvoiceService.charge` (the `InvoiceService` hotspot from DEBT).

## Flow

Roots the **charge-invoice** flow:
[`../flows/flow-charge-invoice.md`](../flows/flow-charge-invoice.md).
That flow reaches the placeholder payment provider
([`../gaps/GAP-007.md`](../gaps/GAP-007.md)).

## Links

- Inventory: [`index.md`](index.md)
- Flow: [`../flows/flow-charge-invoice.md`](../flows/flow-charge-invoice.md)
- Indexed by: [`../stages/stage-INDEX.md`](../stages/stage-INDEX.md)
