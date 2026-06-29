---
name: architecture-recovery
description: >-
  RECOVER stage of the modernization autopilot. Reads the L0 structural graph
  produced by semantic-indexer and recovers the L1 architecture: clusters
  classes into Components by cohesion/coupling, assigns each to a Layer
  (web → service → domain → persistence → integration), draws inter-component
  DEPENDS_ON edges weighted by call volume, flags layering VIOLATES edges
  against expected layer rules, and proposes candidate Seam nodes with an
  extractability score that marks where strangler-fig boundaries can go.
  It only ever ADDS L1 nodes/edges/annotations — it never mutates L0. Use
  this after INDEX and before DISCOVER/DEBT/PLAN.
---

# Architecture Recovery — L1 Reconstruction

You are the **cartographer**. The indexer told us what the code *is* (L0:
Classes, Methods, calls, reads/writes); you decide how it's *organized* and
where it can be safely cut. You read structure and write the architectural
layer (L1) on top of it. You **never** modify L0 nodes — every output is an
additive L1 node, edge, or annotation, so a re-index can invalidate and
recompute you cleanly (semantic-graph-schema §5).

Read these contracts before acting and treat them as authoritative:

- `architecture/semantic-graph-schema.md` — GID format, envelopes, layered fact model, provenance/confidence
- `graph/ontology.md` — invariants enforced on write (esp. §3)
- `graph/node-types.md` — L1 `Component`, `Layer`, `Seam` props
- `graph/edge-types.md` — L1 `BELONGS_TO`, `DEPENDS_ON`, `LAYERED_AS`, `VIOLATES`, `CANDIDATE_SEAM`
- `graph/cypher-queries.md` — reusable reads (you add new L1 queries here)

## When to use

- The orchestrator advances a repo from INDEX to RECOVER (state machine).
- "Recover the architecture / components / layers / seams for repo X."
- "Where could we put a strangler-fig boundary in this codebase?"
- Re-run after a re-index invalidated L0 nodes (incremental: recompute only
  the affected components, not the whole graph).

Do **not** use it to parse source (query L0 instead), to infer business
meaning (that's L2 `business-discovery`), or to score debt/risk (L3/L5).

## Inputs / Outputs

**Inputs (read-only):**

- The **L0 structural graph** for the repo: `Class`, `Method`, `Field`,
  `Package`, `Module`, `Endpoint`, `Table` nodes and `CALLS`, `EXTENDS`,
  `IMPLEMENTS`, `READS`, `WRITES`, `DECLARES`, `HANDLED_BY`, `DEPENDS_ON_LIB`
  edges. Pulled via cypher — **never** by re-parsing source.
- Class `props.annotations[]` (e.g. `@RestController`, `@Service`,
  `@Repository`, `@Entity`) and `Package` names as layer/cluster priors.
- Optional run config: the expected layer DAG and any per-repo overrides.

**Outputs (additive L1 only):**

| New node | When |
|----------|------|
| `Component` | one per recovered class cluster (`cohesion`, `coupling`, `responsibility`, `size`) |
| `Layer` | one per architectural layer present (`name`, `expectedDependencies[]`) |
| `Seam` | one per viable extraction boundary (`seamKind`, `extractability`, `boundaryClasses[]`) |

| New edge | from → to | meaning |
|----------|-----------|---------|
| `BELONGS_TO` | Class → Component, Class → Layer | recovered grouping (`confidence`) |
| `DEPENDS_ON` | Component → Component | inter-component coupling (`weight` = call count) |
| `LAYERED_AS` | Component → Layer | component's layer assignment |
| `VIOLATES` | Component/Class → Layer | layering rule broken (`rule`) |
| `CANDIDATE_SEAM` | Seam → Component | seam attaches at this boundary (`extractability`) |

Every node/edge carries the common envelope with `provenance.stage =
"architecture-recovery"` and `provenance.runId`; recovered (inferred)
groupings set `confidence < 1.0` per ontology §3.

## Procedure

1. **Pull the structural graph — per module, paged, not the whole repo.** Load the
   L0 projection with cypher (see *Example reads*) **one `Module` at a time**, and
   within a module page `Class` nodes with `SKIP`/`LIMIT` in batches of
   `execution.batchSize` (`architecture/scalability-and-retrieval.md` §1a): each
   class's `package`, `annotations[]`, `isPublicApi`; the method-level `CALLS`
   graph lifted to class granularity; `EXTENDS`/`IMPLEMENTS`; `READS`/`WRITES` to
   `Field`/`Table`. **Do not load "all Class nodes" at once** — that is the bulk
   read that times out large repos. Hold only the current module's projection
   (plus the cross-module `DEPENDS_ON` weight tallies, which are small aggregates,
   not node lists). Do not parse source. If a node's source `hash` is unchanged
   since the last run, reuse its prior cluster assignment (incremental).

2. **Build the class dependency graph.** Project method `CALLS` onto an
   undirected (for clustering) and a directed (for layering/weights) class
   graph. Edge weight = number of distinct call sites between the two classes
   plus inheritance/impl and shared `Table`/`Field` access. Keep the directed
   version for `DEPENDS_ON` weights and violation detection.

3. **Compute coupling/cohesion metrics** per class and per candidate cluster:
   - **Afferent coupling (Ca):** distinct classes that call into it.
   - **Efferent coupling (Ce):** distinct classes it calls out to.
   - **Instability** `I = Ce / (Ca + Ce)` (0 = stable, 1 = unstable).
   - **Relational cohesion (LCOM-style):** internal-edge density of the
     cluster vs. edges crossing its boundary; high internal / low crossing
     is what we want a Component to maximize.

4. **Cluster classes into Components.** Combine signals, don't trust one:
   - **Community detection** on the weighted class graph (Louvain/Leiden,
     maximizing modularity) is the primary signal — it finds groups that
     call each other far more than they call outside.
   - **Package-structure prior:** classes sharing a package prefix lean
     toward the same component (regularize the modularity result toward it;
     it usually encodes the original author's intent).
   - **Naming prior:** suffix/role conventions (`*Controller`, `*Service`,
     `*Repository`, `*Dao`, `*Client`, `*Mapper`) bias both clustering and
     layer inference.
   - Each cluster becomes one `Component`. Set `cohesion` (internal density),
     `coupling` (boundary-crossing edge mass), `size` (class count), and a
     short `responsibility` summarizing its dominant role/package.

5. **Infer layers.** Assign each Component a `Layer ∈ {web, service, domain,
   persistence, integration}` from converging evidence:
   - **Annotations** (strongest): `@RestController`/`@Controller` → web,
     `@Service` → service, `@Repository`/JPA/JDBC usage → persistence,
     `@Entity`/POJO domain types → domain, HTTP/AMQP/Kafka clients →
     integration.
   - **Endpoint adjacency:** components reachable directly from `Endpoint`
     `HANDLED_BY` are web/edge.
   - **Table adjacency:** components with `WRITES`/`READS` to `Table` are
     persistence.
   - **Dependency direction:** layers flow one way; sinks (high Ca, low Ce)
     trend toward domain/persistence, sources toward web.
   Emit `LAYERED_AS` (Component → Layer) and per-class `BELONGS_TO` (→ Layer);
   set `confidence` lower where signals disagree.

6. **Detect layering violations.** Define the expected layer DAG (default:
   `web → service → domain → persistence`, `integration` callable from
   service/persistence). For every `DEPENDS_ON` (or class-level call) that
   goes **against** the allowed direction — e.g. persistence → web, domain →
   service, or a skip-layer call — emit a `VIOLATES` (Component/Class → Layer)
   with `rule` naming the broken constraint (e.g. `"persistence→web"`,
   `"skip-layer:web→persistence"`, `"upward-dependency"`).

7. **Score seam extractability.** For each candidate boundary (typically a
   Component boundary, or a sub-cluster the modularity step nearly split),
   compute an `extractability ∈ [0,1]` — *how cleanly this can be carved out
   behind a strangler-fig shim*. Higher is easier (see *Seam extractability*
   below). Above a threshold, emit a `Seam` with the most plausible
   `seamKind` and the `boundaryClasses[]` that form the cut, plus a
   `CANDIDATE_SEAM` edge to the Component(s) it fences.

8. **Emit L1 with provenance/confidence.** Write all nodes/edges additively
   with `provenance.stage="architecture-recovery"`, `runId`, and `tool`
   (clusterer + version). Inferred groupings/layers carry `confidence < 1.0`;
   resolved structural aggregations (e.g. a `DEPENDS_ON` weight summed from
   resolved `CALLS`) may carry higher confidence. Never touch L0 nodes.

## Example reads (from `graph/cypher-queries.md` + L1 additions)

**Lift the method call graph to class granularity (clustering + DEPENDS_ON weights):**

```cypher
MATCH (a:Class)-[:DECLARES]->(:Method)-[c:CALLS]->(:Method)<-[:DECLARES]-(b:Class)
WHERE a.repo = $repo AND b.repo = $repo AND a <> b
RETURN a.gid AS from, b.gid AS to, count(c) AS weight
```

**Class-level afferent/efferent coupling (Ca/Ce → instability):**

```cypher
MATCH (c:Class {repo:$repo})
OPTIONAL MATCH (caller:Class)-[:DECLARES]->(:Method)-[:CALLS]->(:Method)<-[:DECLARES]-(c)
OPTIONAL MATCH (c)-[:DECLARES]->(:Method)-[:CALLS]->(:Method)<-[:DECLARES]-(callee:Class)
RETURN c.gid,
       count(DISTINCT caller) AS ca,
       count(DISTINCT callee) AS ce
```

Reuse the existing **"Component coupling (extraction difficulty)"** query
from `cypher-queries.md` to read back `DEPENDS_ON` weights once written, and
promote the two queries above into that file (per its conventions, since
DISCOVER/DEBT also read them).

## How seam extractability is computed (and why the planner needs it)

A `Seam` marks where a strangler-fig boundary can be inserted so a slice can
be migrated behind a shim while the rest of the system keeps calling the old
code. `extractability ∈ [0,1]` estimates how cheap and safe that cut is.
We combine four boundary measurements:

1. **Afferent/efferent coupling at the boundary.** Few, narrow edges crossing
   the cut → high score. A boundary the whole system calls through (high
   crossing edge mass) is expensive to wrap → low score.
2. **Shared-state crossings.** Count `READS`/`WRITES` where caller and callee
   of the cut touch the **same** `Field` or `Table`. Shared mutable state
   across a seam means a proxy can't be transparent → strong penalty.
3. **Transaction boundaries.** If calls across the cut sit inside one
   transaction (e.g. `@Transactional` spanning both sides), splitting risks
   atomicity → penalty; a seam that aligns with an existing transaction edge
   scores higher.
4. **Interface readiness.** A boundary already fronted by an interface /
   `isPublicApi` surface, or where one clean facade dominates inbound calls,
   is near-trivial to wrap → bonus, and sets `seamKind` (`interface-facade`
   when an interface exists, `branch-by-abstraction` when not yet,
   `http-proxy`/`event-bridge` for cross-process edges, `feature-flag` for
   runtime cutover).

Roughly: `extractability = w1·(1−normCrossingCoupling) +
w2·(1−sharedStatePenalty) + w3·(1−txnSplitPenalty) + w4·interfaceReadiness`,
clamped to `[0,1]` with weights pinned per run for determinism.

**Why the planner needs it:** the planner (L4) sizes and orders `ChangeUnit`s
and picks where to land changes behind a seam (`GUARDS` edge). A high-
extractability `Seam` is a cheap, low-blast-radius place to strangle first;
a low one warns the planner that a slice is entangled (shared state /
transactions) and must be sequenced later or decomposed. The score is an
input to ordering and risk, so it must be explainable and reproducible.

## Graph writes (exact L1 mapping)

| Produced | Type | Key props |
|----------|------|-----------|
| Class cluster | `Component` node | `cohesion`, `coupling`, `responsibility`, `size` |
| Architectural layer | `Layer` node | `name`, `expectedDependencies[]` |
| Extraction boundary | `Seam` node | `seamKind`, `extractability` (0–1), `boundaryClasses[]` |
| Class → its component | `BELONGS_TO` edge (Class→Component) | `confidence` |
| Class → its layer | `BELONGS_TO` edge (Class→Layer) | `confidence` |
| Component → component coupling | `DEPENDS_ON` edge | `weight` (call count) |
| Component → layer | `LAYERED_AS` edge | — |
| Illegal dependency | `VIOLATES` edge (Component/Class→Layer) | `rule` |
| Seam → fenced component | `CANDIDATE_SEAM` edge | `extractability` |

All edges with `from.repo != to.repo` set `props.crossRepo = true` and name
the shared `contract` GID (ontology §3.5 / edge-types Cross-repo).

## Quality / invariants

- **L0 is immutable.** Only add L1 nodes/edges/annotations; if you'd need to
  change an L0 fact, the indexer is wrong — surface it, don't patch it.
- **Every Class belongs to exactly one Component and one Layer** (`BELONGS_TO`
  is a partition, not overlapping). Unassignable classes go to a residual
  component with low `cohesion`, never dropped.
- **`DEPENDS_ON` weights are derived, not invented:** each weight must equal
  the summed `CALLS`/inheritance/shared-access mass between the components, so
  the edge is reconstructable from L0.
- **Confidence reflects evidence:** inferred clusters/layers set
  `confidence < 1.0` (ontology §3.3); record `provenance.tool`.
- **Determinism:** clusterer, resolution/threshold params, and seam weights
  are pinned per `runId` so a re-run yields the same L1 (system-design §6).
- **No orphans:** every `LAYERED_AS`/`VIOLATES` targets a `Layer` node that
  exists; every `CANDIDATE_SEAM` targets a real `Component`.
- **Defensive to unknowns:** ignore node/edge kinds you don't recognize
  (forward-compat, ontology §5) rather than failing.

## Definition of done

Every `Class` in the repo is assigned to exactly one `Component` and one
`Layer` via `BELONGS_TO`; inter-component `DEPENDS_ON` edges carry derived
`weight`s; layering `VIOLATES` edges are flagged with a named `rule`; at
least the viable extraction boundaries are emitted as `Seam` nodes with an
explainable `extractability` and a `CANDIDATE_SEAM` edge; all outputs are
additive L1 with `provenance.stage="architecture-recovery"`, the run is
re-runnable to an identical graph, and L0 is untouched. The orchestrator can
now advance the repo to DISCOVER.
