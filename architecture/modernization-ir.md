# Modernization IR (Intermediate Representation)

The Modernization IR is the **planner's output and the compiler's input**.
It is a declarative, tool-agnostic description of *what* should change,
*why*, *in what order*, and *how it will be proven safe* — without
committing to a specific transformation engine. The transformation-compiler
lowers IR into concrete recipes/patches.

## 1. Top-level document

```jsonc
{
  "irVersion": "1.0",
  "runId": "run_2026_06_28_001",
  "scope": {
    "repos": ["billing-svc", "web-app"],
    "objective": "Migrate from Java 8 + Spring Boot 1.x to Java 21 + Spring Boot 3.x; extract payments seam",
    "constraints": ["no behavior change to public API", "max 1 service extracted per wave"]
  },
  "waves": [ /* ordered groups of change-units, see §3 */ ],
  "changeUnits": [ /* see §2 */ ],
  "seams": [ /* see §4 */ ],
  "approach": { /* selected execution playbooks + approach gate, see §4b */ },
  "gates": { /* see §5 */ }
}
```

## 2. ChangeUnit — the atomic planned change

A ChangeUnit is the smallest independently-validatable, independently-
rollback-able change. It maps 1:1 to a `ChangeUnit` node in the graph (L4).

```jsonc
{
  "id": "CU-014",
  "title": "Replace javax.* with jakarta.* in billing-svc",
  "kind": "api-migration",          // see §2.1
  "playbook": "strangler-fig",      // execution playbook governing this CU (see architecture/execution-playbooks.md)
  "repo": "billing-svc",
  "targets": [                      // graph GIDs this CU edits
    "billing-svc::Package::com.acme.billing",
    "billing-svc::Class::com.acme.billing.InvoiceService"
  ],
  "rationale": "Spring Boot 3 requires Jakarta EE 9+ namespaces",
  "businessRefs": ["billing-svc::Capability::Invoicing"],  // what behavior must survive
  "debtRefs": ["billing-svc::DebtItem::deprecated-javax"],
  "strategy": {
    "preferred": "recipe",          // recipe | codemod | llm-patch | generate | manual
                                    // (generate = cross-language synthesis for language-port CUs)
    "recipe": "openrewrite:org.openrewrite.java.migrate.jakarta.JavaxToJakarta",
    "fallback": "llm-patch"
  },
  "equivalence": {                  // how the compiler/validation proves safety
    "level": "behavioral",         // syntactic | behavioral | golden
    "tests": ["existing", "generate-if-missing"],
    "replay": false
  },
  "blastRadius": { "files": 12, "callersAffected": 37, "crossRepo": false },
  "dependsOn": ["CU-002"],          // other CU ids that must land first
  "rollback": "revert-commit"
}
```

### 2.1 ChangeUnit kinds

`api-migration`, `dependency-upgrade`, `language-level`, `framework-upgrade`,
`seam-extraction`, `pattern-refactor`, `dead-code-removal`,
`security-fix`, `test-backfill`, `config-migration`,
`language-port` (cross-language rewrite, e.g. Java→Go — see
`architecture/multi-language-and-porting.md`; uses `strategy: generate` and a
behavioral/golden equivalence proof, never an AST recipe).

> Note: `language-level` means a *version* bump within one language (Java 8→21).
> A change of language is `language-port`, which is a different mechanism.

## 3. Waves — execution ordering

A wave is a set of ChangeUnits that may proceed in parallel; waves run in
sequence. The planner computes waves by topologically sorting the
`dependsOn`/`PRECEDES` graph and grouping by blast radius and risk.

```jsonc
{
  "id": "W1",
  "goal": "Decouple before upgrade",
  "changeUnits": ["CU-001", "CU-002"],
  "parallelizable": true,
  "exitCriteria": "all CUs validated green; no new arch violations"
}
```

## 4. Seam — strangler-fig boundary

A seam is where new and old code are allowed to coexist behind an
indirection (interface, façade, proxy, feature flag, routing rule). Seams
make incremental, reversible cutover possible.

```jsonc
{
  "id": "SEAM-payments",
  "seamKind": "interface-facade",    // interface-facade | http-proxy | event-bridge | feature-flag | branch-by-abstraction
  "boundary": "billing-svc::Component::Payments",
  "introduces": "com.acme.billing.payments.PaymentGateway",
  "routing": { "mechanism": "feature-flag", "key": "payments.v2", "default": "v1" },
  "cutover": "incremental",          // incremental | big-bang
  "rollback": "flip-flag"
}
```

## 4b. Approach — selected execution playbooks (L2)

The planner selects an **execution playbook** per ChangeUnit/wave from the vetted
catalog (`architecture/execution-playbooks.md`) based on discovery facts, and
records the selection + the fact that triggered it. The `approach` block is what
the **post-discovery approach gate** reviews before EXECUTE.

```jsonc
{
  "status": "proposed",              // proposed | approved | redirected
  "selections": [
    { "scope": "billing-svc::Component::Payments", "playbook": "language-port",
      "trigger": "target.language=go for portScope", "rationale": "cross-language rewrite" },
    { "scope": "billing-svc::Component::Reporting", "playbook": "characterization-first",
      "trigger": "changed-code coverage 12% < coverageFloor 40%", "rationale": "pin behavior before edits" },
    { "scope": "default", "playbook": "strangler-fig", "trigger": "default" }
  ],
  "gate": { "policy": "review", "requireApproval": true, "approvedBy": null, "approvedAt": null }
}
```

Each ChangeUnit's chosen playbook is also stamped on the CU (`ChangeUnit.playbook`,
§2). EXECUTE dispatches each CU through its playbook's phases.

## 5. Gates — autonomy controls

```jsonc
{
  "validation": { "requireGreenBuild": true, "minCoverageDelta": 0.0, "requireEquivalence": true },
  "risk":       { "autoApplyCeiling": 0.4, "pauseAboveCeiling": true, "blockAbove": 0.8 },
  "review":     { "requireHumanFor": ["seam-extraction", "security-fix"] },
  "approach":   { "approachGate": "review" }   // auto | review (default) | always — see execution-playbooks.md §4
}
```

## 6. Lifecycle / status

Each ChangeUnit carries a `status` updated by the orchestrator:
`PLANNED → COMPILED → VALIDATED → RISK_SCORED → APPLIED | PAUSED | BLOCKED | ROLLED_BACK | FAILED`.
Status transitions mirror the orchestration state machine
(`orchestration-state-machine.md`).

## 7. Invariants

- Every ChangeUnit references at least one graph target GID.
- Every ChangeUnit with `equivalence.level != syntactic` MUST resolve to a
  validation plan (tests or replay) before it can be APPLIED.
- A ChangeUnit may not be APPLIED before all `dependsOn` are APPLIED.
- Seam-extraction CUs MUST declare a `rollback` that is O(1) (flag flip /
  route change), never a code revert.
