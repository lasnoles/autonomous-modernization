# System Design — Autonomous Modernization Autopilot

## 1. Purpose

An autonomous system that modernizes legacy Java codebases across many
repositories. It ingests source, builds a semantic understanding of each
system, discovers the business behavior worth preserving, plans a safe
sequence of changes, compiles those changes into deterministic
transformations (with an LLM fallback), validates behavioral equivalence,
and applies them under a strangler-fig pattern with automatic rollback.

The system is **agentic but gated**: every irreversible action passes a risk
gate and a validation gate before it touches a real branch.

## 2. Design principles

1. **Deterministic-first, LLM-fallback.** Prefer AST-level recipes
   (OpenRewrite, ts-morph, Roslyn) that are reproducible and reviewable.
   Use the LLM only when no recipe exists, and always validate its output.
2. **Behavior-preserving by default.** Every transformation must prove
   behavioral equivalence (tests, golden replay, characterization tests)
   unless the plan explicitly authorizes a behavior change.
3. **Everything is a graph fact.** All knowledge — structure, business
   rules, debt, risk — is materialized as nodes/edges in the semantic
   graph so downstream skills query rather than re-parse.
4. **Idempotent, resumable stages.** Each stage reads inputs from the
   graph + artifact store and writes outputs back; re-running a stage is
   safe and cache-aware.
5. **Per-repo isolation, cross-repo awareness.** Each repo is transformed
   in its own worktree/branch, but the graph spans repos so cross-repo
   contracts (APIs, events, schemas) are respected.
6. **Human-in-the-loop optional, not required.** The orchestrator can run
   fully autonomously up to a configured risk ceiling; above it, it pauses
   for approval.

## 3. Component overview

```
                         ┌─────────────────────────────┐
                         │        ORCHESTRATOR          │
                         │  (state machine + scheduler) │
                         └──────────────┬──────────────┘
                                        │ drives
        ┌───────────────┬──────────────┼──────────────┬───────────────┐
        ▼               ▼              ▼               ▼               ▼
 ┌────────────┐  ┌────────────┐  ┌────────────┐ ┌────────────┐  ┌────────────┐
 │  semantic  │  │architecture│  │  business  │ │ technical  │  │  planner   │
 │  indexer   │─▶│  recovery  │─▶│ discovery  │ │   debt     │─▶│            │
 └─────┬──────┘  └────────────┘  └────────────┘ └────────────┘  └─────┬──────┘
       │ writes facts          (all read+write the semantic graph)     │ emits IR
       ▼                                                               ▼
 ┌────────────────────────── SEMANTIC GRAPH ──────────────────┐ ┌────────────┐
 │  nodes + edges  (see graph/ontology.md)                     │ │transformat.│
 └────────────────────────────────────────────────────────────┘ │ compiler   │
                                                                  └─────┬──────┘
        ┌───────────────────────────────────────────────────────────────┘
        ▼               ▼               ▼
 ┌────────────┐  ┌────────────┐  ┌────────────┐
 │ validation │  │   replay   │  │   risk     │
 │            │◀▶│            │  │ assessment │
 └────────────┘  └────────────┘  └────────────┘
```

## 4. Data planes

| Plane | Store | Owner | Contents |
|-------|-------|-------|----------|
| Semantic graph | Graph DB (Neo4j / in-mem) | indexer + all | structure, business, debt, risk facts |
| Artifact store | Object store / filesystem | all stages | IR docs, diffs, recipes, reports, replay traces |
| State store | Orchestrator | orchestrator | per-repo state machine position, run log |
| Source | Git (per-repo worktrees) | compiler/validation | working branches, strangler shims |

## 5. End-to-end flow (happy path)

1. **Index** each repo → semantic graph (AST, symbols, calls, data flow).
2. **Recover** architecture → modules, layers, components, seams.
3. **Discover** business capabilities & rules → business nodes.
4. **Assess debt** → debt nodes scored & linked to code.
5. **Plan** → Modernization IR: ordered, scoped change-units with seams.
6. **Compile** each change-unit → concrete transformation (recipe or LLM patch).
7. **Validate** → build + tests + behavioral equivalence.
8. **Replay** (where tests are thin) → capture/replay characterization.
9. **Risk-gate** → score; below ceiling auto-apply, above pause.
10. **Apply** under strangler-fig; **rollback** on regression.

## 6. Cross-cutting concerns

- **Provenance:** every node/edge and every diff records the stage, model,
  recipe version, and inputs that produced it.
- **Determinism:** recipe versions and LLM prompt/template versions are
  pinned per run for reproducibility (see `replay` skill).
- **Safety:** no force-push, no direct writes to default branches, all
  changes via PRs on isolated branches.
- **Observability:** the orchestration state machine emits structured
  events consumed by a run dashboard.

See companion docs:
`semantic-graph-schema.md`, `modernization-ir.md`,
`orchestration-state-machine.md`, `deployment-architecture.md`,
and `graph/ontology.md`.
