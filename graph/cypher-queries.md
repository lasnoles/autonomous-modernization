# Reusable Cypher Queries

Canonical, reviewed queries skills should reuse instead of re-parsing source.
Parameters are `$snake_case`. Neo4j/openCypher syntax; the portable NDJSON
form supports the same traversals via the graph client.

## Structure (L0)

**Transitive callers of a method (impact set):**
```cypher
MATCH (caller:Method)-[:CALLS*1..$depth]->(m:Method {gid:$gid})
RETURN DISTINCT caller.gid, caller.fqn
```

**Endpoints that ultimately touch a table (data lineage):**
```cypher
MATCH (e:Endpoint)-[:HANDLED_BY]->(:Method)-[:CALLS*0..]->(:Method)
      -[:READS|WRITES]->(t:Table {gid:$table_gid})
RETURN DISTINCT e.gid, e.path
```

**Public API surface of a module:**
```cypher
MATCH (mod:Module {gid:$module_gid})-[:CONTAINS*]->(c:Class {isPublicApi:true})
RETURN c.gid, c.fqn
```

**Dead code candidates (no inbound calls, not an entry point):**
```cypher
MATCH (m:Method) WHERE NOT (m)<-[:CALLS]-() AND m.isEntryPoint = false
  AND NOT (:Endpoint)-[:HANDLED_BY]->(m)
RETURN m.gid
```

## Entry points & flows (L0 — see entry-point-discovery.md)

**All entry points, grouped by class (the full inventory):**
```cypher
MATCH (e:Endpoint) WHERE coalesce(e.synthetic,false) = false
RETURN e.entryClass AS class, count(*) AS n, collect(e.gid)[..50] AS examples
ORDER BY n DESC
```

**Flow trace from an entry point (methods, data, externals):**
```cypher
MATCH (f:Flow)-[:ENTERS_AT]->(:Endpoint {gid:$entry_gid})
OPTIONAL MATCH (f)-[:FLOWS_THROUGH]->(m:Method)
OPTIONAL MATCH (f)-[t:TOUCHES]->(x)
RETURN f.gid, collect(DISTINCT m.gid) AS methods,
       collect(DISTINCT {target:x.gid, op:t.op}) AS touches
```

**Entry points with NO flow (discovery gaps — must be investigated):**
```cypher
MATCH (e:Endpoint) WHERE coalesce(e.synthetic,false) = false
  AND NOT (:Flow)-[:ENTERS_AT]->(e)
RETURN e.gid, e.entryClass
```

**Methods unreachable from any (non-synthetic) entry point:**
```cypher
MATCH (m:Method)
WHERE NOT EXISTS {
  MATCH (f:Flow)-[:FLOWS_THROUGH]->(m) WHERE coalesce(f.synthetic,false)=false
}
RETURN m.gid
```

**Entry points / flows touching a table or external system:**
```cypher
MATCH (f:Flow)-[:TOUCHES]->(x {gid:$target_gid})
MATCH (f)-[:ENTERS_AT]->(e:Endpoint)
RETURN DISTINCT e.gid, e.entryClass, f.gid
```

## Architecture (L1)

**Layering violations:**
```cypher
MATCH (c:Component)-[v:VIOLATES]->(l:Layer)
RETURN c.gid, l.name, v.rule ORDER BY c.gid
```

**Component coupling (extraction difficulty):**
```cypher
MATCH (a:Component {gid:$comp})-[d:DEPENDS_ON]-(b:Component)
RETURN b.gid, d.weight ORDER BY d.weight DESC
```

## Business (L2)

**Business rules a change would touch (behavior to preserve):**
```cypher
MATCH (cu:ChangeUnit {gid:$cu})-[:TRANSFORMS]->(:Class)-[:DECLARES]->(m:Method)
      -[:IMPLEMENTS_RULE]->(r:BusinessRule)
RETURN DISTINCT r.gid, r.statement, r.confidence
```

**Capability footprint (classes realizing a capability):**
```cypher
MATCH (n)-[:REALIZES_CAPABILITY]->(cap:Capability {gid:$cap})
RETURN n.gid, labels(n)
```

## Debt (L3)

**Open debt within a component, ranked:**
```cypher
MATCH (c:Component {gid:$comp})<-[:BELONGS_TO]-(:Class)-[:HAS_DEBT]->(d:DebtItem)
RETURN d.gid, d.category, d.severity ORDER BY d.severity DESC
```

**Deprecated-API usage to migrate:**
```cypher
MATCH (m:Method)-[r:DEPENDS_ON_DEPRECATED]->(dep)
RETURN m.gid, dep.gid, r.replacement
```

## Plan & risk (L4/L5)

**Wave-ordered, unblocked ChangeUnits ready to execute:**
```cypher
MATCH (cu:ChangeUnit {status:'PLANNED'})-[:IN_WAVE]->(w:Wave)
WHERE NOT (cu)-[:PRECEDES]->(:ChangeUnit {status:'PLANNED'})  // deps satisfied
RETURN cu.gid, w.order ORDER BY w.order
```

**Blast radius of a ChangeUnit:**
```cypher
MATCH (cu:ChangeUnit {gid:$cu})-[:TRANSFORMS]->(target)
OPTIONAL MATCH (caller:Method)-[:CALLS*1..2]->(:Method)<-[:DECLARES]-(target)
RETURN count(DISTINCT target) AS targets, count(DISTINCT caller) AS callers
```

**Cross-repo contracts a ChangeUnit affects (locking):**
```cypher
MATCH (cu:ChangeUnit {gid:$cu})-[:TRANSFORMS]->(t)-[e]-(o)
WHERE e.crossRepo = true
RETURN DISTINCT e.contract
```

## Conventions

- Always `RETURN` GIDs (stable) plus a human label.
- Parameterize repo/scope; never inline GIDs.
- Promote any query reused by ≥2 skills into this file with a heading.
