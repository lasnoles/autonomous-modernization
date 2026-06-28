---
type: log
title: "Run log — run_2026_06_28_001"
description: "Append-only chronological state-transition + token log. Reading top-to-bottom replays the state machine."
status: COMPLETE
runId: run_2026_06_28_001
gid: run::run_2026_06_28_001::log
tags: [log, state-machine, audit-trail]
links:
  - index.md
  - stages/index.md
  - change-units/index.md
  - decisions/index.md
  - tokens.md
---

# Run Log — `run_2026_06_28_001`

> Append-only fold of the event stream. Every line is one transition: timestamp
> (UTC) · scope · `from → to` · guard/result · tokens · link to the concept page.
> Never rewrite — this is the audit trail. Token rollup → [tokens.md](tokens.md).

## INTAKE → PLAN (billing-svc)

- `10:00:02` **INTAKE** billing-svc `∅ → READY` · ok · toolchain pinned (JDK21, OpenRewrite 2.x) · 0 tok · [stages/index.md](stages/index.md)
- `10:00:05` **INTAKE** web-app `∅ → READY` · ok · consumer repo, ts-morph pinned · 0 tok · [stages/index.md](stages/index.md)
- `10:08:31` **INDEX** `READY → DONE` · ok · 37 entry points, 1,204 nodes (+14 synthetic excluded) · 48.1k tok · [stages/stage-INDEX.md](stages/stage-INDEX.md)
- `10:14:47` **RECOVER** `READY → DONE` · ok · 9 components, 3 candidate seams (Payments 0.78, Dunning 0.55, Invoicing 0.40), 2 layering violations · 31.4k tok · [stages/RECOVER.md](stages/RECOVER.md)
- `10:19:58` **DISCOVER** `READY → DONE` · ok · 6 capabilities, 41 business rules · 27.3k tok · [stages/DISCOVER.md](stages/DISCOVER.md)
- `10:23:40` **DEBT** `READY → DONE` · ok · 58 debt items, 3 CVEs incl. Log4j CVE-2021-44228 (critical), top hotspot InvoiceService · 19.0k tok · [stages/DEBT.md](stages/DEBT.md)
- `10:21:00` **ASK** `Q-001` · target language ambiguous (Java 21 vs port to Go) · **PLAN paused, run status → needs-human** · blocks PLAN · [questions/Q-001-target-language.md](questions/Q-001-target-language.md)
- `10:24:30` **ANSWER** `Q-001` · "Java 21 (modernize in place)" by min_than · PLAN resumes · [questions/Q-001-target-language.md](questions/Q-001-target-language.md)
- `10:25:11` **PLAN** `READY → DONE` · ok · 14 ChangeUnits, 4 waves (modernization, not language-port) · gates autoApplyCeiling 0.40 / blockAbove 0.80 / requireHumanFor [seam-extraction, security-fix] · 91.2k tok · [stages/PLAN.md](stages/PLAN.md)

## EXECUTE — wave 1

- `10:42:18` **CU-002** decouple-payments-deps `VALIDATED → RISK_SCORED → APPLIED` · risk 0.18 (auto < 0.40) · 21.0k tok · [change-units/CU-002.md](change-units/CU-002.md)
- `10:51:33` **CU-031** security-fix Log4j→2.17 `RISK_SCORED → PAUSED` · guard `requireHumanFor:security-fix` (risk 0.61) · [decisions/d-CU-031-pause.md](decisions/d-CU-031-pause.md)
- `13:05:00` **CU-031** security-fix Log4j→2.17 `PAUSED → human-approved` · approver min_than_zaw@msi-global.com.sg · [decisions/d-CU-031-pause.md](decisions/d-CU-031-pause.md)
- `13:12:44` **CU-031** security-fix Log4j→2.17 `human-approved → APPLIED` · risk 0.61 · 34.5k tok · [change-units/CU-031.md](change-units/CU-031.md)

## EXECUTE — wave 2

- `11:02:55` **CU-014** javax→jakarta `VALIDATED → RISK_SCORED → APPLIED` · risk 0.31 (auto < 0.40) · 47.2k tok · [change-units/CU-014.md](change-units/CU-014.md)
- `11:24:09` **CU-018** UpgradeSpringBoot_3_x `VALIDATED → RISK_SCORED → APPLIED` · risk 0.38 (auto < 0.40) · 88.1k tok · [change-units/CU-018.md](change-units/CU-018.md)

## EXECUTE — wave 3

- `11:39:10` **CU-021** payments seam-extraction `RISK_SCORED → PAUSED` · guard `requireHumanFor:seam-extraction` (risk 0.52) · depends GAP-007 · [decisions/d-CU-021-pause.md](decisions/d-CU-021-pause.md)
- `11:55:02` **CU-022** field→constructor injection `VALIDATED → RISK_SCORED → APPLIED` · risk 0.22 (auto < 0.40) · 18.9k tok · [change-units/CU-022.md](change-units/CU-022.md)
- `13:05:00` **CU-021** payments seam-extraction `PAUSED → human-approved` · approver min_than_zaw@msi-global.com.sg · interface-facade + feature-flag `payments.v2` default-off, O(1) rollback flip-flag · [decisions/d-CU-021-pause.md](decisions/d-CU-021-pause.md)
- `13:50:00` **CU-021** payments seam-extraction `human-approved → APPLIED` · risk 0.52 · 63.4k tok · [change-units/CU-021.md](change-units/CU-021.md)

## EXECUTE — wave 4

- `13:58:21` **EXECUTE** wave 4 `RUNNING → DONE` · remaining CUs APPLIED (risk < 0.40, auto) · all 14/14 CUs APPLIED · EXECUTE total 1,071.5k tok · [stages/EXECUTE.md](stages/EXECUTE.md)

## INTEGRATE → COMPLETE

- `14:02:10` **INTEGRATE** `READY → RUNNING` · Podman topology up: billing-svc + postgres:16 + rabbitmq:3 + mock-payment-provider(placeholder) + otel-collector + prometheus + grafana + loki · [environment/environment.md](environment/environment.md)
- `14:11:36` **INTEGRATE** E2E from prod traces `RUNNING → ok` · 38/38 pass equivalence · p95 182ms (<250), error 0.00% (<0.5%) · [e2e/e2e-report.md](e2e/e2e-report.md)
- `14:18:52` **INTEGRATE** `RUNNING → DONE` · all services healthy, certification **PASS** · 120.4k tok · [stages/INTEGRATE.md](stages/INTEGRATE.md)
- `14:20:00` **RUN** `running → COMPLETE` · KR1–KR4 MET · GAP-007 open (non-blocking) · total 1,423,900 tok (~$22.40) · [index.md](index.md)
