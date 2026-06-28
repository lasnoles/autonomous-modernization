# Semantic Graph Ontology

The ontology is the controlled vocabulary for the semantic graph. It binds
the layered fact model from `architecture/semantic-graph-schema.md` to
concrete node and edge types. `node-types.md` and `edge-types.md` enumerate
each; this file gives the conceptual model and the rules that hold them
together.

## 1. Layers (recap)

| Layer | Theme | Produced by |
|-------|-------|-------------|
| L0 | Structure (what the code *is*) | semantic-indexer |
| L1 | Architecture (how it's *organized*) | architecture-recovery |
| L2 | Business (what it *does* for users) | business-discovery |
| L3 | Debt (what's *wrong*) | technical-debt |
| L4 | Plan (what we'll *change*) | planner |
| L5 | Risk (how *dangerous* the change is) | risk-assessment |

## 2. Core entity relationships (conceptual)

```
Repo ─contains─▶ Module ─contains─▶ Package ─contains─▶ Class
Class ─declares─▶ Method ─calls─▶ Method
Class ─declares─▶ Field
Method ─reads/writes─▶ Field | Table
Endpoint ─handledBy─▶ Method
Method ─realizes─▶ BusinessRule ─partOf─▶ Capability
Component ─groups─▶ Class            (L1)
Component ─dependsOn─▶ Component      (L1)
DebtItem ─affects─▶ Class | Method   (L3)
ChangeUnit ─transforms─▶ {Class|Method|Package|Component}  (L4)
ChangeUnit ─precedes─▶ ChangeUnit    (L4)
RiskScore ─scores─▶ ChangeUnit       (L5)
```

## 3. Ontology rules (invariants enforced on write)

1. **Containment is a tree** within L0: a node has at most one
   `BELONGS_TO`/containment parent of the next-coarser kind.
2. **Cross-layer references point downward in age**: L4 ChangeUnits
   reference L0–L3 nodes, never the reverse.
3. **Business edges carry confidence**: L2 edges derived by inference/LLM
   set `confidence < 1.0` and record `provenance.tool`.
4. **No orphan debt/risk**: every L3 DebtItem and L5 RiskScore must attach to
   at least one concrete L0/L4 node.
5. **Cross-repo edges are explicit**: an edge whose `from.repo != to.repo`
   must set `props.crossRepo = true` and name the shared `contract` GID.

## 4. Naming & typing conventions

- Node `kind` values are PascalCase singular (`Class`, `BusinessRule`).
- Edge `type` values are SCREAMING_SNAKE verbs (`CALLS`, `DEPENDS_ON`).
- `fqn` is language-native fully-qualified name; `gid` is the addressable id
  (see semantic-graph-schema §1).

## 5. Extensibility

New node/edge types are added by appending to `node-types.md` / `edge-types.md`
with: definition, owning layer, required props, and at least one example.
Skills must treat unknown types defensively (ignore, don't fail) to allow
forward-compatible graph evolution.
