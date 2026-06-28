# Orchestration State Machine

The orchestrator drives each repo (and each ChangeUnit within it) through a
deterministic state machine. This document is the **authoritative contract**
for states, transitions, guards, and events. The `orchestrator` skill
implements it; every step-skill is invoked as a state action.

## 1. Two nested machines

1. **Repo machine** — coarse pipeline position for a whole repository.
2. **ChangeUnit machine** — fine-grained lifecycle of a single planned change.

The repo machine runs the analysis stages once; the ChangeUnit machine runs
per CU during the apply stages and can execute many CUs concurrently
(bounded by wave membership and a concurrency cap).

## 2. Repo machine

```
        ┌──────────┐
        │  INTAKE  │  clone/refresh worktree, detect build system, pin toolchain,
        │          │  run tool preflight (tooling/manifest.yaml — verify→container→adapt-install→verify)
        └────┬─────┘
             ▼
        ┌──────────┐
        │  INDEX   │  → skill: semantic-indexer        (writes L0)
        └────┬─────┘
             ▼
        ┌──────────┐
        │ RECOVER  │  → skill: architecture-recovery   (writes L1)
        └────┬─────┘
             ▼
        ┌──────────┐
        │ DISCOVER │  → skill: business-discovery      (writes L2)
        └────┬─────┘
             ▼
        ┌──────────┐
        │  DEBT    │  → skill: technical-debt          (writes L3)
        └────┬─────┘
             ▼
        ┌──────────┐
        │   PLAN   │  → skill: planner                 (emits Modernization IR, writes L4)
        └────┬─────┘
             ▼
        ┌──────────┐
        │ EXECUTE  │  ── spawns ChangeUnit machine per CU, wave by wave ──┐
        └────┬─────┘   (roles: developer + devops + tester — see §8)      │
             ▼                                                            │
        ┌──────────┐                                                      │
        │INTEGRATE │  → skill: integration-verifier                       │
        └────┬─────┘   stand up full system (Podman) + monitoring,        │
             │         run E2E from prod traces, certify it works         │
             ▼                                                            │
        ┌──────────┐                                                      │
        │ COMPLETE │  E2E green & SLOs met; emit run report + gaps list    │
        └──────────┘                                                      │
                                                                          ▼
                                                          (see ChangeUnit machine §3)
```

Guards between analysis stages: each stage must write its layer with
`status=ok` and no fatal errors before the next begins. INTEGRATE's guard is the
end-to-end exit criteria in §9. Any stage may emit `NEEDS_HUMAN` to pause the
repo machine — but only after exhausting the open-question resolution ladder
(`architecture/open-question-resolution.md`): the system self-resolves from the
sources of truth and falls back to typed placeholders + tracked gaps before
escalating.

## 3. ChangeUnit machine

```
 PLANNED
    │  guard: all dependsOn APPLIED
    ▼
 COMPILED ──────► skill: transformation-compiler   (produces diff/recipe)
    │  guard: compile produced a candidate patch
    ▼
 VALIDATED ─────► skill: validation (+ replay if equivalence.replay)
    │  guard: build green AND tests pass AND equivalence proven
    │  fail ─► FAILED ─► (planner re-plan or escalate)
    ▼
 RISK_SCORED ───► skill: risk-assessment
    │
    ├─ risk ≤ autoApplyCeiling ───────────────► APPLY
    ├─ autoApplyCeiling < risk < blockAbove ──► PAUSED (await human)
    └─ risk ≥ blockAbove ─────────────────────► BLOCKED
    ▼
 APPLY ─────────► open PR on isolated branch, land behind seam
    │  guard: post-apply validation still green
    │  regression detected ─► ROLLED_BACK (seam flip / revert)
    ▼
 APPLIED
```

### 3.1 Terminal & recovery states
- `APPLIED` — change is live (possibly behind a flag at default-off).
- `PAUSED` — waiting on human approval; resumable.
- `BLOCKED` — risk too high; requires plan change, not just approval.
- `ROLLED_BACK` — reverted automatically; feeds back to planner.
- `FAILED` — compile/validate failure; feeds back to planner for re-strategy
  (e.g. switch `strategy.preferred` from `recipe` to `llm-patch`).

## 4. Events (emitted on every transition)

```jsonc
{
  "ts": "2026-06-28T10:15:00Z",
  "runId": "run_2026_06_28_001",
  "repo": "billing-svc",
  "scope": "changeUnit",            // repo | changeUnit
  "id": "CU-014",
  "from": "VALIDATED",
  "to": "RISK_SCORED",
  "skill": "risk-assessment",
  "result": { "risk": 0.31 },
  "artifacts": ["artifacts/CU-014/risk.json"],
  "tokens": { "in": 51002, "out": 12380, "total": 63382, "model": "claude-opus-4-8" },
  "checkpoint": { "event": 1487 }    // cursor folded into the checkpoint after this transition
}
```

Events are the observability backbone — they drive the live monitor, the
checkpoint (resume), the token rollup, and the OKF run wiki. Every transition
carries its `tokens` so cost is attributable per step. See
`architecture/run-state-observability.md` (state store, resume, tokens, monitoring)
and `architecture/okf-run-wiki.md` (the event log folded into a browsable wiki).

## 5. Resumability

State is event-sourced and checkpointed after every transition. On restart the
orchestrator loads the latest checkpoint (or rebuilds it by replaying the event
log) and continues from each repo/CU position. Because each skill is idempotent
and graph-backed, re-entering a state re-uses cached outputs unless inputs
changed (hash check, see semantic-graph-schema §5); APPLY uses per-CU idempotency
keys so a retried apply never duplicates a PR/branch. Token counters resume from
the checkpoint and keep accumulating across restarts. Full protocol:
`architecture/run-state-observability.md`.

## 6. Concurrency & isolation

- Repos run in parallel (cap: `maxRepos`).
- Within a repo, CUs in the same wave run in parallel (cap: `maxCUsPerRepo`),
  each in its own git worktree/branch.
- Cross-repo CUs (touching a shared contract) acquire a logical lock on the
  contract GID to serialize conflicting edits.

## 7. Failure policy

| Failure | Action |
|---------|--------|
| Stage transient error | retry with backoff (max 3) |
| Compile fail (recipe) | fall back to `strategy.fallback`; if still failing → FAILED |
| Validation regression | ROLLED_BACK; mark CU for re-plan |
| Risk ≥ blockAbove | BLOCKED; surface to human with rationale |
| Cross-repo conflict | serialize via contract lock; re-validate dependents |
| Unknown interface/dependency | typed placeholder + Gap; build integration against it (never skip); raise risk |
| INTEGRATE E2E red | re-plan or rollback; do NOT advance to COMPLETE |

## 8. Role-based execution (EXECUTE)

Within EXECUTE, each ChangeUnit is delivered by collaborating roles
(`architecture/integration-and-environments.md` §1). The orchestrator dispatches
them; they are skills, not people.

```
per ChangeUnit (after transformation-compiler produces the diff):
  developer → finish/wire the change, integrate against real or PLACEHOLDER
              interfaces, keep build green, resolve compiler/LLM assumptions
  devops    → build the service + dependent systems as Podman containers/pods,
              provision monitoring/observability
  tester    → build interface mocks (real + placeholder) and prod-trace E2E
              scenarios for the affected capabilities
  → then the CU machine continues: VALIDATED → RISK_SCORED → APPLY
```

The roles also operate at repo scope during INTEGRATE to assemble and verify the
whole system. All role outputs honor the open-question/placeholder policy:
unknowns become placeholders + gaps, never omissions.

## 9. INTEGRATE exit criteria ("the system works")

INTEGRATE passes — and the repo may reach COMPLETE — only when ALL hold:

1. The full topology (migrated service + dependents, real or mock) comes up
   **healthy under Podman** (all health checks green).
2. **E2E scenarios built from production traces pass** with behavioral
   equivalence vs. baseline.
3. **Monitoring** shows no new error classes and **SLOs** (from the Target Spec)
   are within bounds.
4. Every external dependency is satisfied by a real backend or a labeled
   placeholder mock — **no dangling integration**.
5. All remaining gaps are **non-blocking** and enumerated in the run report.

Failing any of 1–4 → re-plan or rollback. A run never reports success with red
E2E.
