# Semantic Graph Schema

The semantic graph is the single source of truth shared by every skill.
This file defines the **contract**; `graph/ontology.md`, `graph/node-types.md`,
and `graph/edge-types.md` hold the exhaustive enumerations and
`graph/cypher-queries.md` holds reusable queries.

## 1. Identity & addressing

Every node has a stable, content-independent **GID** (graph id):

```
gid = "<repo>::<kind>::<fqn>[#<disambiguator>]"
# examples
billing-svc::Class::com.acme.billing.InvoiceService
billing-svc::Method::com.acme.billing.InvoiceService#charge(java.math.BigDecimal)
billing-svc::Endpoint::POST /api/v1/invoices
shared::Table::INVOICE
```

GIDs are repo-prefixed so the graph can span repositories while keeping
collisions impossible. Cross-repo edges (e.g. an HTTP call from
`web-app` to `billing-svc`) are first-class.

## 2. Node envelope (common fields)

Every node, regardless of `kind`, carries:

```jsonc
{
  "gid": "billing-svc::Method::...#charge(...)",
  "kind": "Method",                 // see graph/node-types.md
  "repo": "billing-svc",
  "fqn": "com.acme.billing.InvoiceService.charge",
  "name": "charge",
  "lang": "java",
  "loc": { "file": "src/.../InvoiceService.java", "startLine": 42, "endLine": 88 },
  "props": { /* kind-specific, see node-types.md */ },
  "provenance": {
    "stage": "semantic-indexer",
    "tool": "tree-sitter-java@0.21 + custom resolver",
    "runId": "run_2026_06_28_001",
    "hash": "sha256:..."           // hash of source span, for change detection
  }
}
```

## 3. Edge envelope (common fields)

```jsonc
{
  "type": "CALLS",                  // see graph/edge-types.md
  "from": "billing-svc::Method::...#charge(...)",
  "to":   "billing-svc::Method::...#persist(...)",
  "props": { "callSite": 57, "dynamic": false, "confidence": 1.0 },
  "provenance": { "stage": "semantic-indexer", "runId": "run_..." }
}
```

`confidence ∈ [0,1]` lets later stages distinguish resolved facts (1.0)
from inferred ones (e.g. reflection, DI, LLM-derived business edges).

## 4. Layered fact model

The graph accumulates facts in layers; each stage adds its layer and never
mutates a prior stage's nodes (it adds edges/annotations instead).

| Layer | Added by | Representative nodes / edges |
|-------|----------|------------------------------|
| L0 Structural | semantic-indexer | Repo, Module, Package, Class, Method, Field, Endpoint, Table; `CALLS`, `EXTENDS`, `IMPLEMENTS`, `READS`, `WRITES`, `DECLARES` |
| L1 Architectural | architecture-recovery | Component, Layer, Seam; `BELONGS_TO`, `DEPENDS_ON`, `VIOLATES` |
| L2 Business | business-discovery | Capability, BusinessRule, DomainEntity; `IMPLEMENTS_RULE`, `REALIZES_CAPABILITY` |
| L3 Debt | technical-debt | DebtItem, Vulnerability, Hotspot; `HAS_DEBT`, `DEPENDS_ON_DEPRECATED` |
| L4 Plan | planner | ChangeUnit, Seam (refined); `TRANSFORMS`, `PRECEDES`, `GUARDS` |
| L5 Risk | risk-assessment | RiskScore; `HAS_RISK` |

## 5. Change detection

On re-index, a node whose source-span `hash` is unchanged is reused; a
changed hash invalidates the node **and** all derived facts that reference
it (via provenance back-links), so downstream stages recompute only the
affected subgraph. This makes the whole pipeline incremental.

## 6. Querying contract

Skills MUST NOT re-parse source to answer structural questions; they query
the graph. Canonical reusable queries live in `graph/cypher-queries.md`
(e.g. "transitive callers of method X", "endpoints touching table T",
"classes in component C with open debt"). Skills add their own queries but
should promote broadly useful ones into that file.

## 7. Serialization

The graph is exchanged between stages as either:
- **Live**: a Neo4j/Bolt connection (production), or
- **Portable**: newline-delimited JSON (`nodes.ndjson`, `edges.ndjson`) in
  the artifact store (CI / offline / replay).

Both honor the envelopes above. The portable form is what `replay` snapshots.
