---
type: flow
title: "Flow — charge-invoice"
description: "Request flow rooted at POST /api/v1/payments: InvoiceService.charge → PaymentGateway (placeholder) → writes INVOICE/PAYMENT."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Flow::charge-invoice
tags: [flow, request, payments, invoicing, placeholder]
links: [index.md, ../entry-points/http-post-invoices.md, ../entry-points/kafka-payment-events.md, ../gaps/GAP-007.md, ../change-units/CU-021.md]
---

# Flow: charge-invoice

The core revenue path: charge a payment against an invoice and persist the
result.

## Properties

| Prop | Value |
|------|-------|
| `gid` | `billing-svc::Flow::charge-invoice` |
| `kind` | request |
| `rootEntryPoint` | `billing-svc::Endpoint::POST /api/v1/payments` |
| `methodSet` | (transitive call set from the handler) |
| `crossRepo` | true — entered from the `web-app` consumer |
| `coverage` | ~0.86 (run avg) |
| `synthetic` | false |

## `ENTERS_AT`

- Root: `POST /api/v1/payments`.
- Related synchronous create surface:
  [`../entry-points/http-post-invoices.md`](../entry-points/http-post-invoices.md).
- Related asynchronous settlement leg (Kafka):
  [`../entry-points/kafka-payment-events.md`](../entry-points/kafka-payment-events.md).

## `FLOWS_THROUGH` (method path)

```
POST /api/v1/payments
  → PaymentController.charge(ChargeRequest)
    → InvoiceService.charge(invoiceId, amount)        # the DEBT hotspot
      → PaymentGateway.charge(ChargeRequest)          # → external payment provider
      → InvoiceRepository.save(invoice)
      → PaymentRepository.save(payment)
```

## `TOUCHES`

| Target | Kind | Access |
|--------|------|--------|
| `INVOICE` | Table | WRITES |
| `PAYMENT` | Table | WRITES |
| **payment-provider** | ExternalSystem (placeholder) | `DEPENDS_ON_EXTERNAL` |

The external **payment-provider** API is unknown. It is represented by a
placeholder `InterfaceContract` (`POST /charge`, confidence 0.30, non-blocking)
and mocked in integration — see [`../gaps/GAP-007.md`](../gaps/GAP-007.md). In
the integration topology it is backed by `mock-payment-provider`
([`../environment/environment.md`](../environment/environment.md)).

## Cross-repo note

This flow is entered from the `web-app` consumer over the HTTP contract, so it is
`crossRepo: true`; discovery continued the trace across the repo boundary rather
than truncating at it.

## Links

- Seam extraction that wraps `PaymentGateway`: [`../change-units/CU-021.md`](../change-units/CU-021.md)
- Placeholder gap: [`../gaps/GAP-007.md`](../gaps/GAP-007.md)
- All flows: [`index.md`](index.md)
