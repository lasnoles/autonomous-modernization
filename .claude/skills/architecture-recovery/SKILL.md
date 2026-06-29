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

You are the **cartographer**: read L0 structure (Classes, Methods, calls,
reads/writes), write the L1 architectural layer on top. Output is **always
additive** L1 (node/edge/annotation) — **never** mutate L0, so a re-index can
invalidate and recompute you cleanly (semantic-graph-schema §5).

Authoritative contracts — read before acting:
- `architecture/semantic-graph-schema.md` — GID format, envelopes, layered fact model, provenance/confidence
- `graph/ontology.md` — write invariants (esp. §3)
- `graph/node-types.md` — L1 `Component`, `Layer`, `Seam` props
- `graph/edge-types.md` — L1 `BELONGS_TO`, `DEPENDS_ON`, `LAYERED_AS`, `VIOLATES`, `CANDIDATE_SEAM`
- `graph/cypher-queries.md` — reusable reads (add new L1 queries here)
- `architecture/scalability-and-retrieval.md` §1a — batching / bounded-context retrieval

## When to use
- Orchestrator advances a repo INDEX → RECOVER.
- "Recover architecture / components / layers / seams for repo X" or "where could a strangler-fig boundary go?"
- Re-run after a re-index invalidated L0 (incremental: recompute only affected components).

Do **not** use to parse source (query L0 instead), infer business meaning (L2
`business-discovery`), or score debt/risk (L3/L5).

## Inputs / Outputs

**In (read-only, via cypher — never re-parse source):**
- L0 structural graph for the repo: `Class`, `Method`, `Field`, `Package`,
  `Module`, `Endpoint`, `Table` nodes; `CALLS`, `EXTENDS`, `IMPLEMENTS`,
  `READS`, `WRITES`, `DECLARES`, `HANDLED_BY`, `DEPENDS_ON_LIB` edges.
- Class `props.annotations[]` (`@RestController`, `@Service`, `@Repository`,
  `@Entity`) and `Package` names as layer/cluster priors.
- Optional run config: expected layer DAG + per-repo overrides.

**Out (additive L1 only):**

| New node | When |
|----------|------|
| `Component` | one per recovered class cluster (`cohesion`, `coupling`, `responsibility`, `size`) |
| `Layer` | one per architectural layer present (`name`, `expectedDependencies[]`) |
| `Seam` | one per viable extraction boundary (`seamKind`, `extractability`, `boundaryClasses[]`) |

| New edge | from → to | meaning |
|----------|-----------|---------|
| `BELONGS_TO` | Class → Component, Class → Layer | recovered grouping (`confidence`) |
| `DEPENDS_ON` | Component → Component | inter-component coupling (`weight` = call count) |
| `LAYERED_AS` | Component → Layer | layer assignment |
| `VIOLATES` | Component/Class → Layer | layering rule broken (`rule`) |
| `CANDIDATE_SEAM` | Seam → Component | seam attaches at this boundary (`extractability`) |

Every node/edge carries the common envelope with `provenance.stage =
"architecture-recovery"` + `provenance.runId`; inferred groupings set
`confidence < 1.0` per ontology §3.

## Procedure

```
1. PULL — per module, paged; never "all Class nodes" at once (that bulk read times out large repos).
   For each Module (one at a time), page Class with SKIP/LIMIT in batches of
   execution.batchSize (scalability-and-retrieval.md §1a). Per class: package,
   annotations[], isPublicApi; method CALLS lifted to class granularity;
   EXTENDS/IMPLEMENTS; READS/WRITES → Field/Table.
   Hold only current module's projection + small cross-module DEPENDS_ON weight tallies.
   Incremental: if a node's source hash is unchanged since last run, reuse its prior cluster.

2. BUILD class dependency graph — project method CALLS onto undirected (clustering)
   + directed (layering/weights) class graphs. Edge weight = distinct call sites
   between classes + inheritance/impl + shared Table/Field access. Keep directed
   version for DEPENDS_ON weights and violation detection.

3. METRICS per class + candidate cluster:
   • Ca (afferent) = distinct classes calling in.  • Ce (efferent) = distinct classes called out.
   • Instability I = Ce/(Ca+Ce)  (0 stable … 1 unstable).
   • Relational cohesion (LCOM-style) = internal-edge density vs. boundary-crossing edges;
     maximize internal / minimize crossing.

4. CLUSTER into Components — combine signals, don't trust one:
   • Community detection (Louvain/Leiden, maximize modularity) = primary signal.
   • Package-structure prior — shared package prefix → same component (regularize toward it).
   • Naming prior — *Controller/*Service/*Repository/*Dao/*Client/*Mapper bias cluster + layer.
   Each cluster → one Component; set cohesion (internal density), coupling
   (boundary-crossing mass), size (class count), responsibility (dominant role/package).

5. INFER layers — Component → Layer ∈ {web, service, domain, persistence, integration}
   from converging evidence:
   • Annotations (strongest): @RestController/@Controller→web, @Service→service,
     @Repository/JPA/JDBC→persistence, @Entity/POJO→domain, HTTP/AMQP/Kafka clients→integration.
   • Endpoint adjacency: reachable from Endpoint HANDLED_BY → web/edge.
   • Table adjacency: WRITES/READS to Table → persistence.
   • Dependency direction: sinks (high Ca, low Ce) → domain/persistence; sources → web.
   Emit LAYERED_AS (Component→Layer) + per-class BELONGS_TO (→Layer); lower confidence where signals disagree.

6. DETECT violations — expected layer DAG (default web→service→domain→persistence;
   integration callable from service/persistence). Any DEPENDS_ON / class call against
   allowed direction (e.g. persistence→web, domain→service, skip-layer) → emit VIOLATES
   (Component/Class→Layer) with rule naming the broken constraint
   (e.g. "persistence→web", "skip-layer:web→persistence", "upward-dependency").

7. SCORE seams — for each candidate boundary (a Component boundary, or a sub-cluster
   modularity nearly split), compute extractability ∈ [0,1] (see below). Above threshold,
   emit a Seam with most plausible seamKind + boundaryClasses[] forming the cut,
   plus a CANDIDATE_SEAM edge to the Component(s) it fences.

8. EMIT L1 — write all additively with provenance.stage="architecture-recovery", runId,
   tool (clusterer + version). Inferred groupings/layers confidence < 1.0; resolved
   structural aggregations (e.g. DEPENDS_ON weight summed from resolved CALLS) may be higher.
   Never touch L0.
```

## Example reads (from `graph/cypher-queries.md` + L1 additions)

**Lift method call graph to class granularity (clustering + DEPENDS_ON weights):**
```cypher
MATCH (a:Class)-[:DECLARES]->(:Method)-[c:CALLS]->(:Method)<-[:DECLARES]-(b:Class)
WHERE a.repo = $repo AND b.repo = $repo AND a <> b
RETURN a.gid AS from, b.gid AS to, count(c) AS weight
```

**Class-level Ca/Ce → instability:**
```cypher
MATCH (c:Class {repo:$repo})
OPTIONAL MATCH (caller:Class)-[:DECLARES]->(:Method)-[:CALLS]->(:Method)<-[:DECLARES]-(c)
OPTIONAL MATCH (c)-[:DECLARES]->(:Method)-[:CALLS]->(:Method)<-[:DECLARES]-(callee:Class)
RETURN c.gid, count(DISTINCT caller) AS ca, count(DISTINCT callee) AS ce
```

Reuse the existing **"Component coupling (extraction difficulty)"** query from
`cypher-queries.md` to read back `DEPENDS_ON` weights once written, and promote
the two queries above into that file (DISCOVER/DEBT also read them).

## Seam extractability (input to the planner)

A `Seam` marks where a strangler-fig boundary can be inserted to migrate a slice
behind a shim while the rest keeps calling old code. `extractability ∈ [0,1]`
estimates how cheap/safe the cut is, combining four boundary measurements:

1. **Crossing coupling** — few, narrow edges across the cut → high; whole-system
   call-through (high crossing mass) → low.
2. **Shared-state crossings** — count READS/WRITES where both sides touch the
   **same** Field/Table; shared mutable state means a proxy can't be transparent → strong penalty.
3. **Transaction boundaries** — calls across the cut inside one transaction (e.g.
   `@Transactional` spanning both sides) → penalty (atomicity risk); a seam aligned
   with an existing txn edge scores higher.
4. **Interface readiness** — a boundary already fronted by an interface/`isPublicApi`,
   or one clean facade dominating inbound calls → bonus, and sets `seamKind`
   (`interface-facade` if interface exists, `branch-by-abstraction` if not,
   `http-proxy`/`event-bridge` for cross-process edges, `feature-flag` for runtime cutover).

`extractability = w1·(1−normCrossingCoupling) + w2·(1−sharedStatePenalty) +
w3·(1−txnSplitPenalty) + w4·interfaceReadiness`, clamped to [0,1], weights pinned per run.

**Why the planner needs it:** the L4 planner sizes/orders ChangeUnits and picks
where to land changes behind a seam (`GUARDS` edge). High extractability = cheap,
low-blast-radius place to strangle first; low = entangled (shared state/txns),
sequence later or decompose. The score feeds ordering + risk, so it must be
explainable and reproducible.

## Graph writes (exact L1 mapping)

| Produced | Type | Key props |
|----------|------|-----------|
| Class cluster | `Component` node | `cohesion`, `coupling`, `responsibility`, `size` |
| Architectural layer | `Layer` node | `name`, `expectedDependencies[]` |
| Extraction boundary | `Seam` node | `seamKind`, `extractability` (0–1), `boundaryClasses[]` |
| Class → its component | `BELONGS_TO` (Class→Component) | `confidence` |
| Class → its layer | `BELONGS_TO` (Class→Layer) | `confidence` |
| Component coupling | `DEPENDS_ON` | `weight` (call count) |
| Component → layer | `LAYERED_AS` | — |
| Illegal dependency | `VIOLATES` (Component/Class→Layer) | `rule` |
| Seam → fenced component | `CANDIDATE_SEAM` | `extractability` |

Edges with `from.repo != to.repo` set `props.crossRepo = true` and name the
shared `contract` GID (ontology §3.5 / edge-types Cross-repo).

## Quality / invariants
- **L0 is immutable** — only add L1; if an L0 fact looks wrong, surface it (indexer bug), don't patch it.
- **Partition:** every Class belongs to exactly one Component and one Layer (`BELONGS_TO` is not overlapping). Unassignable classes → residual component, low `cohesion`, never dropped.
- **DEPENDS_ON weights are derived, not invented** — each weight = summed CALLS/inheritance/shared-access mass between components; reconstructable from L0.
- **Confidence reflects evidence** — inferred clusters/layers `confidence < 1.0` (ontology §3.3); record `provenance.tool`.
- **Determinism** — clusterer, resolution/threshold params, seam weights pinned per `runId` → re-run yields identical L1 (system-design §6).
- **No orphans** — every `LAYERED_AS`/`VIOLATES` targets an existing `Layer`; every `CANDIDATE_SEAM` targets a real `Component`.
- **Defensive to unknowns** — ignore unrecognized node/edge kinds (forward-compat, ontology §5) rather than failing.

## Definition of done

Every `Class` assigned to exactly one `Component` and one `Layer` via
`BELONGS_TO`; inter-component `DEPENDS_ON` edges carry derived `weight`s;
layering `VIOLATES` edges flagged with a named `rule`; at least the viable
extraction boundaries emitted as `Seam` nodes with explainable `extractability`
+ `CANDIDATE_SEAM` edge; all outputs additive L1 with
`provenance.stage="architecture-recovery"`; run is re-runnable to an identical
graph; L0 untouched. Orchestrator can advance to DISCOVER.
