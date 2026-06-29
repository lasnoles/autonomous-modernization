---
type: change-unit
title: "ChangeUnits — run_2026_06_28_001"
description: "All 14 ChangeUnits planned for billing-svc: id, kind, wave, strategy, risk, gate decision, status, tokens."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
tokens: 432300
tags: [change-units, index, billing-svc]
links: [../index.md, ../log.md, ../stages/PLAN.md, ../stages/EXECUTE.md, ../decisions/index.md, ../gaps/index.md, ../tokens.md]
---

# ChangeUnits — `billing-svc`

The planner (see [PLAN](../stages/PLAN.md)) emitted **14 ChangeUnits** across **4 waves**.
Each CU is the smallest independently-validatable, independently-rollback-able
change. Every CU was scored against the [gates](../decisions/index.md):

> **Decision rule** — `risk ≤ 0.40` → **auto-apply** · `0.40 < risk < 0.80` → **pause** (human) ·
> `risk ≥ 0.80` → **block** · `kind ∈ requireHumanFor` (`seam-extraction`, `security-fix`) → **pause regardless**.

All 14 units reached **APPLIED**. Two were paused for human approval first
(CU-031 security-fix, CU-021 seam-extraction) and approved by the repo owner at
`13:05`. See [decisions](../decisions/index.md) and the [run log](../log.md).

## The register

| CU | Title | Kind | Wave | Strategy | Risk | Decision | Status | Tokens |
|----|-------|------|------|----------|-----:|----------|--------|-------:|
| [CU-001](#cu-001) | Pin & align test deps | dependency-upgrade | 1 | recipe (openrewrite) | 0.12 | auto | APPLIED | 12.4k |
| [CU-002](CU-002.md) | Decouple payments deps | dependency-decouple | 1 | recipe (openrewrite) | 0.18 | auto | APPLIED | 21.0k |
| [CU-003](#cu-003) | Remove dead invoice exporters | dead-code-removal | 1 | codemod | 0.09 | auto | APPLIED | 8.7k |
| [CU-031](CU-031.md) | Log4j CVE-2021-44228 → 2.17 | security-fix | 1 | recipe (openrewrite) | 0.61 | **pause** | APPLIED | 34.5k |
| [CU-014](CU-014.md) | javax.* → jakarta.* | api-migration | 2 | recipe (openrewrite) | 0.31 | auto | APPLIED | 47.2k |
| [CU-015](#cu-015) | Language level Java 8 → 21 | language-level | 2 | recipe (openrewrite) | 0.27 | auto | APPLIED | 29.6k |
| [CU-016](#cu-016) | JUnit 4 → JUnit 5 | test-backfill | 2 | recipe (openrewrite) | 0.19 | auto | APPLIED | 22.1k |
| [CU-018](CU-018.md) | Upgrade Spring Boot 1.x → 3.x | framework-upgrade | 2 | recipe (openrewrite) | 0.38 | auto | APPLIED | 88.1k |
| [CU-019](#cu-019) | Migrate `application.properties` → YAML | config-migration | 2 | codemod | 0.14 | auto | APPLIED | 11.3k |
| [CU-020](#cu-020) | Replace deprecated `WebSecurityConfigurerAdapter` | api-migration | 2 | recipe (openrewrite) | 0.33 | auto | APPLIED | 26.8k |
| [CU-021](CU-021.md) | Extract payments seam (`payments.v2`) | seam-extraction | 3 | seam (interface-facade + flag) | 0.52 | **pause** | APPLIED | 63.4k |
| [CU-022](CU-022.md) | Field-injection → constructor-injection | pattern-refactor | 3 | recipe (openrewrite) | 0.22 | auto | APPLIED | 18.9k |
| [CU-027](#cu-027) | Backfill payments characterization tests | test-backfill | 3 | llm-patch | 0.24 | auto | APPLIED | 31.2k |
| [CU-030](#cu-030) | Tighten actuator exposure | config-migration | 4 | codemod | 0.11 | auto | APPLIED | 6.9k |

**Token total across all CUs: 432.3k** (see the full rollup in [tokens.md](../tokens.md)).

## Detailed ChangeUnits

The six CUs below have their own pages with full rationale, targets,
strategy/recipe, equivalence proof, PRESERVES rules, risk factors, the gate
decision, rollback, and token cost:

- [**CU-002** — decouple payments deps](CU-002.md) · wave 1 · auto
- [**CU-031** — Log4j CVE-2021-44228 → 2.17](CU-031.md) · wave 1 · paused (security-fix) → approved
- [**CU-014** — javax → jakarta](CU-014.md) · wave 2 · auto
- [**CU-018** — Spring Boot 3.x upgrade](CU-018.md) · wave 2 · auto
- [**CU-021** — payments seam extraction](CU-021.md) · wave 3 · paused (seam-extraction) → approved · depends on [GAP-007](../gaps/GAP-007.md)
- [**CU-022** — constructor injection](CU-022.md) · wave 3 · auto

## Summarized ChangeUnits (APPLIED, auto-applied)

The remaining eight units were low-risk, auto-applied (`risk ≤ 0.40`), and
validated green with no human gate. They are summarized here rather than given
their own page.

- <a id="cu-001"></a>**CU-001** — *Pin & align test deps* (wave 1, `dependency-upgrade`). Pins JUnit/Mockito/AssertJ to coherent versions ahead of the framework upgrade. Recipe `org.openrewrite.java.dependencies.UpgradeDependencyVersion`. Risk **0.12** → auto. **APPLIED**, 12.4k tok.
- <a id="cu-003"></a>**CU-003** — *Remove dead invoice exporters* (wave 1, `dead-code-removal`). Drops two unreferenced CSV exporter classes (confirmed zero call sites in the graph). Codemod. Risk **0.09** → auto. **APPLIED**, 8.7k tok.
- <a id="cu-015"></a>**CU-015** — *Language level Java 8 → 21* (wave 2, `language-level`). Recipe `org.openrewrite.java.migrate.UpgradeToJava21`. Behavioral equivalence; existing tests. Risk **0.27** → auto. **APPLIED**, 29.6k tok.
- <a id="cu-016"></a>**CU-016** — *JUnit 4 → JUnit 5* (wave 2, `test-backfill`). Recipe `org.openrewrite.java.testing.junit5.JUnit4to5Migration`. Risk **0.19** → auto. **APPLIED**, 22.1k tok.
- <a id="cu-019"></a>**CU-019** — *`application.properties` → YAML* (wave 2, `config-migration`). Codemod; syntactic equivalence verified by config round-trip. Risk **0.14** → auto. **APPLIED**, 11.3k tok.
- <a id="cu-020"></a>**CU-020** — *Replace deprecated `WebSecurityConfigurerAdapter`* (wave 2, `api-migration`). Recipe `org.openrewrite.java.spring.security6.UpgradeSpringSecurity_6_x`. Risk **0.33** → auto. **APPLIED**, 26.8k tok.
- <a id="cu-027"></a>**CU-027** — *Backfill payments characterization tests* (wave 3, `test-backfill`). Captures current payments behavior as golden tests ahead of the seam cutover; supports [CU-021](CU-021.md) equivalence. llm-patch. Risk **0.24** → auto. **APPLIED**, 31.2k tok.
- <a id="cu-030"></a>**CU-030** — *Tighten actuator exposure* (wave 4, `config-migration`). Restricts `management.endpoints.web.exposure` to `health,info`. Codemod. Risk **0.11** → auto. **APPLIED**, 6.9k tok.

## See also

- [Gate decisions](../decisions/index.md) — the gate math for every CU
- [Gaps register](../gaps/index.md) — [GAP-007](../gaps/GAP-007.md) feeds CU-021's risk
- [PLAN stage](../stages/PLAN.md) · [EXECUTE stage](../stages/EXECUTE.md) · [INTEGRATE stage](../stages/INTEGRATE.md)
- [Key results](../key-results/index.md) · [E2E report](../e2e/e2e-report.md)
