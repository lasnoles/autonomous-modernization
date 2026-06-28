# Edge Types

Exhaustive catalog of edge `type`s. Each: **layer**, direction (`from → to`),
purpose, key `props`. Common envelope in
`architecture/semantic-graph-schema.md` §3. Skills ignore unknown types.

## L0 — Structural (semantic-indexer)

| type | from → to | purpose | key props |
|------|-----------|---------|-----------|
| `CONTAINS` | Repo→Module, Module→Package, Package→Class | containment tree | — |
| `DECLARES` | Class→Method, Class→Field | membership | — |
| `CALLS` | Method→Method | invocation | `callSite`, `dynamic`, `confidence` |
| `EXTENDS` | Class→Class | inheritance | — |
| `IMPLEMENTS` | Class→Class(interface) | interface impl | — |
| `OVERRIDES` | Method→Method | override | — |
| `READS` | Method→Field/Table | data read | `via` (jdbc|jpa|field) |
| `WRITES` | Method→Field/Table | data write | `via` |
| `RETURNS` / `PARAM_OF` | Method→Class | type usage | — |
| `HANDLED_BY` | Endpoint→Method | route to handler | — |
| `ENTERS_AT` | Flow→Endpoint | flow's root entry point | — |
| `FLOWS_THROUGH` | Flow→Method | method reachable in the flow | `order`, `confidence` |
| `TOUCHES` | Flow→Table/ExternalSystem | data/external touched by the flow | `op` (read|write|call) |
| `DEPENDS_ON_LIB` | Module→Dependency | build dependency | `scope`, `direct` |
| `THROWS` | Method→Class(exception) | error contract | — |

## L1 — Architectural (architecture-recovery)

| type | from → to | purpose | key props |
|------|-----------|---------|-----------|
| `BELONGS_TO` | Class→Component, Class→Layer | recovered grouping | `confidence` |
| `DEPENDS_ON` | Component→Component | inter-component coupling | `weight` (call count) |
| `LAYERED_AS` | Component→Layer | layer assignment | — |
| `VIOLATES` | Component/Class→Layer | layering violation | `rule` |
| `CANDIDATE_SEAM` | Seam→Component | seam attaches to boundary | `extractability` |

## L2 — Business (business-discovery)

| type | from → to | purpose | key props |
|------|-----------|---------|-----------|
| `REALIZES_CAPABILITY` | Component/Class→Capability | impl of capability | `confidence` |
| `IMPLEMENTS_RULE` | Method→BusinessRule | rule enforced here | `confidence`, `derivation` |
| `MANIPULATES` | Method→DomainEntity | touches domain entity | `op` (create|read|update|delete) |
| `MAPS_TO` | Class→DomainEntity | impl-to-concept mapping | `confidence` |

## L3 — Debt (technical-debt)

| type | from → to | purpose | key props |
|------|-----------|---------|-----------|
| `HAS_DEBT` | Class/Method/Module→DebtItem | debt attachment | — |
| `DEPENDS_ON_DEPRECATED` | Method→Method/Dependency | uses deprecated API | `replacement` |
| `EXPOSED_TO` | Class/Method→Vulnerability | security exposure | — |
| `IS_HOTSPOT` | Class→Hotspot | churn×complexity | — |

## L4 — Plan (planner)

| type | from → to | purpose | key props |
|------|-----------|---------|-----------|
| `TRANSFORMS` | ChangeUnit→{Class,Method,Package,Component} | edit target | — |
| `PRECEDES` | ChangeUnit→ChangeUnit | ordering dependency | `reason` |
| `GUARDS` | Seam→ChangeUnit | seam protects this change | — |
| `IN_WAVE` | ChangeUnit→Wave | wave membership | — |
| `ADDRESSES` | ChangeUnit→DebtItem/Vulnerability | debt resolved | — |
| `PRESERVES` | ChangeUnit→BusinessRule/Capability | behavior to keep | — |

## L5 — Risk (risk-assessment)

| type | from → to | purpose | key props |
|------|-----------|---------|-----------|
| `HAS_RISK` | ChangeUnit→RiskScore | risk attachment | — |
| `RISK_DRIVEN_BY` | RiskScore→{node} | dominant risk factor source | `factor`, `weight` |

## Integration & gaps (cross-cutting)

Support the integration roles and the placeholder/open-question policy
(`architecture/integration-and-environments.md`,
`architecture/open-question-resolution.md`).

| type | from → to | purpose | key props |
|------|-----------|---------|-----------|
| `DEPENDS_ON_EXTERNAL` | Method/Component→ExternalSystem | runtime dependency | `via`, `confidence` |
| `CONTRACTED_BY` | ExternalSystem→InterfaceContract | the system's interface | — |
| `STUBBED_BY` | InterfaceContract→Mock | mock backing the interface | `fidelity` |
| `PROVISIONS` | Environment→ExternalSystem/Service | devops stood it up (Podman) | `image`, `placeholder` |
| `MONITORS` | Environment→Service | observability wired | `signals` (metrics|logs|traces) |
| `HAS_GAP` | {any node}→Gap | unresolved unknown / placeholder | — |
| `RESOLVED_BY` | Gap→{InterfaceContract,source,decision} | how the gap was answered | `basis`, `confidence` |
| `RAISES_QUESTION` | {Gap,stage,ChangeUnit}→Question | a surfaced, human-answerable question | — |
| `BLOCKS` | Question→{ChangeUnit,stage,repo,run} | scope that waits until the question is answered | — |
| `ANSWERED_BY` | Question→{answer,decision,source} | how the question was resolved | `answeredBy` |
| `SUPERSEDES` | InterfaceContract→InterfaceContract | real contract replaces placeholder | — |
| `VERIFIES` | E2EScenario→{Capability,Endpoint} | E2E coverage of behavior | — |
| `EXERCISES` | E2EScenario→Service/ExternalSystem | systems a scenario drives | — |

## Cross-repo

Any edge with `from.repo != to.repo` MUST set `props.crossRepo = true` and
`props.contract` = the shared contract GID (an Endpoint, Table, or published
event). Used by the orchestrator to lock and re-validate dependents.
