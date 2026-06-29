# Autonomous Modernization Autopilot

An agentic system that modernizes legacy **Java** codebases across many
repositories — autonomously, but **gated**. It builds a semantic understanding
of each system, discovers the business behavior worth preserving, plans a safe
sequence of changes, compiles them into deterministic transformations (with an
LLM fallback), proves behavioral equivalence, and applies them under a
strangler-fig pattern with automatic rollback.

> Core stance: **deterministic-first, LLM-fallback; behavior-preserving by
> default; every irreversible action passes a validation gate and a risk gate.**

## How it fits together

```
orchestrator (state machine)
  └─ for each repo:  INDEX → RECOVER → DISCOVER → DEBT → PLAN → EXECUTE → INTEGRATE → COMPLETE
                       │       │          │         │      │       │          └─ stand up full
                       │       │          │         │      │       │             system (Podman) +
                       │       │          │         │      │       │             monitoring, run E2E
                       │       │          │         │      │       │             from prod traces,
                       │       │          │         │      │       │             certify it works
                       │       │          │         │      │       └─ per ChangeUnit (roles:
                       │       │          │         │      │          developer+devops+tester):
                       │       │          │         │      │          COMPILE → VALIDATE → RISK
                       │       │          │         │      │          → APPLY/ROLLBACK
                       ▼       ▼          ▼         ▼      ▼
                  semantic  arch.     business   tech.  planner ──emits──▶ Modernization IR
                  -indexer  recovery  discovery  debt
                     │         │          │         │
                     └────── all read/write the SEMANTIC GRAPH (L0–L5) ──────┘
```

Everything the skills know is materialized as facts in a layered **semantic
graph** (L0 structure → L1 architecture → L2 business → L3 debt → L4 plan →
L5 risk), so each stage queries rather than re-parses.

## Repository layout

| Path | What it holds |
|------|---------------|
| `input/` | **Where you drive a run.** `target-spec.md` is the fill-in template for your modernization intent (objective, constraints, target design, behavior to preserve, autonomy). Copy it per run. |
| `examples/` | A fully worked run rendered as real output — the OKF run wiki, `status.json`, `checkpoint.json`, and the Podman `environment.yaml`. Start at `examples/run_2026_06_28_001/wiki/index.md`. |
| `SOURCES_OF_TRUTH.md` | The authority hierarchy — which artifact wins when two disagree (Intent > Reality > Contracts > Derived). Read before trusting any output. |
| `OPERATOR-GUIDE.md` | **For whoever runs the autopilot** — start/watch/answer/approve/resume/rollback, the `modernize` commands, and a daily-session walkthrough. |
| `architecture/` | The authoritative contracts: system design, graph schema, **Modernization IR**, **orchestration state machine**, deployment. Read these first. |
| `graph/` | Graph ontology, node types, edge types, reusable Cypher queries. |
| `.claude/skills/` | The agent skills — `orchestrator` (the control layer) plus one per pipeline step. Each has a `SKILL.md`. Lives under `.claude/` so Claude Code auto-discovers them. |
| `prompts/` | LLM prompt templates (planner, reviewer, architect, test-generator, patch-generator). |
| `workflows/` | Declarative pipelines binding the state machine to skills (`modernization`, `strangler-fig`, `seam-extraction`, `rollback`). |
| `recipes/` | The transformation catalogs the compiler draws from (OpenRewrite, **python** (LibCST/ruff/pyupgrade), ts-morph, Roslyn, llm-fallback, port profiles). |
| `tooling/` | `manifest.yaml` — the pinned list of tools each stage needs + how to verify them. Resolved at INTAKE by preflight (`architecture/tooling-and-provisioning.md`); portable across environments, never floating versions. |

## The skills

| Skill | Stage | Graph layer | Role |
|-------|-------|-------------|------|
| `orchestrator` | all | — | Drives the state machine, enforces gates, sequences the rest. |
| `semantic-indexer` | INDEX | L0 | Parse → symbols, call graph, data flow, endpoints, deps. |
| `architecture-recovery` | RECOVER | L1 | Components, layers, violations, candidate seams. |
| `business-discovery` | DISCOVER | L2 | Capabilities, business rules, domain entities (behavior to preserve). |
| `technical-debt` | DEBT | L3 | Debt items, vulnerabilities, hotspots — scored. |
| `planner` | PLAN | L4 | Emits the Modernization IR: ChangeUnits, waves, seams, gates. |
| `transformation-compiler` | COMPILE | — | Lowers a ChangeUnit to a candidate diff (recipe → codemod → llm-patch). |
| `validation` | VALIDATE | — | Build + tests + behavioral equivalence; backfills characterization tests. |
| `replay` | VALIDATE/audit | — | Golden record-replay equivalence + run reproducibility snapshots. |
| `risk-assessment` | RISK | L5 | Calibrated risk score + auto/pause/block decision with explanation. |
| `developer` | EXECUTE | — | Finishes the compiler's diff into working, integrated code (incl. against placeholder interfaces). |
| `devops` | EXECUTE / INTEGRATE | — | Stands up the service + dependents as **Podman** containers + monitoring. |
| `tester` | EXECUTE / INTEGRATE | — | Builds interface mocks + **end-to-end tests from production traces**. |
| `integration-verifier` | INTEGRATE | — | Runs the full system end-to-end and certifies it works. |
| `run-historian` | every transition | — | Folds the event stream into the OKF run wiki + live STATUS + token rollup. |

## Scale & multiple languages

- **Tools, portably.** Tools are *declared* with pinned versions in
  `tooling/manifest.yaml`; an INTAKE **preflight** verifies them, prefers
  containers, and *adapts the host install to whatever environment you're on* —
  then re-verifies. Same pinned toolchain everywhere, no fixed setup script. See
  `architecture/tooling-and-provisioning.md`.
- **Huge repos without re-reading.** Source is fully read once, at INDEX;
  thereafter skills read the *graph* and fetch only the exact source span by GID
  (hash-gated, cached) — never whole files. Indexing shards per module, loads
  subgraphs lazily, and runs each stage as a bounded subagent. See
  `architecture/scalability-and-retrieval.md`.
- **Multiple languages via profiles.** Declare `source.language` (java, python,
  typescript, csharp, go) and INTAKE loads that **language profile**, auto-routing
  every stage to the right parser, resolver, entry-point/ORM detectors, recipe
  engine, scanners, and test tools — no per-language fork of the skills. Adding a
  language is one profile + tool entries. See `architecture/language-profiles.md`
  (Java and **Python** are the two reference profiles).
- **Including cross-language ports.** Same-language modernization uses AST
  recipes/codemods; a change of language (e.g. **Java→Go**, **Python→Go**) is a
  `language-port` ChangeUnit that *regenerates* the component from its
  language-neutral spec (rules, flows, contracts, replay traces) and proves
  equivalence **black-box** at the interface — the same replay/E2E machinery, now
  comparing two languages. See `architecture/multi-language-and-porting.md`.

## Discovery, monitoring, resume & the run wiki

- **No single entry point.** Discovery enumerates *every* entry surface (HTTP,
  messaging, scheduled, batch, CLI/`main`, webhook, startup, file, library) and
  traces a `Flow` from each — a zero-flow entry point is a gap, not a blind spot.
  See `architecture/entry-point-discovery.md`.
- **Monitor where the agent is.** One source of truth (the event stream) drives a
  live console (`modernize watch <runId>`), `status.json`, and `STATUS.md` — all
  refreshed every transition so they never disagree. See
  `architecture/run-state-observability.md`.
- **Resume after anything + per-step token cost.** Every transition checkpoints
  with token usage; on restart the run reloads the checkpoint (or replays the
  event log) and continues with no recompute and no duplicate applies. Token cost
  rolls up step → CU → stage → run.
- **The run wiki = OKF bundle.** The run continuously emits an Open Knowledge
  Format bundle (indexed Markdown, cross-linked, `log.md` = the state machine).
  It is both the live monitor and the permanent "what happened" record — see
  `architecture/okf-run-wiki.md` and the worked `examples/`.

## Autonomy: open questions, placeholders & end-to-end delivery

- **It resolves its own open questions** by walking the source-of-truth hierarchy
  (Intent → Reality → Contracts → inference) before ever asking a human —
  recording how each was answered. See `architecture/open-question-resolution.md`.
- **It never skips an integration.** When an external interface/dependency is
  unknown, it builds a **typed placeholder + a tracked Gap**, codes the
  integration against it, mocks it, and tests it — then supersedes it when the
  real interface arrives. Unknowns are loud, not silent.
- **It delivers a working system, not just edited files.** The EXECUTE phase is
  staffed by roles — `developer` (code), `devops` (Podman dependents +
  monitoring), `tester` (mocks + prod-trace E2E) — and the **INTEGRATE** stage
  stands the whole system up, replays production journeys end-to-end, reads
  monitoring, and **certifies the end-to-end system works** before COMPLETE.
  See `architecture/integration-and-environments.md`.

## Read order for a new contributor

0. `input/target-spec.md` — how you express what to modernize, and
   `SOURCES_OF_TRUTH.md` — which artifact wins when two disagree.
1. `architecture/system-design.md` — the big picture.
2. `architecture/orchestration-state-machine.md` — how a run flows (incl. roles + INTEGRATE).
3. `architecture/modernization-ir.md` — the plan format everything revolves around.
4. `architecture/semantic-graph-schema.md` + `graph/*` — the shared data model.
4b. `architecture/open-question-resolution.md` + `architecture/integration-and-environments.md`
   — how unknowns become placeholders and how the system is proven end-to-end.
4c. `architecture/entry-point-discovery.md`, `architecture/run-state-observability.md`,
   `architecture/okf-run-wiki.md` — all-flows discovery, monitoring/resume/tokens,
   and the OKF run wiki. Then browse `examples/run_2026_06_28_001/wiki/index.md`.
4d. `architecture/scalability-and-retrieval.md` (huge repos: read the graph, fetch
   spans by GID, never re-read whole files) and
   `architecture/multi-language-and-porting.md` (same-language modernization vs.
   cross-language ports like Java→Go).
4e. `architecture/execution-playbooks.md` — how the planner fits the *execution
   method* to what the scan found (playbook catalog) and lets you review/approve
   the approach after discovery before it runs (the L2 approach gate). Best for
   complex, heterogeneous estates.
5. `.claude/skills/orchestrator/SKILL.md`, then the step skills in pipeline order.

## Running a modernization (conceptually)

1. **Write your intent** — copy `input/target-spec.md` to `input/<run>.md` and
   fill in the objective, repos, target design, behavior to preserve, and
   autonomy ceilings. This is the source of truth for *what you want*.
2. **Point the `orchestrator` at it** — it parses the spec's machine-readable
   block into the `workflows/modernization.yaml` run config + Modernization IR
   `scope`/`gates`, and feeds the design narrative to the planner/architect.
3. **Run `environment: dry-run` first** — the autopilot analyzes, plans,
   compiles, and validates against your spec **without opening any PRs**, so you
   review the plan it derives before anything is applied.

See `input/target-spec.md` for the full, annotated template and
`SOURCES_OF_TRUTH.md` for how intent, source code, and contracts rank.

## Status

This repository is the **design + skill specification** for the autopilot. The
`architecture/` and `graph/` docs are the contracts; the `.claude/skills/*/SKILL.md`
files are the executable instructions for each agent; `workflows/` and
`recipes/` are scaffolded with the contracts and starter catalogs.
