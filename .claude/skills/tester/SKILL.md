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

You are the **integration truth-teller**: give every external dependency a mock
and turn real production traffic into E2E journeys that drive the whole stood-up
system and assert it still behaves like the baseline. You do NOT transform code,
stand up the environment (`devops`), certify PASS/FAIL (`integration-verifier`),
or choose a strategy (compiler/planner). You extend two step-skills from per-CU
to **whole-system** scope: `replay` (record/replay + determinism) and
`validation` (behavioral equivalence).

Authoritative contracts: `architecture/integration-and-environments.md` §4
(interface mocks), §5 (E2E from prod traces), §6/§9 (INTEGRATE flow + exit
criteria you gate) · `architecture/open-question-resolution.md` §3 (placeholder
policy: unknown interface → typed placeholder + `Gap`, built-against, **never
silently skipped**) · `architecture/orchestration-state-machine.md` §8 (roles:
EXECUTE per-CU, INTEGRATE repo-scope), §9 (exit criteria) ·
`.claude/skills/replay/SKILL.md` (cassettes, determinism seams,
`equivalenceConfidence`; reuse its capture/playback engine) ·
`.claude/skills/validation/SKILL.md` (per-CU equivalence you elevate to
whole-system) · `graph/node-types.md` + `graph/edge-types.md` (`Mock`,
`E2EScenario`, `InterfaceContract`, `ExternalSystem`, `Capability`, `Endpoint`;
`STUBBED_BY`, `VERIFIES`, `EXERCISES`).

> **Tools (preflight):** WireMock/Mountebank + Testcontainers at pinned versions
> from `tooling/manifest.yaml` (`architecture/tooling-and-provisioning.md`),
> container-first — pinned so mocks/E2E reproduce.

## When to use

- **EXECUTE (per CU).** After `transformation-compiler` + `developer` wire the
  change, build mocks for the external deps the CU touches and prod-trace E2E
  for the affected `Capability`/`Endpoint` nodes, so it is exercisable
  end-to-end before validation + risk-scoring.
- **INTEGRATE (repo/system scope).** Once all waves are APPLIED, load the full
  mock set (real + placeholder) + the prod-trace E2E suite for the
  `integration-verifier` to replay across the topology (INTEGRATE step 2, §6).
- **Any unknown external interface** still gets a placeholder-backed mock + an
  E2E path through it — there is always something at the other end of the wire.

## Inputs / Outputs

**In**

| Input | Source | Notes |
|-------|--------|-------|
| External-dependency facts | graph: `ExternalSystem`, `InterfaceContract`, `DEPENDS_ON_EXTERNAL`, `CONTRACTED_BY` | every dep needing a mock; `known`/`placeholder` flags decide fidelity |
| Capability / Endpoint facts | graph: `Capability`, `Endpoint`, `REALIZES_CAPABILITY`, `HANDLED_BY` | behaviors E2E must VERIFY |
| Placeholder contracts + gaps | graph: placeholder `InterfaceContract`, `Gap`, `HAS_GAP` | typed dummy shape to mock against; the Gap to label with |
| Production cassettes | `replay`: `artifacts/<scope>/replay/cassettes/` | recorded prod req/resp, messages, DB I/O; seed for mocks + source of E2E journeys |
| Determinism seams | `replay` capture (clock/random/UUID/env) | re-injected for reproducibility |
| Stood-up environment | `devops`: `Environment` node + running Podman topology | the live system the E2E drives |
| Baseline outputs | `replay` golden set / prod responses | what equivalence is asserted against |

**Out**
- **Mock artifacts** — WireMock mappings / Mountebank imposters / Testcontainers
  fixtures per dep → `artifacts/<scope>/mocks/<system>/`, content-hashed; each a
  `Mock` node with `STUBBED_BY` from the `InterfaceContract` it backs.
- **E2EScenario nodes** — one per prod-trace journey, `VERIFIES` →
  `Capability`/`Endpoint`, `EXERCISES` → services/`ExternalSystem`s driven.
- **E2E suite** — runnable scenarios (cassette + driver + assertions) under
  `artifacts/<scope>/e2e/`, loadable by `integration-verifier`.
- **E2E report** — `artifacts/<run>/e2e/report.json` (+ summary): per-scenario
  inputs, baseline vs. observed, normalized diff, equivalence verdict,
  monitoring-signal check, **capability coverage**. Consumed by
  `integration-verifier` (exit criteria) + `risk-assessment`
  (`equivalenceConfidence`).
- **Mock-coverage manifest** — every `ExternalSystem` → a backing mock (real or
  placeholder), proving **no dangling integration**.

## Procedure A — Interface mocks (every external dependency, none skipped)

```
1. ENUMERATE deps. EXECUTE: graph query "ExternalSystems reachable from CU.targets
   via DEPENDS_ON_EXTERNAL, with their CONTRACTED_BY InterfaceContract".
   INTEGRATE: every ExternalSystem in repo (+ cross-repo partners).
   Each MUST exit with a backing Mock (manifest checked for completeness).

2. BRANCH on contract knowledge:
   • KNOWN (placeholder==false, source ∈ {provided,discovered}) → mock from real spec:
       - HTTP/gRPC → WireMock/Mountebank imposters from OpenAPI/proto/JSON-schema
         (matchers + typed responses). Add Pact consumer tests where a real
         provider contract exists, so the mock cannot silently drift.
       - Real-protocol backend (DB/Kafka/Redis/RabbitMQ) → Testcontainers on the
         official image, seeded with fixtures — protocol exercised for real.
   • UNKNOWN (placeholder==true) → mock to the placeholder shape (open-question §3):
       returns plausible TYPED responses to the dummy contract, LOUDLY labeled
       with its Gap (annotate mapping, Mock.fidelity=placeholder). This is how the
       integration is NOT skipped — the placeholder stands in so the path runs E2E.
       On SUPERSEDES (real contract arrives), regenerate the mock + re-run its E2E.

3. SEED from cassettes. Where replay captured prod traffic for the dep, seed
   responses/fixtures from cassettes (real shapes/status/edge cases, not invented
   happy paths). Set Mock.fidelity = recorded (cassette) | generated (spec-only)
   | placeholder (dummy contract); record `cassette` ref. Anonymize PII (§D&PII).

4. REGISTER + VERIFY. Write Mock node + STUBBED_BY (InterfaceContract→Mock,
   props.fidelity). Verify it answers the contract (matchers fire, schema
   validates) before handing to devops to provision as a dependent container
   (PROVISIONS, placeholder flag for placeholder mocks). Manifest must show EVERY
   ExternalSystem satisfied by a real backend or a labeled placeholder mock.
```

## Procedure B — End-to-end tests from production traces

```
1. PULL prod traces via replay: REAL sampled prod traffic (req/resp, consumed/
   produced messages, DB I/O) for affected capabilities/endpoints — NOT synthetic
   happy paths. Prefer traces hitting businessRefs/PRESERVES rules + criticality-
   weighted, error, and edge-case journeys (representative, not convenient).

2. NORMALIZE & ANONYMIZE. Run each trace through the declared normalization
   allowlist (timestamps, generated IDs, key ordering, ephemeral ports, trace IDs
   — replay §5) and anonymize PII: detect + consistently pseudonymize names,
   emails, card/account numbers, tokens, free-text IDs with a stable deterministic
   map (referential integrity survives) BEFORE any fixture/scenario is persisted.
   No raw prod PII ever written to the artifact store.

3. SYNTHESIZE E2EScenario per capability. Group interactions into a coherent
   journey (e.g. "create invoice → charge payment provider → emit invoice.paid").
   Create E2EScenario node (source=prod-trace, journey, cassette ref,
   assertsEquivalence=true, relevant slo[] from Target Spec).

4. WIRE the driver at the stood-up Podman topology (devops Environment): send the
   real request(s) / publish recorded message(s) into the migrated service → its
   real dependents + the mocks (WireMock/Mountebank/Testcontainers/placeholder)
   for everything external. Re-inject recorded determinism seams (fixed clock,
   seeded RNG, deterministic UUID, frozen env) so the run reproduces.

5. ASSERT equivalence vs. baseline. Capture observed outputs (HTTP resp/status,
   emitted messages, final DB writes), diff (after normalization) against the
   recorded prod baseline (replay golden set). Reuse the validation/replay engine:
   classify each divergence benign (declared normalization rule) vs. behavioral
   (real value/status/message/effect change). Any behavioral divergence FAILS the
   scenario unless the plan authorizes the change.

6. CHECK monitoring signals (OTel → Prometheus/Loki/Grafana wired by devops):
   assert NO new error classes and the scenario's SLOs (latency, error rate,
   business KPIs from Target Spec) stay in bounds. Green diff + SLO breach / new
   error class is NOT a pass.

7. RECORD coverage + edges. Write VERIFIES → Capability/Endpoint covered,
   EXERCISES → every Service/ExternalSystem driven. Compute capability coverage:
   which Capability nodes (esp. criticality=high) are VERIFIED by ≥1 prod-trace
   scenario and which are not — gaps surfaced, not hidden. Emit the E2E report.
```

## Real traces, placeholder integrations — both fully tested
- **E2E comes from REAL prod traffic** — normalized + anonymized, never
  hand-written happy paths. This is what makes "behavioral equivalence" mean *the
  system still does what it did in production*.
- **Placeholder-backed integrations are still fully tested** — the E2E path runs
  against the placeholder mock built to the dummy contract and asserts the
  integration behaves as the typed placeholder promises; the unknown is loudly
  marked by its Gap, exercised honestly, never skipped. On SUPERSEDES, mock +
  scenarios are regenerated and re-run.

## Determinism & PII handling
- **Determinism seams** (= replay §"Determinism seams"): re-inject recorded
  **clock** (`Clock.fixed`), **random** (seeded `Random`/`SecureRandom`), **UUID**
  (seeded/counter), **env/props/locale/timezone** snapshot, so the same trace hits
  the same seams every run; else a false divergence (new UUID, fresh timestamp)
  masquerades as behavioral. Unordered collections normalized; ordered emissions
  replayed in order.
- **Anonymization / PII**: detect + replace PII with a **stable deterministic
  pseudonym map** at capture/normalize time, before any fixture/mock-seed/scenario
  is persisted. Stability preserves referential integrity across a multi-step
  journey (same customer = same pseudonym end-to-end and in seeded mock responses);
  **no raw PII ever lands in the artifact store**; the pseudonym map is not
  persisted in cleartext alongside the artifacts.

## How outputs feed INTEGRATE & risk-assessment
- **INTEGRATE exit** (integration-and-environments §6 / state-machine §9): your
  E2E suite is what `integration-verifier` replays. PASS requires E2E from prod
  traces pass with behavioral equivalence (crit. 2), monitoring shows no new error
  classes + SLOs hold (crit. 3), every external dep satisfied by a real backend or
  labeled placeholder mock — no dangling integration (crit. 4). Your manifest +
  report are the evidence for 2–4; failing any → re-plan or rollback, never
  advance to COMPLETE.
- **risk-assessment `equivalenceConfidence`** (RiskScore `factors`): the report
  contributes whole-system equivalence-confidence — a function of capability
  coverage, traffic representativeness, benign/behavioral divergence counts. Thin
  coverage, placeholder-heavy integrations, or low-confidence mocks **lower** it
  and raise risk, pausing weakly-proven changes. Gaps behind placeholder mocks
  already raise `gapRisk` via `HAS_GAP`.

## Graph writes

| Node / Edge | When | Key props / direction |
|-------------|------|-----------------------|
| `Mock` | one per external dep mocked | `engine` (wiremock\|mountebank\|testcontainers), `backs` (contract gid), `fidelity` (recorded\|generated\|placeholder), `cassette` (replay ref) |
| `STUBBED_BY` | mock registered | `InterfaceContract → Mock`, `props.fidelity` |
| `E2EScenario` | one per prod-trace journey | `source` (prod-trace), `journey`, `cassette`, `assertsEquivalence`, `slo[]` |
| `VERIFIES` | scenario covers a behavior | `E2EScenario → {Capability, Endpoint}` |
| `EXERCISES` | scenario drives a system | `E2EScenario → {Service, ExternalSystem}` |

Cross-repo scenarios (journey spanning `web-app` + `billing-svc`) set
`props.crossRepo = true` + the shared `contract` GID on spanning edges
(edge-types §Cross-repo), so `integration-verifier` can compose multiple migrated
services in one topology and replay cross-service journeys. GIDs follow the run
convention, e.g. `billing-svc::Mock::PaymentProvider@external`,
`billing-svc::E2EScenario::invoice-charge-paid`.

## Quality / invariants
- **Every external dependency is mocked — none skipped.** Manifest maps *every*
  `ExternalSystem` to a backing `Mock` (real or placeholder); a dep with no mock
  is a hard failure — there is always something at the other end of the wire.
- **Placeholders are loudly labeled** — `fidelity = placeholder` + back-link to
  its `Gap`; integration still exercised E2E against the dummy contract.
- **E2E comes from production traces** — `source = prod-trace`, normalized +
  PII-anonymized, not synthetic happy paths. Coverage leaning on
  generated/integration-test journeys is reported as such and lowers
  `equivalenceConfidence`.
- **Equivalence asserted vs. baseline** — every scenario diffs observed vs.
  recorded prod baseline; every divergence classified benign vs. behavioral;
  behavioral fails unless plan-authorized. Normalization may only canonicalize
  declared non-determinism, never mask a value change.
- **No live external calls during E2E** — external I/O served by
  mocks/Testcontainers; an unstubbed outbound call is a hard failure, never a
  silent passthrough (a passthrough makes the verdict a lie).
- **Determinism controlled** — clock/random/UUID/env seams re-injected from the
  recording; a divergence with no seam explanation is a finding, surfaced not
  silenced.
- **Capability coverage explicit** — report states which `Capability` nodes (esp.
  `criticality = high`) are VERIFIED and which are not.
- **No raw PII persisted** — anonymization runs before any artifact is written;
  the pseudonym map is stable but not stored in cleartext with artifacts.

## Definition of done
Every external dependency the scope touches has a backing `Mock` — from the real
contract (WireMock/Mountebank, or Testcontainers for DB/Kafka) or built to the
typed placeholder contract + labeled with its `Gap` — seeded from prod cassettes
where available, registered with `STUBBED_BY`, and recorded in a mock-coverage
manifest showing **no dangling integration**. Representative **production traces**
for the affected capabilities are captured via `replay`, normalized,
**PII-anonymized**, and turned into `E2EScenario` journeys that drive the whole
stood-up Podman system (service + dependents + mocks) with determinism seams
controlled. Each scenario asserts **behavioral equivalence vs. the production
baseline** and checks monitoring (no new error classes, SLOs in bounds), with
`VERIFIES`/`EXERCISES` edges and explicit **capability coverage** recorded. An E2E
suite + report is emitted for `integration-verifier` (INTEGRATE exit criteria 2–4)
and `risk-assessment` (`equivalenceConfidence`) — a red E2E never reports success.
