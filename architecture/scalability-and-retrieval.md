# Scalability & the Retrieval Contract (huge repos without re-reading)

The autopilot must work on repos far larger than any single context window. The
governing idea: **read source once, at INDEX; thereafter read the *graph*, and
when you do need source, fetch only the exact span by GID — never re-open whole
files.** This document makes that a hard contract and adds the mechanisms that
keep cost bounded as the repo grows.

Foundations already in place: the graph is the index; nodes carry
`loc{file,startLine,endLine}` + a source-span `hash`; re-index reuses unchanged
spans and invalidates only dependent derived facts
(`semantic-graph-schema.md` §5); skills "MUST NOT re-parse source for structural
questions" (§6). This doc generalizes that into a uniform retrieval discipline.

## 1. The Retrieval Contract (binds every skill)

1. **INDEX is the only full read.** `semantic-indexer` streams each file once,
   chunked per declaration, and never holds a whole repo in memory. Every other
   stage is forbidden from reading whole files.
2. **Resolve, then fetch a span.** To inspect code, a skill resolves a node's
   `gid → loc`, and reads **only `[startLine, endLine]`** (plus a small, fixed
   context margin). The compiler/developer already do this ("exact source spans
   for `targets[]`, not whole files"); it is now the rule for *all* evidence reads
   (business-discovery, technical-debt, validation, reviewer, …).
3. **Query the graph for structure.** Callers, flows, dependencies, debt, rules —
   all come from graph queries (`graph/cypher-queries.md`), never from re-walking
   source.
4. **Hash-gated re-reads.** A span is re-read only if its `hash` changed. Cache
   spans by `(gid, hash)`; an unchanged span is never fetched twice in a run.
5. **Declare a read budget.** Each stage runs under a `maxSpans` / `maxBytes`
   budget (run config). Exceeding it is a signal to *subdivide* the work (per
   component/flow), not to slurp more — and it's logged (no silent truncation).

A skill that wants to "just read the file" is doing it wrong: it asks the graph
which spans matter and reads those.

## 2. Indexing at scale — shard, parallelize, incrementalize

- **Partition by module/package.** The graph is built (and can be queried)
  per-module; INDEX fans out across modules in parallel
  (`deployment-architecture.md` worker plane). A monorepo becomes N bounded
  indexing jobs, not one giant pass.
- **Incremental by hash.** On re-index only changed spans are re-parsed; their
  dependent derived facts (via provenance back-links) are invalidated and only
  the affected subgraph is recomputed. Steady-state cost tracks the size of the
  *diff*, not the repo.
- **Per-shard snapshots.** Portable `nodes.ndjson`/`edges.ndjson` are written per
  partition so they can be loaded, cached, and diffed independently.

## 3. Lazy, scoped subgraph loading

No stage loads "the graph." Each stage issues a **scoped query** and receives
only the subgraph it needs:

- architecture-recovery loads the call/dependency edges of one module at a time.
- A ChangeUnit's compiler/validator loads only `targets[]` + their 1–2-hop
  neighborhood (the blast-radius subgraph), not the repo graph.
- risk/planner read aggregate rollups, not leaf nodes.

For a live Neo4j backend this is ordinary parameterized Cypher; for the portable
NDJSON form, the graph client streams and filters rather than loading all rows.

## 4. Hierarchical summaries (graph-level progressive disclosure)

Coarse reasoning shouldn't pay for leaf detail. The graph carries **summary
nodes** at Component/Module level (responsibility, public surface, debt rollup,
flow count) so the planner/architect can reason over a repo's shape from a few
hundred summary nodes, drilling into leaf Methods only where a ChangeUnit
actually edits. This mirrors OKF `index.md` progressive disclosure, but in the
graph: summarize down, link to detail, fetch detail on demand.

## 5. Bounded per-stage context = the unit of execution

Each stage/skill runs as its **own subagent with a bounded context**: its
SKILL.md + the specific contracts it cites + only the scoped subgraph + only the
spans it fetches. The orchestrator never holds the whole system (or the whole
repo) in one context — it dispatches bounded units and keeps only their
structured results. This is what makes both *big repos* and *many skills*
tractable (see `okf-run-wiki.md` for how results are recorded without retaining
raw context). It is also exactly how this project was built: one subagent per
skill/file, each reading only the contracts it needed.

## 6. Caching across the run and across runs

- **Span cache** keyed by `(gid, hash)` — within and across stages.
- **Subgraph cache** for hot queries (blast radius, flows).
- **Build/recipe/AST caches** (Maven/Gradle, OpenRewrite ASTs) — `deployment-architecture.md` §4.
- **Embedding index (optional, complementary):** for *semantic* lookup ("where is
  retry logic?") an embedding index over span summaries can shortlist GIDs — but
  it only ever returns GIDs to resolve through the graph; the graph stays the
  source of truth, the embeddings are just a finder.

## 7. Invariants

- Whole-file reads happen at INDEX only; every later read is a hash-gated span
  fetch resolved from a GID.
- No stage loads more of the graph or source than its scoped query + read budget
  allow; budget overruns subdivide the work and are logged.
- Re-running a stage on an unchanged subgraph reads zero source (all spans
  served from cache).
- Summaries never replace evidence: a fact that drives a transformation still
  cites its exact source span.
