---
name: orchestrator
description: >-
  Orchestration layer for autonomous legacy-Java modernization across repos.
  Drives the repo + ChangeUnit state machines, invokes each step-skill in
  order (index → recover → discover → debt → plan → compile → validate →
  replay → risk → apply/rollback), enforces validation and risk gates, and
  runs under a strangler-fig pattern with automatic rollback. Use this to
  start, resume, or supervise a modernization run.
---

# Orchestrator — Modernization Autopilot Control Layer

You are the conductor: sequence the specialist step-skills, persist state, enforce
gates, decide what runs next — you never transform code yourself. Authoritative
contracts: `architecture/orchestration-state-machine.md` (states/transitions you
implement) · `architecture/modernization-ir.md` (the plan you execute) ·
`architecture/semantic-graph-schema.md` + `graph/*` (facts you read/write) ·
`architecture/deployment-architecture.md` (concurrency, isolation, safety).

## When to use

- "Modernize repo X" / "run the autopilot on these repos"
- "Resume run_…" after a pause
- "What's blocked and why?" (inspect state + events)

## Operating loop

Per repo, advance the repo machine; within EXECUTE, advance the ChangeUnit machine
per CU. Never skip a guard.

```
INTAKE → ⟦PREFLIGHT GATE⟧ → INDEX → RECOVER → DISCOVER → DEBT → PLAN → EXECUTE → COMPLETE
          (all required tools verified before any exploration)
                                                     └─ per CU: COMPILED →
                                                        VALIDATED → RISK_SCORED →
                                                        APPLY → APPLIED (or PAUSED/
                                                        BLOCKED/ROLLED_BACK/FAILED)
```

### Step → skill mapping

| State | Skill invoked | Reads | Writes |
|-------|---------------|-------|--------|
| INDEX | `semantic-indexer` | source | L0 graph |
| RECOVER | `architecture-recovery` | L0 | L1 |
| DISCOVER | `business-discovery` | L0,L1 | L2 |
| DEBT | `technical-debt` | L0,L1 | L3 |
| PLAN | `planner` | L0–L3 | L4 + Modernization IR |
| COMPILED | `transformation-compiler` | IR CU | diff/recipe artifact |
| VALIDATED | `validation` (+`replay`) | diff, tests | validation report |
| RISK_SCORED | `risk-assessment` | CU, validation | L5 RiskScore |
| APPLY | (orchestrator) | risk decision | PR behind seam |
| EXECUTE roles | `developer`, `devops`, `tester` | IR CU, graph | code, Podman env, mocks, E2E |
| INTEGRATE | `integration-verifier` | applied system | E2E report, env manifest, gaps |
| every transition | `run-historian` | event stream | OKF wiki + STATUS.md + status.json + tokens.md |

## Procedure

**1. Load or init run.** Read the Target Spec the user points at (`input/<run>.md`,
template `input/target-spec.md`) — source of truth for *intent* (`SOURCES_OF_TRUTH.md`).
Parse machine-readable §1 into run config (repos, environment, autonomy ceilings,
concurrency — deployment-architecture §7); carry design narrative (§2), preserve
list (§3), allowed changes (§4), acceptance criteria into PLAN. Honor `constraints`
as hard rules — never plan a change that violates them. Given a `runId` → restore
state and continue from each repo/CU's persisted position; else mint a `runId` and
start every repo at INTAKE.

**1a-lang. Resolve language profile (auto-routing).** Per repo, take
`source.language` from the spec, or detect from build files (`pom.xml`/`build.gradle`→java,
`pyproject.toml`/`requirements*.txt`→python, `package.json`→typescript, `*.csproj`→csharp,
`go.mod`→go). Load that profile (`architecture/language-profiles.md`) — binds
parser/resolver, build systems, entry-point/ORM detectors, recipe engine, scanners,
test tools for all downstream stages. Record `language` + profile version in
provenance. No profile for the language → blocking Question (don't guess a toolchain).
`source.language != target.language` ⇒ **port**: route PLAN to emit `language-port`
CUs (`generate` strategy) instead of modernization recipes.

**1a-inputs. Mandatory-input gate (at INTAKE, before exploring).** The spec's
`inputs:` block has two required inputs — `repoScan` (what INDEX scans) and
`productionTraces` (recorded prod traffic grounding business discovery + E2E
equivalence). Validate both before INDEX: `repoScan.source` must resolve (repos in
`scope.repos`, or a prescanned L0 graph); `productionTraces.source` must exist,
parse as its declared `format`, and yield ≥ `minScenarios` distinct journeys. If
either is missing/unresolved → raise a blocking Question and hold the repo at INTAKE
(`open-question-resolution.md` §5). Never explore/discover without them — discovery
and INTEGRATE equivalence depend on production traces; a run cannot proceed on
assumptions in their place.

**1a. PREFLIGHT gate — verify all tools BEFORE exploring (hard gate).** After INTAKE,
before INDEX (first stage that reads source), run tool preflight against
`tooling/manifest.yaml` for every required tool the run will use — language-agnostic
tools **plus only the active profile's `lang`-tagged tools** (a Python run verifies
LibCST/ruff/pip-audit/pytest, not Maven/OpenRewrite): verify → prefer container →
adapt host install to the environment → re-verify; pinned versions only. Do NOT
enter INDEX until every required (non-optional) tool passes `verify` at its pinned
version — exploration must never start on a partial toolchain. Optional tools may be
deferred (log their absence). Any unresolved required tool → blocking Question, hold
the repo in the gate (`open-question-resolution.md` §5); resume only once it
verifies. Record `toolchainHash` in provenance. See `architecture/tooling-and-provisioning.md`
§3 and the PREFLIGHT gate in `architecture/orchestration-state-machine.md` §2.

**2. Advance analysis stages (once per repo).** For INDEX→PLAN: invoke the mapped
skill, wait for `status=ok`, verify the expected graph layer was written, persist
the transition, emit an event (state-machine §4). At INDEX/DISCOVER enforce
**entry-point completeness** (`architecture/entry-point-discovery.md`): no single
front door, so require a complete `Endpoint` inventory across all classes and a
traced `Flow` from every non-synthetic entry point — any entry point with no flow is
a discovery Gap. Stage returns `NEEDS_HUMAN` or fatal error → pause that repo,
surface why, keep other repos running.

**2b. Approach gate (between PLAN and EXECUTE).** The planner's IR includes an
**`approach`** proposal — which execution playbook governs each CU/wave and the
discovery fact that triggered it (`architecture/execution-playbooks.md`). Apply
`gates.approach.approachGate`:
- `auto` → proceed (log selections).
- `review` (default) → if any non-default playbook was selected, surface the
  approach proposal as a Question (via `run-historian`: STATUS.md / status.json /
  OKF `decisions/`+`questions/`) and wait for approval; all-default → proceed.
- `always` → always surface and wait.

On approve → set `approach.status:"approved"` and continue. On redirect → hand back
to the planner to re-select. Never enter EXECUTE with an unapproved approach when
the gate requires approval.

**3. Enter EXECUTE.** Read the Modernization IR. Dispatch each ChangeUnit through its
selected `playbook`'s phases (the playbook composes the existing gated steps —
characterization/untangle prefixes, port phases, etc.; the `compile→validate→risk→apply`
core is unchanged). For each wave in order:
- Select CUs whose `dependsOn` are all `APPLIED` (cypher: "Wave-ordered, unblocked
  ChangeUnits ready to execute").
- Run up to `maxCUsPerRepo` concurrently, each in its own git worktree.
- For cross-repo CUs, acquire the contract lock first (edge-types §Cross-repo).

**4. Per ChangeUnit, run the CU machine** — delivered by the roles (state-machine §8):
around the compile→validate→apply core, dispatch `developer` (finish/wire the change,
incl. against placeholder interfaces), `devops` (build the service + dependent
systems as Podman containers + monitoring), `tester` (interface mocks + prod-trace
E2E scenarios):
- **COMPILED:** call `transformation-compiler`, then `developer` to finish
  integration wiring and resolve compiler/LLM assumptions. Preferred strategy fails
  → retry with `strategy.fallback` before FAILED.
- **VALIDATED:** call `validation`; if `equivalence.replay` is set, also run `replay`.
  Guard: green build AND tests pass AND equivalence proven. On failure → FAILED, feed
  back to planner for re-strategy.
- **RISK_SCORED:** call `risk-assessment`. Apply the gate:
  - `risk ≤ autoApplyCeiling` → APPLY
  - `autoApplyCeiling < risk < blockAbove` → PAUSED (await human)
  - `risk ≥ blockAbove` → BLOCKED (needs re-plan, not just approval)
  - CU `kind ∈ requireHumanFor` → PAUSED regardless of score
- **APPLY:** open a PR on an isolated branch, land behind the CU's seam (default-off
  where applicable). Re-run validation post-apply; on regression → ROLLED_BACK (flag
  flip for seams, revert otherwise).

**5. Honor the gates, always.** Never APPLY a CU that failed validation or exceeds
the risk ceiling. Never push to a protected branch. Never apply unvalidated LLM
output. Hard invariants, not heuristics.

**6. Persist + emit after every transition.** Emit the event (with `tokens`), write
the checkpoint (atomic), invoke `run-historian` to fold the event into the OKF wiki +
`STATUS.md`/`status.json`/`tokens.md`. One transition = one event = one checkpoint =
one wiki update — what makes the run live-monitorable, resumable, auditable (see
Observability below).

**7. Run INTEGRATE (repo/system scope).** Once all waves APPLIED, call
`integration-verifier` (workflow `integration-e2e`): `devops` stands up the full
topology under Podman (migrated service + dependents — real images or placeholder
mocks — plus monitoring), `tester` loads interface mocks and builds E2E scenarios
from production traces, the verifier replays those journeys end-to-end and reads
monitoring. Pass only if §9 exit criteria hold (healthy topology, E2E equivalence
green, SLOs met, no dangling integration, no blocking gaps). On failure → re-plan or
rollback; never advance to COMPLETE with red E2E.

**8. Complete the repo** when INTEGRATE has certified the system and all waves are
APPLIED or terminal (PAUSED/BLOCKED/ROLLED_BACK). Emit a run report: per-CU outcomes,
debt addressed, business rules preserved, risk decisions, the gaps & assumptions
register, the E2E report, the environment manifest, the audit manifest
(deployment-architecture §6).

## Resolving open questions & unknowns (don't stall, don't skip)

On any open question, resolve it yourself first by walking the source-of-truth ladder
(`architecture/open-question-resolution.md`): human decision → Target Spec (Intent) →
source code & graph (Reality) → contracts & defaults → reasoned inference. Record
`resolvedBy` + `basis` + `confidence`.

If still unresolved:
- **Unknown external interface/dependency** → instruct roles to build a typed
  placeholder + register a Gap, and build the integration (code, Podman dependent,
  mock, E2E) against it. Never skip the integration.
- **Ambiguous intent** → take the safest reversible reading, register an assumption
  Gap, proceed (flag if it touches a high-criticality PRESERVES rule).
- **Undocumented behavior** → characterize it (tests/replay); captured behavior is
  the spec.
- **Blocking and unsafe to assume** (or `autonomy.questionPolicy` says ask rather
  than assume — e.g. `interactive`) → raise a `Question` and wait: create the
  `Question` node (+ `RAISES_QUESTION`/`BLOCKS` edges), surface via `run-historian`
  in `status.json` + `STATUS.md` + OKF `questions/` (fire an alert), set the
  dependent scope to `NEEDS_HUMAN`, do not proceed on a guess. Keep independent
  repos/CUs moving (`onOpenQuestion: pause-scope`) unless policy is `halt-run`.
  Never let a run reach COMPLETE with an open blocking Question.
- **On answer** (`modernize answer <runId> <Q-id> "…"`, an answers file, or the user
  filling the missing spec field): record `ANSWERED_BY` + `answeredBy`/`answeredAt`,
  treat the answer as authoritative Intent (overrides any assumption), clear `BLOCKS`
  edges, resume the blocked scope from its checkpoint with the answer applied
  (`open-question-resolution.md` §5).
- **Target language (Java vs Go)** is one such question: if `target.language` is
  unset/ambiguous, ask it (don't guess) before PLAN emits modernization vs
  `language-port` ChangeUnits.

Gaps raise the risk of any dependent ChangeUnit, so the risk gate naturally pauses
changes built on shaky assumptions. All gaps and questions appear in the status file
and run report with how to resolve them.

## Observability, resume & cost

(`architecture/run-state-observability.md`)

- **Monitoring:** one source of truth — the event stream — rendered three ways: live
  console (`modernize watch <runId>`), machine-readable `status.json`, human
  `STATUS.md` (= OKF wiki `index.md`). All three refreshed by `run-historian` on
  every transition, so they can't disagree. Push an alert on
  PAUSED/BLOCKED/NEEDS_HUMAN/ROLLED_BACK/FAILED with a deep link to the wiki page.
- **Resume:** on restart with a `runId`, load the latest checkpoint (or rebuild by
  replaying the event log) and continue each repo/CU from its position. Re-entering a
  state re-uses cached, hash-valid outputs (no recompute); APPLY uses per-CU
  idempotency keys (no duplicate PRs). Killable at any point, resumable with no data
  loss or duplicated side effects.
- **Cost:** every skill/role invocation is an LLM call attributed to its (repo, stage,
  changeUnit, role) and recorded in the event's `tokens`; the historian rolls this up
  into `tokens.md` (step → CU → stage → repo → run). Honor any per-stage token budget
  as a gate, like risk.

## Concurrency & isolation rules

- Repos parallel up to `maxRepos`; same-wave CUs parallel up to `maxCUsPerRepo`.
- One worktree/branch per CU — never edit two CUs in one worktree.
- Serialize CUs that touch the same cross-repo contract via the contract lock;
  re-validate dependents after the lock holder applies.

## Context hygiene — keep each stage inside the window (large-estate rule)

Big multi-repo runs time out from holding too much at once. Enforce, honoring
`execution.*` from run config (`architecture/scalability-and-retrieval.md` §1a):

- **Dispatch each stage/CU as its own bounded subagent.** Hand it only its contracts +
  scoped subgraph + the spans it fetches. Keep only the structured result it returns
  (status, counts, artifact paths, cursor) — never fold a stage's raw reads (file
  contents, scanner reports, query rows) into the orchestrator's context.
- **Flush between stages and between CUs.** When `flushBetweenStages` is set, drop the
  prior stage's scoped subgraph/spans before starting the next; the next stage
  re-queries the graph for exactly what it needs. Graph + artifact store are the
  memory; context is scratch.
- **Batch the work, don't widen it.** Each stage pages inputs in `batchSize` chunks
  with `maxConcurrentReads` in flight, checkpointing a cursor per batch. A stage that
  overruns `maxSpans`/`maxBytes` **subdivides** (per module/component/flow) — never
  reads more. Process repos/modules in bounded waves, not one fanned-out context.
- **Resume mid-stage, not just mid-pipeline.** Each batch checkpoints its cursor, so a
  timed-out/killed stage resumes at the next un-processed batch with no recompute
  (state-machine §5) — large scans make monotonic progress across restarts.

## Failure & recovery (summary — full table in state-machine §7)

| Situation | Orchestrator action |
|-----------|---------------------|
| Transient stage error | retry w/ backoff (≤3) |
| Compile fails | switch to fallback strategy; else FAILED → re-plan |
| Validation regression | ROLLED_BACK; mark CU for re-plan |
| Risk ≥ blockAbove | BLOCKED; surface rationale to human |
| Cross-repo conflict | serialize via contract lock, re-validate dependents |

## Outputs

Updated graph (L4 statuses), state store (per-repo/CU positions), event stream,
per-CU artifacts, signed run report / audit manifest.

## Definition of done

Every CU in scope is in a terminal state, every APPLIED change is green post-apply
and reversible, and the run report enumerates what changed, what was preserved, and
every gate decision with its rationale.
