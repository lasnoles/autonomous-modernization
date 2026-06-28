# Integration, Environments & End-to-End Verification

Migrating code is not "done" until the **whole system runs end-to-end** —
the migrated service, its dependent systems, monitoring, and realistic traffic.
This document defines the delivery roles, how dependent environments are stood
up (Podman), how interfaces are mocked, how end-to-end tests are built from
production traces, and what "the system works" means as an exit criterion.

This adds an **INTEGRATE** stage to the repo state machine (after EXECUTE,
before COMPLETE) and a **role model** for the code-write/execute phase.

## 1. Delivery roles

During EXECUTE and INTEGRATE the orchestrator dispatches four collaborating
roles (each a skill under `skills/`). They are distinct responsibilities, not
distinct people — the autopilot plays all of them.

| Role | Skill | Owns |
|------|-------|------|
| **Developer** | `developer` | Writes/finishes the migration code for a ChangeUnit; integrates against real or placeholder interfaces; keeps the build green. |
| **DevOps** | `devops` | Builds & runs the dependent systems and the migrated service as **Podman** containers/pods; provisions **monitoring/observability**; manages env lifecycle. |
| **Tester** | `tester` | Builds **interface mocks** (real contracts where known, placeholders where not) and **end-to-end integration tests from production traces**. |
| **Integration Verifier** | `integration-verifier` | Owns the INTEGRATE stage: stands the full system up, runs E2E, reads monitoring, and certifies the system works end-to-end. |

The existing `transformation-compiler` produces the mechanical diff; the
**developer** role finishes anything the compiler/LLM left as an assumption,
wires integrations (including against placeholders), and resolves build issues.

## 2. Environments via Podman (DevOps)

Every environment is **ephemeral, reproducible, and rootless**. DevOps
materializes a runnable system from the graph's dependency facts.

- **Containerize** the migrated service (multi-stage build, pinned base image,
  the run's pinned JDK).
- **Stand up every dependent system** discovered as an `ExternalSystem` in the
  graph:
  - Known real dependents (DB, cache, broker) → official images
    (e.g. `postgres`, `redis`, `rabbitmq`) seeded with fixtures.
  - **Unknown dependents (placeholders) → a mock service container** built to
    the placeholder `InterfaceContract` (see §4). *Integration is never skipped*
    — there is always something at the other end of the wire.
- **Compose the topology** with rootless Podman:
  - `podman play kube <pod-spec.yaml>` for pod/topology definitions, or
  - `podman-compose -f environment.yaml up` for compose-style stacks,
  - a dedicated `podman network` per environment for isolation,
  - **health checks** on every service so readiness is deterministic.
- Environments are recorded as `Environment` nodes (graph) listing services,
  images, and monitoring endpoints, so a run is reproducible and auditable.

Example topology (conceptual) the devops role generates:

```yaml
# environment.yaml (rendered by devops from graph ExternalSystem facts)
services:
  billing-svc:        { image: localhost/billing-svc:run-xyz, ports: ["8080"], healthcheck: /actuator/health }
  postgres:           { image: docker.io/postgres:16, env: {...}, seed: fixtures/billing.sql }
  rabbitmq:           { image: docker.io/rabbitmq:3-management }
  payment-provider:   { image: localhost/mock-payment-provider:run-xyz, note: "PLACEHOLDER mock (GAP-007)" }
observability:
  otel-collector:     { image: docker.io/otel/opentelemetry-collector }
  prometheus:         { image: docker.io/prom/prometheus,  scrape: [billing-svc] }
  grafana:            { image: docker.io/grafana/grafana,  dashboards: [billing-e2e] }
  loki:               { image: docker.io/grafana/loki }
```

## 3. Monitoring & observability (DevOps)

The env ships with an observability stack so end-to-end behavior is *visible*,
not assumed:

- **Traces/metrics/logs** via an OpenTelemetry collector → Prometheus (metrics),
  Loki (logs), Grafana (dashboards). Tempo/Jaeger optional for traces.
- **SLOs from the Target Spec** (latency, error rate, business KPIs) become
  Prometheus alert rules and Grafana panels.
- The INTEGRATE stage and the strangler-fig **cutover** read these signals:
  an SLO breach during cutover triggers `workflows/rollback.yaml`.

## 4. Interface mocks (Tester)

For every external interface the system depends on, the tester builds a mock so
the system is runnable and testable in isolation:

- **Known contracts** → mocks generated from the real spec (OpenAPI/proto/schema)
  using service virtualization (**WireMock**, **Mountebank**) or
  **Testcontainers** for real-protocol backends (DB, Kafka).
- **Unknown contracts (placeholders)** → a mock built to the placeholder
  `InterfaceContract` shape (`open-question-resolution.md` §3). It returns
  plausible typed responses and is loudly labeled with its `Gap`. This is how we
  **don't skip the integration** — the mock stands in for the unknown system so
  the integration path is exercised end-to-end.
- Mocks are seeded, where possible, from **recorded production traces** (cassettes
  captured by the `replay` skill) so responses mirror real-world shapes.

## 5. End-to-end tests from production traces (Tester)

The E2E suite is built from **real production traffic**, not synthetic happy
paths:

1. `replay` captures production request/response traces (HTTP, messages, DB I/O)
   as cassettes, with determinism seams controlled (clock/random/UUID).
2. The tester turns representative traces into **end-to-end scenarios** that
   drive the *whole stood-up system* (service + dependents + mocks) through real
   user journeys.
3. Each scenario asserts **behavioral equivalence** vs. the baseline outputs and
   checks the monitoring signals (no new errors, SLOs met).
4. Scenarios map to `Capability`/`Endpoint` nodes (`VERIFIES` edges) so coverage
   of business capabilities is explicit.

This reuses and extends the `replay` skill (record/replay + determinism) and the
`validation` skill (equivalence), elevated from per-ChangeUnit to **whole-system**.

## 6. The INTEGRATE stage — "is the end-to-end system working?"

After all ChangeUnits in a repo (and its cross-repo partners) are APPLIED, the
`integration-verifier` runs INTEGRATE:

```
1. devops:   bring up the full system + dependents + monitoring (Podman)
2. tester:   load mocks (real + placeholder) and the prod-trace E2E suite
3. verify:   replay production journeys end-to-end across services
4. observe:  read monitoring — errors, latency, SLOs, business KPIs
5. certify:  PASS only if E2E green AND SLOs met AND no unresolved blocking gap
```

**Exit criteria (definition of "system works"):**
- Full topology comes up healthy under Podman (all health checks green).
- E2E scenarios from production traces pass with behavioral equivalence.
- Monitoring shows no new error classes and SLOs are within target-spec bounds.
- Every external dependency is satisfied by a real backend or a labeled
  placeholder mock — **no dangling integration**.
- Remaining gaps are non-blocking and listed in the run report.

On failure → feed back to the planner (re-strategy) or rollback; the run does
not reach COMPLETE claiming success when E2E is red.

## 7. Cross-repo end-to-end

Because the graph spans repos, INTEGRATE can compose **multiple migrated
services together** (e.g. `web-app` + `billing-svc`) in one Podman topology and
replay cross-service production journeys — proving the *system*, not just each
service, still works after modernization.
