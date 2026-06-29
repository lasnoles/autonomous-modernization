---
type: run
title: "billing-jakarta-payments-2026q3 — run_2026_06_28_001"
description: "Live STATUS front door for the billing-svc Java 21 + Spring Boot 3.x migration and Payments strangler-fig seam extraction. Run COMPLETE."
status: COMPLETE
runId: run_2026_06_28_001
owner: min_than_zaw@msi-global.com.sg
environment: staged
gid: run::run_2026_06_28_001
tokens: 1423900
tags: [run, billing-svc, web-app, java-21, spring-boot-3, strangler-fig, payments]
links:
  - objective.md
  - key-results/index.md
  - stages/index.md
  - change-units/index.md
  - gaps/index.md
  - questions/index.md
  - decisions/index.md
  - environment/environment.md
  - e2e/e2e-report.md
  - tokens.md
  - log.md
---

# Run `run_2026_06_28_001` — billing-jakarta-payments-2026q3

> **Progressive disclosure front door.** This page is both the live monitor
> (`STATUS.md`) and the permanent wiki index. Start here, then traverse down.

| | |
|---|---|
| **Run** | `run_2026_06_28_001` — billing-jakarta-payments-2026q3 |
| **Status** | ✅ **COMPLETE** — certification **PASS** |
| **Owner** | min_than_zaw@msi-global.com.sg |
| **Environment** | staged |
| **Started** | 2026-06-28T10:00:00Z |
| **Completed** | 2026-06-28T14:20:00Z (elapsed 4h20m) |
| **Tokens** | 1,423,900 (in 1,131,200 / out 292,700) · ~$22.40 |
| **Repos** | [billing-svc](#repo--billing-svc) (primary), [web-app](#repo--web-app) (consumer) |

**Objective** → [objective.md](objective.md). Migrate `billing-svc` to **Java 21 +
Spring Boot 3.x** and extract the **Payments** capability behind a strangler-fig
seam. Constraints: no breaking change to `/api/v1/**`; max 1 seam per wave; no
data migration.

---

## Key-result scoreboard

Full board → [key-results/index.md](key-results/index.md).

| KR | Result | Status |
|----|--------|--------|
| [KR-01](key-results/kr-01-build-green.md) | Build green on Java 21 / Spring Boot 3 | ✅ **MET** |
| [KR-02](key-results/kr-02-debt-cleared.md) | Zero deprecated API + zero high-sev vuln in scope | ✅ **MET** |
| [KR-03](key-results/kr-03-e2e-equivalence.md) | E2E equivalence on prod traces | ✅ **MET** (38/38) |
| [KR-04](key-results/kr-04-payments-seam.md) | Payments behind seam w/ v1 fallback + O(1) rollback | ✅ **MET** ¹ |

¹ KR-04 met; [GAP-007](gaps/GAP-007.md) remains **open but non-blocking**.

---

## Repo — billing-svc

Primary repo. Java 8 + Spring Boot 1.5 → **Java 21 + Spring Boot 3.3**. Full
pipeline → [stages/index.md](stages/index.md).

```
billing-svc                              COMPLETE  ▓▓▓▓▓▓▓▓ 100%   1,408.9k tok
├─ ✔ INTAKE     toolchain pinned (JDK21, OpenRewrite 2.x)
├─ ✔ INDEX      37 entry points, 1,204 nodes                       48.1k
├─ ✔ RECOVER    9 components, 3 candidate seams, 2 violations      31.4k
├─ ✔ DISCOVER   6 capabilities, 41 business rules                  27.3k
├─ ✔ DEBT       58 debt items, 3 CVEs (Log4j critical)             19.0k
├─ ✔ PLAN       14 ChangeUnits, 4 waves                            91.2k
├─ ✔ EXECUTE    14/14 CUs APPLIED across 4 waves               1,071.5k
└─ ✔ INTEGRATE  Podman topology healthy, E2E 38/38, SLOs met      120.4k
   GAPS: 1 open (GAP-007 payment-provider interface — non-blocking)
```

Per-stage status + tokens → [stages/index.md](stages/index.md) ·
INTAKE/[INDEX](stages/stage-INDEX.md) · [RECOVER](stages/RECOVER.md) ·
[DISCOVER](stages/DISCOVER.md) · [DEBT](stages/DEBT.md) ·
[PLAN](stages/PLAN.md) · [EXECUTE](stages/EXECUTE.md) ·
[INTEGRATE](stages/INTEGRATE.md).

### EXECUTE — change-unit tree (wave 3/4 shown; all 14 APPLIED)

Full register → [change-units/index.md](change-units/index.md).

| CU | Wave | What | Risk | Decision | Tokens | Status |
|----|------|------|------|----------|--------|--------|
| [CU-002](change-units/CU-002.md) | 1 | decouple payments deps | 0.18 | auto | 21.0k | ✅ APPLIED |
| [CU-031](change-units/CU-031.md) | 1 | security-fix Log4j → 2.17 | 0.61 | ⏸ human-approved 13:05 | 34.5k | ✅ APPLIED |
| [CU-014](change-units/CU-014.md) | 2 | javax → jakarta | 0.31 | auto | 47.2k | ✅ APPLIED |
| [CU-018](change-units/CU-018.md) | 2 | upgrade Spring Boot 3.x | 0.38 | auto | 88.1k | ✅ APPLIED |
| [CU-021](change-units/CU-021.md) | 3 | payments seam-extraction | 0.52 | ⏸ human-approved 13:05 | 63.4k | ✅ APPLIED |
| [CU-022](change-units/CU-022.md) | 3 | field → constructor injection | 0.22 | auto | 18.9k | ✅ APPLIED |

> 14 CUs total — table shows the load-bearing ones. The remaining wave-1/2/4
> CUs applied automatically (risk < autoApplyCeiling 0.40).

## Repo — web-app

Consumer repo. Call-site updates via `ts-morph` to follow the `billing-svc`
seam. No seam of its own; tracked under the same run rollup.

---

## Open gaps

Register → [gaps/index.md](gaps/index.md).

| Gap | Summary | Confidence | Status | Risk |
|-----|---------|-----------|--------|------|
| [GAP-007](gaps/GAP-007.md) | External payment-provider API unknown → typed placeholder `POST /charge` | 0.30 | **OPEN (non-blocking)** | +0.15 |

GAP-007 is backed by the Podman mock `mock-payment-provider` and does **not**
block certification. It is a dependency of [CU-021](change-units/CU-021.md).

## Open questions (waiting on you)

**0 open** — the run is not waiting on anything. Register → [questions/index.md](questions/index.md).

| Question | Blocked | Status | Answer |
|----------|---------|--------|--------|
| [Q-001](questions/Q-001-target-language.md) | PLAN | answered | Java 21 (modernize in place) |

> When a question is open it is listed here **first**, the run status shows
> `needs-human`, and the dependent scope waits until you answer
> (`modernize answer <runId> <id> "…"`).

---

## Gate decisions

Every gate decision (auto / pause / block) → [decisions/index.md](decisions/index.md).
Two human-approval pauses occurred, both approved 13:05:

- [d-CU-031-pause.md](decisions/d-CU-031-pause.md) — `requireHumanFor: security-fix`
- [d-CU-021-pause.md](decisions/d-CU-021-pause.md) — `requireHumanFor: seam-extraction`

---

## Discovery & analysis (read-down)

- **Entry points** — 37 non-synthetic surfaces → [entry-points/index.md](entry-points/index.md)
- **Flows** — 41 traced, avg coverage 0.86 → [flows/index.md](flows/index.md)
- **Environment** — Podman topology + monitoring → [environment/environment.md](environment/environment.md)
- **E2E** — equivalence from prod traces → [e2e/e2e-report.md](e2e/e2e-report.md)
- **Tokens** — per-stage rollup + totals → [tokens.md](tokens.md)
- **Log** — chronological state machine → [log.md](log.md)

---

## Machine artifacts

- Live machine status → [`../status.json`](../status.json)
- Resume checkpoint → [`../checkpoint.json`](../checkpoint.json)
- Environment topology → [`../artifacts/environment.yaml`](../artifacts/environment.yaml)
