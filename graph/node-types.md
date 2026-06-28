# Node Types

Exhaustive catalog of node `kind`s. Each entry: **layer**, purpose, required
`props` (beyond the common envelope in `architecture/semantic-graph-schema.md`
§2), and an example. Skills ignore unknown kinds (forward-compat).

## L0 — Structural (semantic-indexer)

### Repo
- **Purpose:** a source repository.
- **props:** `vcsUrl`, `defaultBranch`, `buildSystem` (maven|gradle|...), `lang`.

### Module
- **Purpose:** build module / Maven artifact / Gradle subproject.
- **props:** `coordinates` (group:artifact:version), `packaging`.

### Package
- **Purpose:** Java package / namespace.
- **props:** `classCount`.

### Class
- **Purpose:** class, interface, enum, record, annotation.
- **props:** `classKind` (class|interface|enum|record|annotation), `modifiers[]`,
  `isPublicApi` (bool), `annotations[]`.

### Method
- **Purpose:** method/constructor.
- **props:** `signature`, `returnType`, `params[]`, `modifiers[]`,
  `cyclomatic`, `annotations[]`, `isEntryPoint` (bool).

### Field
- **Purpose:** field / property.
- **props:** `type`, `modifiers[]`, `initializer` (bool).

### Endpoint  *(the entry-point node — systems have many, not one)*
- **Purpose:** any externally reachable entry surface. See
  `architecture/entry-point-discovery.md` for the full taxonomy — discovery
  enumerates **all** of them.
- **props:** `protocol` (http|graphql|grpc|soap|amqp|kafka|jms|cron|cli|file|lib),
  `entryClass` (http|messaging|scheduled|batch|cli|webhook|startup|file|lib|main),
  `verb`, `path`/`topic`/`schedule`, `mediaTypes[]`, `synthetic` (bool — e.g. test
  fixtures, excluded from production flows).

### Flow
- **Purpose:** an execution slice rooted at one entry point — the transitive
  call graph from the entry through methods into the data and external systems it
  touches. One Endpoint may have several Flows; a Flow may span repos.
- **props:** `rootEntryPoint` (Endpoint gid), `kind` (request|message|scheduled|
  batch|cli|startup|lib), `methodSet` (count), `crossRepo` (bool),
  `coverage` (0–1), `synthetic` (bool).

### Table
- **Purpose:** persistent store object (RDBMS table, collection).
- **props:** `schema`, `columns[]`, `pk[]`.

### Dependency
- **Purpose:** external library/artifact the code depends on.
- **props:** `coordinates`, `version`, `scope`, `direct` (bool).

## L1 — Architectural (architecture-recovery)

### Component
- **Purpose:** recovered logical component/service-candidate (cluster of classes).
- **props:** `cohesion`, `coupling`, `responsibility` (short text), `size`.

### Layer
- **Purpose:** architectural layer (web|service|domain|persistence|integration).
- **props:** `name`, `expectedDependencies[]`.

### Seam
- **Purpose:** candidate strangler-fig boundary (also refined in L4).
- **props:** `seamKind` (interface-facade|http-proxy|event-bridge|feature-flag|branch-by-abstraction),
  `extractability` (0–1), `boundaryClasses[]`.

## L2 — Business (business-discovery)

### Capability
- **Purpose:** business capability ("Invoicing", "Payments").
- **props:** `description`, `criticality` (low|med|high), `owners[]`.

### BusinessRule
- **Purpose:** a discrete rule/invariant ("invoice cannot be negative").
- **props:** `statement`, `derivation` (test|code|comment|llm), `confidence`.

### DomainEntity
- **Purpose:** domain concept distinct from its class impl.
- **props:** `name`, `aliases[]`, `lifecycleStates[]`.

## L3 — Debt (technical-debt)

### DebtItem
- **Purpose:** a unit of technical debt.
- **props:** `category` (deprecated-api|code-smell|complexity|outdated-dep|
  test-gap|architecture-violation|config), `severity` (0–1), `effort` (S|M|L|XL),
  `interestRate` (how fast it worsens), `evidence`.

### Vulnerability
- **Purpose:** security issue (CVE / insecure pattern).
- **props:** `cve`, `cvss`, `component`, `fixedIn`.

### Hotspot
- **Purpose:** high-churn × high-complexity region (change risk concentrator).
- **props:** `churn`, `complexity`, `score`.

## L4 — Plan (planner)

### ChangeUnit
- **Purpose:** atomic planned change (see `architecture/modernization-ir.md` §2).
- **props:** mirrors the IR ChangeUnit (`kind`, `strategy`, `equivalence`,
  `blastRadius`, `status`).

### Wave
- **Purpose:** ordered execution group.
- **props:** `order`, `parallelizable`, `exitCriteria`.

## L5 — Risk (risk-assessment)

### RiskScore
- **Purpose:** risk of applying a ChangeUnit.
- **props:** `score` (0–1), `factors{}` (blastRadius, testCoverage,
  businessCriticality, equivalenceConfidence, crossRepo, **gapRisk**), `decision`
  (auto|pause|block), `explanation`.

## Integration & gaps (cross-cutting)

These support the integration roles and the open-question/placeholder policy
(`architecture/integration-and-environments.md`,
`architecture/open-question-resolution.md`). Discovered during INDEX; refined by
the devops/tester/integration-verifier roles.

### ExternalSystem
- **Purpose:** a system the code depends on (DB, downstream service, broker,
  cache, third-party API, filesystem).
- **props:** `sysKind` (db|http-service|queue|cache|thirdparty|filesystem),
  `name`, `known` (bool), `owner`, `provisioning` (image ref or `mock`).

### InterfaceContract
- **Purpose:** the interface to an ExternalSystem (API/schema/event shape).
- **props:** `protocol` (http|amqp|kafka|jdbc|grpc), `shape` (typed contract),
  `placeholder` (bool), `confidence` (0–1), `source`
  (provided|discovered|inferred), `gap` (Gap gid if placeholder).

### Mock
- **Purpose:** a test double standing in for an ExternalSystem/InterfaceContract.
- **props:** `engine` (wiremock|mountebank|testcontainers), `backs` (contract gid),
  `fidelity` (recorded|generated|placeholder), `cassette` (replay trace ref).

### Environment
- **Purpose:** a runnable, reproducible system topology (Podman).
- **props:** `runtime` (podman), `services[]` (image+role), `network`,
  `monitoring[]` (otel|prometheus|grafana|loki), `healthy` (bool).

### Gap
- **Purpose:** a tracked open question / unknown dependency / assumption.
- **props:** `category` (unknown-interface|missing-dep|ambiguous-intent|
  undocumented-behavior), `subject`, `status` (open|placeholdered|assumed|resolved),
  `assumption`, `confidence` (0–1), `blocking` (bool), `whatWouldResolveIt`,
  `raisesRiskBy` (0–1). See `architecture/open-question-resolution.md` §4.

### Question
- **Purpose:** an open question the agent surfaces for a human and **waits** on
  (a Gap the system chose not to auto-resolve). Shown in the status file + OKF
  wiki; answering it unblocks the dependent work. See
  `architecture/open-question-resolution.md` §6.
- **props:** `prompt` (the question), `context`, `options[]` (suggested answers,
  first = recommended), `default` (what it would assume if forced), `blocking`
  (bool), `status` (open|answered|withdrawn), `answer`, `answeredBy`, `answeredAt`,
  `raisedBy` (stage/skill), `askedAt`.

### E2EScenario
- **Purpose:** an end-to-end test journey built from a production trace.
- **props:** `source` (prod-trace|integration-test|generated), `journey`,
  `cassette` (replay ref), `assertsEquivalence` (bool), `slo[]`.
