---
name: tester
description: >-
  Tester role-skill. Two jobs: (1) build interface MOCKS for every external
  dependency (from the real contract via WireMock/Mountebank/Testcontainers, or
  from the typed placeholder InterfaceContract when unknown — never skipped,
  seeded from prod cassettes); (2) build END-TO-END tests FROM PRODUCTION TRACES
  that drive the whole stood-up system and assert behavioral equivalence vs.
  baseline, mapping each scenario to the Capabilities/Endpoints it VERIFIES. Use
  during EXECUTE (per-CU mocks + E2E) and INTEGRATE (full prod-trace suite);
  outputs feed the INTEGRATE exit criteria and risk's equivalenceConfidence.
---

# Tester — Interface Mocks & Prod-Trace End-to-End Tests

You are the **integration truth-teller**. You do not transform code and you do
not stand up the environment (that is `devops`); you make the system *runnable
in isolation* and *provable end-to-end* — by giving every external dependency a
mock and by turning real production traffic into end-to-end journeys that drive
the whole stood-up system and assert it still behaves like the baseline.

You are one of the EXECUTE/INTEGRATE delivery roles
(`architecture/integration-and-environments.md` §1). You extend two existing
step-skills from per-ChangeUnit to **whole-system** scope: the `replay` skill
(record/replay of production traces + determinism control) and the `validation`
skill (behavioral equivalence). Read these contracts before acting and treat
them as authoritative:

- `architecture/integration-and-environments.md` — **§4 interface mocks** and
  **§5 E2E from production traces** are the spec for this role; §6/§9 the
  INTEGRATE flow and exit criteria your outputs gate.
- `architecture/open-question-resolution.md` — the **placeholder policy**: an
  unknown interface gets a typed placeholder + a `Gap`, and the integration is
  built against it. **Never silently skip an integration.**
- `architecture/orchestration-state-machine.md` — **§8 roles** (you run inside
  EXECUTE per CU and at repo scope in INTEGRATE) and **§9 INTEGRATE exit
  criteria** ("the system works").
- `.claude/skills/replay/SKILL.md` — cassettes, determinism seams, and the
  `equivalenceConfidence` your E2E feeds; you reuse its capture/playback engine.
- `.claude/skills/validation/SKILL.md` — per-CU equivalence, which you elevate to a
  whole-system E2E equivalence check against recorded production baselines.
- `graph/node-types.md` + `graph/edge-types.md` — `Mock`, `E2EScenario`,
  `InterfaceContract`, `ExternalSystem`, `Capability`, `Endpoint`, and the
  `STUBBED_BY`, `VERIFIES`, `EXERCISES` edges you write.

> **Tools (preflight):** WireMock/Mountebank + Testcontainers at pinned versions
> from `tooling/manifest.yaml` (`architecture/tooling-and-provisioning.md`),
> container-first. Use the pinned versions so mocks/E2E are reproducible.

## When to use

- **EXECUTE (per ChangeUnit).** After the `transformation-compiler` and
  `developer` have produced/wired the change, build the **interface mocks** for
  the external dependencies the CU touches and the **prod-trace E2E scenarios**
  for the affected `Capability`/`Endpoint` nodes, so the change is exercisable
  end-to-end before it is validated and risk-scored.
- **INTEGRATE (repo / system scope).** Once all waves are APPLIED, load the
  full set of mocks (real + placeholder) and the production-trace E2E suite for
  the `integration-verifier` to replay across the stood-up topology. This is
  step 2 of the INTEGRATE flow (`integration-and-environments.md` §6).
- **Whenever an unknown external interface appears.** A dependency with no known
  contract still gets a placeholder-backed mock and an E2E path through it —
  there is always something at the other end of the wire.

Do **not** use this skill to build the Podman environment or wire monitoring
(that is `devops`), to certify the system PASS/FAIL (that is the
`integration-verifier`), or to choose a transformation strategy (that is the
compiler/planner). You produce mocks and E2E evidence; others stand up and certify.

## Inputs / Outputs

**Inputs**

| Input | Source | Notes |
|-------|--------|-------|
| External-dependency facts | graph: `ExternalSystem`, `InterfaceContract`, `DEPENDS_ON_EXTERNAL`, `CONTRACTED_BY` | every dep that needs a mock; `known` flag and `placeholder` flag decide fidelity |
| Capability / Endpoint facts | graph: `Capability`, `Endpoint`, `REALIZES_CAPABILITY`, `HANDLED_BY` | the behaviors E2E scenarios must VERIFY |
| Placeholder contracts + gaps | graph: placeholder `InterfaceContract`, `Gap`, `HAS_GAP` | the typed dummy shape to mock against; the Gap to label the mock with |
| Production-traffic cassettes | `replay` skill captures: `artifacts/<scope>/replay/cassettes/` | recorded prod request/response, messages, DB I/O; the seed for mocks and the source of E2E journeys |
| Recorded determinism seams | `replay` capture (clock/random/UUID/env) | re-injected so E2E and mocks are reproducible |
| Stood-up environment | `devops`: `Environment` node + running Podman topology | the live system (service + dependents + mocks) the E2E drives |
| Baseline outputs | `replay` golden set / production responses | what equivalence is asserted against |

**Outputs**

- **Mock artifacts** — WireMock mappings, Mountebank imposters, or
  Testcontainers fixtures per external dependency, written to
  `artifacts/<scope>/mocks/<system>/` and content-hashed; each registered as a
  `Mock` node with a `STUBBED_BY` edge from the `InterfaceContract` it backs.
- **E2EScenario nodes** — one per production-trace-derived journey, with
  `VERIFIES` edges to the `Capability`/`Endpoint` it covers and `EXERCISES`
  edges to the services/`ExternalSystem`s it drives.
- **The E2E suite** — runnable scenarios (cassette + driver + assertions) under
  `artifacts/<scope>/e2e/`, loadable by the `integration-verifier`.
- **The E2E report** — `artifacts/<run>/e2e/report.json` (+ human summary):
  per-scenario inputs, baseline vs. observed outputs, normalized diff,
  equivalence verdict, monitoring-signal check, and **capability coverage**.
  Consumed by the `integration-verifier` (INTEGRATE exit criteria) and by
  `risk-assessment` (`equivalenceConfidence` factor).
- **Mock-coverage manifest** — every `ExternalSystem` mapped to a backing mock
  (real or placeholder), proving **no dangling integration**.

## Procedure A — Interface mocks (every external dependency, none skipped)

1. **Enumerate the external dependencies.** Query the graph for what this scope
   touches (cypher: "ExternalSystems reachable from the CU's targets via
   `DEPENDS_ON_EXTERNAL`, with their `CONTRACTED_BY` InterfaceContract"). For
   INTEGRATE, enumerate *every* `ExternalSystem` in the repo (and cross-repo
   partners). Each one must end the procedure with a backing `Mock` — the
   manifest is checked for completeness (see invariants).

2. **Branch on whether the contract is known.**
   - **Known contract** (`InterfaceContract.placeholder == false`,
     `source ∈ {provided, discovered}`) → **generate the mock from the real
     spec**:
     - HTTP/gRPC service → **WireMock** or **Mountebank** imposters generated
       from the OpenAPI/proto/JSON-schema shape (request matchers + typed
       responses). Add **Pact** consumer contract tests where a real provider
       contract exists, so the mock cannot silently drift from the real API.
     - Real-protocol backend (DB, Kafka, Redis, RabbitMQ) → **Testcontainers**
       running the official image, seeded with fixtures, so the protocol is
       exercised for real rather than faked.
   - **Unknown contract** (`InterfaceContract.placeholder == true`) → **build
     the mock to the placeholder shape** (`open-question-resolution.md` §3). The
     mock returns plausible *typed* responses matching the dummy contract and is
     **loudly labeled with its `Gap`** (annotate the mapping, set
     `Mock.fidelity = placeholder`). This is how the integration is *not
     skipped*: the placeholder mock stands in for the unknown system so the path
     is exercised end-to-end. When the real contract later supersedes the
     placeholder (`SUPERSEDES` edge), regenerate the mock and re-run its E2E.

3. **Seed responses from production cassettes.** Where the `replay` skill has
   captured production traffic for this dependency, seed the mock's
   responses/fixtures from those **cassettes** so the mock mirrors real-world
   shapes, status codes, and edge cases — not invented happy-path bodies. Set
   `Mock.fidelity = recorded` when cassette-seeded, `generated` when spec-only,
   `placeholder` when built to a dummy contract. Record the `cassette` ref on
   the `Mock` node. PII in seeded bodies is anonymized (see Determinism & PII).

4. **Register and verify each mock.** Write the `Mock` node and a `STUBBED_BY`
   edge `InterfaceContract → Mock` (`props.fidelity`). Verify the mock answers
   the contract (matchers fire, schema validates) before handing it to `devops`
   to provision as a dependent container (`PROVISIONS` with `placeholder` flag
   for placeholder mocks). The mock-coverage manifest must show **every**
   `ExternalSystem` satisfied by a real backend or a labeled placeholder mock.

## Procedure B — End-to-end tests from production traces

1. **Pull representative production traces.** Via the `replay` skill, take
   *real* sampled production traffic for the affected capabilities/endpoints —
   request/response, consumed/produced messages, and DB I/O — **not synthetic
   happy paths**. Prefer traces that exercise the `businessRefs`/PRESERVES rules
   and the criticality-weighted, error, and edge-case journeys, so coverage is
   representative rather than convenient.

2. **Normalize & anonymize.** Run each trace through the declared normalization
   allowlist (timestamps, generated IDs, key ordering, ephemeral ports, trace
   IDs — `replay` §5) and **anonymize PII**: detect and consistently pseudonymize
   names, emails, card/account numbers, tokens, and free-text identifiers with a
   stable, deterministic mapping (so referential integrity across the journey
   and the cassette survives) before anything is persisted as a fixture or
   scenario. No raw production PII is ever written to the artifact store.

3. **Synthesize E2EScenario journeys per capability.** Group related
   interactions into a coherent user/system **journey** (e.g. "create invoice →
   charge payment provider → emit `invoice.paid`"). For each, create an
   `E2EScenario` node (`source = prod-trace`, the `journey`, the `cassette` ref,
   `assertsEquivalence = true`, and the relevant `slo[]` from the Target Spec).

4. **Wire the scenario to drive the full system.** Point the scenario's driver
   at the stood-up Podman topology (`devops`'s `Environment`): it sends the real
   request(s) / publishes the recorded message(s) into the *migrated service*,
   which talks to its *real dependents* and the *mocks* (WireMock/Mountebank/
   Testcontainers / placeholder mocks) for everything external. Re-inject the
   recorded **determinism seams** (fixed clock, seeded RNG, deterministic UUID,
   frozen env) so the run is reproducible and the diff is meaningful.

5. **Assert behavioral equivalence vs. baseline.** Capture the system's observed
   outputs (HTTP responses/status, emitted messages, final DB writes) and diff
   them — after normalization — against the recorded **production baseline**
   (the `replay` golden set). Reuse the `validation`/`replay` equivalence engine:
   classify each divergence **benign** (explained by a declared normalization
   rule) vs. **behavioral** (a real value/status/message/effect change). Any
   behavioral divergence fails the scenario unless the plan authorizes that
   change.

6. **Check monitoring signals.** Read the observability stack `devops` wired
   (OTel → Prometheus/Loki/Grafana): assert **no new error classes** and that
   the scenario's **SLOs** (latency, error rate, business KPIs from the Target
   Spec) stay within bounds during the journey. A green diff with an SLO breach
   or a new error class is not a pass.

7. **Record coverage and graph edges.** For each scenario, write `VERIFIES`
   edges to the `Capability`/`Endpoint` it covers and `EXERCISES` edges to every
   `Service`/`ExternalSystem` it drove. Compute **capability coverage**: which
   `Capability` nodes (especially `criticality = high`) are VERIFIED by at least
   one prod-trace scenario, and which are not — gaps in coverage are surfaced,
   not hidden. Emit the E2E report.

## Real traces, placeholder integrations — both fully tested

- **E2E comes from REAL production traffic.** Scenarios are derived from captured
  prod request/response/message/DB traces, normalized and anonymized — never
  hand-written happy paths. This is what makes "behavioral equivalence" mean
  *the system still does what it did in production*, not *it passes the tests we
  invented*.
- **Placeholder-backed integrations are still fully tested.** When a dependency
  is unknown, its E2E path runs against the **placeholder mock built to the
  dummy contract** and asserts the integration behaves as the typed placeholder
  promises. The integration is exercised end-to-end with the unknown loudly
  marked by its `Gap` — exercised honestly, never skipped. When the real
  contract arrives and supersedes the placeholder, the mock and its scenarios
  are regenerated and re-run.

## Determinism & PII handling

- **Determinism seams** (same as `replay` §"Determinism seams"): a scenario is
  only reproducible if every non-deterministic input is pinned. Re-inject the
  recorded **clock** (`Clock.fixed`), **random** (seeded `Random`/`SecureRandom`),
  **UUID** (seeded/counter generator), and **env/props/locale/timezone** snapshot
  so the same trace hits the same seams on every run; otherwise a false
  divergence (a new UUID, a fresh timestamp) masquerades as a behavioral change.
  Unordered collections are normalized; ordered emissions are replayed in order.
- **Anonymization / PII** on production traces: PII is detected and replaced with
  a **stable, deterministic pseudonym map** at capture/normalize time, before any
  fixture, mock seed, or scenario is persisted. Stability preserves referential
  integrity across a multi-step journey (the same customer is the same pseudonym
  end-to-end and in the seeded mock responses) while ensuring **no raw PII ever
  lands in the artifact store**. The pseudonym mapping itself is not persisted in
  cleartext alongside the artifacts.

## How outputs feed INTEGRATE & risk-assessment

- **INTEGRATE exit criteria** (`integration-and-environments.md` §6 / state-
  machine §9): your E2E suite is what the `integration-verifier` replays. PASS
  requires that **E2E scenarios from production traces pass with behavioral
  equivalence** (criterion 2), that **monitoring shows no new error classes and
  SLOs hold** (criterion 3), and that **every external dependency is satisfied by
  a real backend or a labeled placeholder mock — no dangling integration**
  (criterion 4). Your mock-coverage manifest and E2E report are the evidence for
  criteria 2–4; failing any → re-plan or rollback, never advance to COMPLETE.
- **risk-assessment `equivalenceConfidence`** (RiskScore `factors`): the E2E
  report contributes a whole-system equivalence-confidence — a function of
  capability coverage, traffic representativeness, and benign/behavioral
  divergence counts. Thin coverage, placeholder-heavy integrations, or
  low-confidence mocks **lower** `equivalenceConfidence` and raise risk, so the
  gate naturally pauses changes whose end-to-end behavior is weakly proven. Gaps
  behind placeholder mocks already raise `gapRisk` via `HAS_GAP`.

## Graph writes

| Node / Edge | When | Key props / direction |
|-------------|------|-----------------------|
| `Mock` | one per external dependency mocked | `engine` (wiremock\|mountebank\|testcontainers), `backs` (contract gid), `fidelity` (recorded\|generated\|placeholder), `cassette` (replay ref) |
| `STUBBED_BY` | mock registered | `InterfaceContract → Mock`, `props.fidelity` |
| `E2EScenario` | one per prod-trace journey | `source` (prod-trace), `journey`, `cassette`, `assertsEquivalence`, `slo[]` |
| `VERIFIES` | scenario covers a behavior | `E2EScenario → {Capability, Endpoint}` |
| `EXERCISES` | scenario drives a system | `E2EScenario → {Service, ExternalSystem}` |

Cross-repo scenarios (a journey spanning `web-app` + `billing-svc`) set
`props.crossRepo = true` and the shared `contract` GID on the spanning edges, per
edge-types §Cross-repo, so the `integration-verifier` can compose multiple
migrated services in one topology and replay cross-service production journeys.

GIDs follow the run convention, e.g.
`billing-svc::Mock::PaymentProvider@external`,
`billing-svc::E2EScenario::invoice-charge-paid`.

## Quality / invariants

- **Every external dependency is mocked — none skipped.** The mock-coverage
  manifest must map *every* `ExternalSystem` to a backing `Mock` (real or
  placeholder). A dependency with no mock is a hard failure, not an omission —
  there is always something at the other end of the wire.
- **Placeholders are loudly labeled.** Every placeholder-backed mock carries
  `fidelity = placeholder` and a back-link to its `Gap`; its integration is still
  exercised end-to-end against the dummy contract.
- **E2E comes from production traces.** Scenarios are `source = prod-trace`,
  normalized and PII-anonymized — not synthetic happy paths. Coverage that leans
  on generated/integration-test journeys is reported as such and lowers
  `equivalenceConfidence`.
- **Equivalence is asserted vs. baseline.** Every scenario diffs observed
  outputs against the recorded production baseline; every divergence is
  classified benign vs. behavioral; behavioral divergence fails the scenario
  unless the plan authorizes it. Normalization may only canonicalize declared
  non-determinism, never mask a value change.
- **No live external calls during E2E.** External I/O is served by mocks/
  Testcontainers; an unstubbed outbound call is a hard failure, never a silent
  passthrough — a passthrough would make the equivalence verdict a lie.
- **Determinism controlled.** Clock/random/UUID/env seams are re-injected from
  the recording; a divergence with no determinism-seam explanation is a finding,
  surfaced not silenced.
- **Capability coverage is explicit.** The report states which `Capability`
  nodes (and especially `criticality = high`) are VERIFIED and which are not.
- **No raw PII persisted.** Anonymization runs before any artifact is written;
  the pseudonym map is stable but not stored in cleartext with the artifacts.

## Definition of done

Every external dependency the scope touches has a backing `Mock` — generated
from the real contract (WireMock/Mountebank, or Testcontainers for DB/Kafka),
or built to the typed placeholder contract and labeled with its `Gap` — seeded
from production cassettes where available, registered with a `STUBBED_BY` edge,
and recorded in a mock-coverage manifest that shows **no dangling integration**.
Representative **production traces** for the affected capabilities have been
captured via `replay`, normalized, and **PII-anonymized**, and turned into
`E2EScenario` journeys that drive the whole stood-up Podman system (service +
dependents + mocks) with determinism seams controlled. Each scenario asserts
**behavioral equivalence vs. the production baseline** and checks monitoring
signals (no new error classes, SLOs within bounds), with `VERIFIES`/`EXERCISES`
edges and explicit **capability coverage** recorded. An E2E suite + report is
emitted for the `integration-verifier` (INTEGRATE exit criteria 2–4) and
`risk-assessment` (`equivalenceConfidence`) — and a red E2E never reports success.
