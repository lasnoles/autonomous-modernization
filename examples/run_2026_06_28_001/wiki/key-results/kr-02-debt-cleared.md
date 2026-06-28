---
type: key-result
title: "KR-02 — Zero deprecated API + zero high-sev vuln in scope"
description: "javax usages and high-severity vulnerabilities in scope eliminated; Log4j patched. MET."
status: MET
runId: run_2026_06_28_001
repo: billing-svc
gid: run::run_2026_06_28_001::kr::KR-02
tags: [key-result, debt, security, jakarta, log4j]
links:
  - index.md
  - ../change-units/CU-014.md
  - ../change-units/CU-031.md
  - ../stages/DEBT.md
  - ../decisions/d-CU-031-pause.md
---

# KR-02 — Zero deprecated API + zero high-sev vuln in scope

**Status: ✅ MET**

## Acceptance criterion

Within the migration scope: **zero deprecated `javax.*` API usages** remaining
and **zero high-severity vulnerabilities**.

## Result

| Metric | Before | After |
|--------|-------:|------:|
| `javax.*` usages in scope | 12 | **0** |
| High-severity vulnerabilities in scope | 3 | **0** |
| Critical CVE (Log4j CVE-2021-44228) | present | **patched** |

DEBT identified 58 debt items and 3 CVEs (top hotspot `InvoiceService`); the
in-scope deprecated-API and high-severity items were all remediated.

## Evidence

| Evidence | Link |
|----------|------|
| Debt + CVE inventory (58 items, 3 CVEs) | [stages/DEBT.md](../stages/DEBT.md) |
| `javax → jakarta` migration (12 → 0) | [CU-014](../change-units/CU-014.md) |
| Security-fix Log4j → 2.17 (CVE-2021-44228) | [CU-031](../change-units/CU-031.md) |
| Human approval of the security-fix gate | [d-CU-031-pause.md](../decisions/d-CU-031-pause.md) |

## Notes

- [CU-031](../change-units/CU-031.md) is a `security-fix`, so it tripped the
  `requireHumanFor: security-fix` gate (risk 0.61) — PAUSED at 10:51, **approved
  13:05**, APPLIED 13:12. See [d-CU-031-pause.md](../decisions/d-CU-031-pause.md).
- Out-of-scope debt items remain tracked but were not part of this KR.
