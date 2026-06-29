---
type: stage
title: "DEBT — technical-debt & vulnerability analysis"
description: "Catalogued 58 debt items and 3 CVEs (incl. Log4j CVE-2021-44228); InvoiceService is the top hotspot."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Stage::DEBT
tokens: 19000
tags: [stage, debt, vulnerabilities, cve, hotspot]
links: [index.md, DISCOVER.md, PLAN.md, ../change-units/index.md, ../log.md, ../tokens.md]
---

# Stage: DEBT

Quantifies technical debt (L3): deprecated APIs, outdated dependencies, security
vulnerabilities, and high-risk hotspots that the planner must prioritize.

## What it did

- Catalogued `DebtItem` nodes across the codebase (deprecated-api, outdated-dep,
  complexity, test-gap, architecture-violation, config).
- Matched dependencies against advisories to surface `Vulnerability` (CVE) nodes.
- Scored `Hotspot` nodes (churn × complexity).

## Key numbers (CANON)

| Metric | Value |
|--------|-------|
| Debt items | **58** |
| CVEs | **3** |
| Top hotspot | `InvoiceService` |

Notable CVE: **Log4j CVE-2021-44228** (Log4Shell) — drives a security-fix
ChangeUnit, which is gated `requireHumanFor: [security-fix]` (see PLAN).

The `InvoiceService` hotspot concentrates change risk and sits on the
charge-invoice flow — see [`../flows/flow-charge-invoice.md`](../flows/flow-charge-invoice.md).

## Tokens

**19.0k** — see [`../tokens.md`](../tokens.md).

## Links

- ChangeUnits planned to retire this debt: [`../change-units/index.md`](../change-units/index.md)
- Java/Jakarta API migration unit: [`../change-units/CU-014.md`](../change-units/CU-014.md)
- Security-fix unit (Log4j): [`../change-units/CU-031.md`](../change-units/CU-031.md)
- Previous stage: [DISCOVER](DISCOVER.md) · Next stage: [PLAN](PLAN.md)
