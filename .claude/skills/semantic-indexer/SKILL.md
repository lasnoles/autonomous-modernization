---
name: semantic-indexer
description: >-
  Structural indexer (graph layer L0) for autonomous legacy-Java modernization.
  Parses a cloned repo worktree (Java primary, multi-language aware), builds
  ASTs, resolves symbols/types, constructs the call graph, captures data flow
  (field/table reads & writes), detects external entry points (HTTP routes,
  message listeners, scheduled jobs, CLI), and catalogs build modules &
  dependencies — then materializes all of it as L0 nodes and edges with full
  provenance and source-span hashes. Incremental: reuses nodes whose span hash
  is unchanged and invalidates derived facts on change. Use this at the INDEX
  state, before any recover/discover/debt/plan stage can query structure.
---

# Semantic Indexer — Structural Layer (L0) Builder

You are the ground-truth surveyor: every later skill queries the graph instead
of re-parsing source. You own **layer L0 only** — add nodes/edges, never mutate
another stage's. Read-only on source.

Authoritative contracts: `architecture/semantic-graph-schema.md` (GID format,
envelopes, layered fact model, change detection §5, serialization §7) ·
`graph/ontology.md` (vocabulary + write invariants) · `graph/node-types.md` ·
`graph/edge-types.md` (L0 `kind`s/`type`s + required props) ·
`architecture/system-design.md` §5 step 1 (where you sit) ·
`architecture/language-profiles.md` (the language plugin you run as — see below).

> **Run as the active language profile.** Read `source.language`'s profile and
> use *its* grammar, symbol/type resolver, build systems, dependency manifests,
> and entry-point/data-access detectors. Everything language-specific below is
> the **`java`** profile; under `python` substitute the `python` profile
> (tree-sitter-python + Jedi/Pyright; poetry/pip + `pyproject.toml`/`requirements*`;
> FastAPI/Flask/Django/Celery/APScheduler/click entry points;
> SQLAlchemy/Django-ORM/DB-API data access) — same L0 schema out. For
> dynamically-typed languages, honor `profile.indexer.typing: dynamic`: emit
> inferred call/dataflow edges at `confidence < 1.0` (resolver named in
> provenance), never a fake 1.0.

> **Tools (preflight):** `tree-sitter` (+ grammars) and the symbol resolver at
> pinned versions, resolved at INTAKE from `tooling/manifest.yaml`
> (`architecture/tooling-and-provisioning.md`). Pinned only — never ad-hoc/floating.

## When to use
- "Index repo X" / orchestrator drives a repo into **INDEX**.
- Re-index after source changed (incremental — only the touched subgraph).
- A downstream stage reports missing/stale L0 facts.

NOT for recovering architecture, inferring business rules, or scoring debt
(L1–L5, which read your output). If the graph already answers a structural
question, query it (§Querying) — don't re-parse.

## Inputs / Outputs

**In:** a cloned repo worktree at a pinned commit (read-only; never clone or
mutate) · a pinned toolchain (JDK, Maven/Gradle, parser, resolver versions →
recorded in `provenance.tool`) · `runId` + target `repo` slug (GID prefix) ·
optional prior graph snapshot (enables incremental).

**Out:** L0 nodes + edges, written live (Bolt) or portable
(`nodes.ndjson`/`edges.ndjson`) per schema §7.

| L0 nodes | L0 edges |
|----------|----------|
| `Repo`, `Module`, `Package`, `Class`, `Method`, `Field`, `Endpoint`, `Table`, `Dependency`, `Flow`, `ExternalSystem` | `CONTAINS`, `DECLARES`, `CALLS`, `EXTENDS`, `IMPLEMENTS`, `OVERRIDES`, `READS`, `WRITES`, `RETURNS`/`PARAM_OF`, `HANDLED_BY`, `DEPENDS_ON_LIB`, `THROWS`, `ENTERS_AT`, `FLOWS_THROUGH`, `TOUCHES`, `DEPENDS_ON_EXTERNAL` |

Every node carries the common envelope (gid, kind, repo, fqn, name, lang, loc,
props, provenance with source-span `hash`); every edge carries its envelope
(type, from, to, props, provenance). Resolved facts ⇒ `confidence = 1.0`;
inferred (reflection, DI, interface dispatch) ⇒ `confidence < 1.0`.

## Procedure

> **Index in bounded batches — never hold a repo (or its dependency set) in
> context.** Hard rule for large multi-repo runs
> (`architecture/scalability-and-retrieval.md` §1a). Process **one module at a
> time**; within a module, page declarations and dependencies in batches of
> `execution.batchSize` (default 25), respecting `maxConcurrentReads` (default 4
> in flight). **Write each batch straight to graph/NDJSON and drop it from
> context** before the next; checkpoint a cursor (`module`, `lastDeclaration`)
> per batch so a timeout resumes mid-repo, not restarts. Never buffer nodes to
> "write at the end"; never read whole files except this stage's one streaming
> pass per file.

```
1. DETECT build system. Scan root+subdirs for pom.xml (Maven),
   build.gradle[.kts]/settings.gradle (Gradle), else source-tree heuristic.
   Determine reactor/subproject set.
   Emit one Repo (props.buildSystem, vcsUrl, defaultBranch, lang)
        + one Module per build module (coordinates=group:artifact:version, packaging).

2. RESOLVE classpath & deps via the build tool itself (mvn dependency:list /
   :build-classpath, gradle dependencies) — never hand-parse coordinates.
   STREAM resolved artifacts in batches: per batch emit Dependency
   (coordinates, version, scope, direct) + DEPENDS_ON_LIB Module→Dependency,
   write, drop. Never hold the full (thousands-deep transitive) set.
   Resolved jars feed the bytecode resolver in step 3.

3. PARSE to ASTs + resolve, in order of precision:
   • tree-sitter (tree-sitter-java + per-lang grammars) — fast, fault-tolerant,
     survives partially-broken legacy code.
   • symbol/type resolver / LS (JavaParser+SymbolSolver, Eclipse JDT, jdtls) —
     name binding, generic instantiation, overload resolution.
   • bytecode of compiled classes/dep jars (ASM) — precise inherited members,
     bridge/synthetic methods, ambiguous call targets. Prefer bytecode-confirmed.
   Record actual versions in provenance.tool (e.g. "tree-sitter-java@0.21 + JDT 1.39 + ASM 9.7").

4. BUILD per-module symbol table keyed by fqn: packages, types
   (class/interface/enum/record/annotation), methods (erased + generic sig),
   fields. This is the resolution index for step 6 — makes call/override
   resolution deterministic, not textual.

5. EMIT structural nodes (containment first), one batch of batchSize
   declarations at a time (write + release each batch; don't buffer the module):
   • Package(classCount), Class(classKind, modifiers[], isPublicApi, annotations[]),
     Method(signature, returnType, params[], modifiers[], cyclomatic, annotations[], isEntryPoint),
     Field(type, modifiers[], initializer).
   • GID per schema §1: <repo>::<kind>::<fqn>[#<disambiguator>]. Methods MUST
     carry the parameter-typed disambiguator, e.g.
     billing-svc::Method::com.acme.billing.InvoiceService#charge(java.math.BigDecimal).
   • Set loc (file/startLine/endLine); compute provenance.hash = sha256(normalized span).
   • Containment edges as you go: CONTAINS Repo→Module, Module→Package, Package→Class;
     DECLARES Class→Method, Class→Field. Tree: exactly one next-coarser parent (ontology §3.1).

6. RESOLVE & emit relationship edges (symbol table + bytecode):
   • CALLS Method→Method per call site (props.callSite=line, dynamic, confidence);
     bytecode/source-confirmed static target ⇒ 1.0.
   • EXTENDS Class→Class, IMPLEMENTS Class→iface, OVERRIDES Method→supertype-Method.
   • RETURNS / PARAM_OF Method→Class (type usage); THROWS Method→exception Class
     (declared + documented).
   • Hard cases get edges at confidence < 1.0 — never drop, never fake 1.0
     (§Hard cases). Set provenance.tool to the inferring resolver.

7. ENUMERATE ALL entry points (no single front door — build a COMPLETE inventory
   across every class below, not just HTTP). Each ⇒ one Endpoint + HANDLED_BY
   Endpoint→Method, set handler props.isEntryPoint=true, set entryClass + protocol
   (architecture/entry-point-discovery.md):
   • HTTP/REST: Spring @RestController/@RequestMapping/@GetMapping…, JAX-RS
     @Path/@GET…, servlet mappings ⇒ entryClass=http, verb, path.
   • GraphQL/SOAP/gRPC: resolvers, @WebService, proto service impls.
   • Messaging: @KafkaListener, @RabbitListener, JMS @JmsListener/MessageListener,
     SQS pollers ⇒ entryClass=messaging, topic.
   • Scheduled: @Scheduled, Quartz Job, cron ⇒ entryClass=scheduled, schedule.
   • Batch: Spring Batch Job/Step, batch CommandLineRunner.
   • CLI/main: public static void main, picocli/@Command.
   • Webhooks/callbacks: inbound callback/OAuth-redirect handlers.
   • Startup/lifecycle: @PostConstruct, ApplicationRunner,
     @EventListener(ApplicationReadyEvent) ⇒ entryClass=startup.
   • File/FS watchers: WatchService, directory/FTP/S3 event handlers.
   • Public library API: exported isPublicApi methods consumed by other repos.
   • Test fixtures: detect but set props.synthetic=true (excluded from prod flows, kept for coverage).
   Resolve dynamic/implicit entries too (reflection, DI/annotation-driven
   registration, config-declared handlers); where unresolved, emit a
   low-confidence Endpoint + a discovery Gap rather than missing it.

8. DETECT data access (JPA/JDBC) — catalog stores + read/write flow:
   • JPA/Hibernate: @Entity/@Table ⇒ Table (schema, columns[] from @Column/fields,
     pk[] from @Id). Repository/EM calls ⇒ READS/WRITES Method→Table, props.via=jpa.
   • JDBC/MyBatis/Spring JDBC: parse SQL in string literals, @Query, mapper XML;
     SELECT⇒READS, INSERT/UPDATE/DELETE⇒WRITES Method→Table, props.via=jdbc
     (confidence < 1.0 when SQL is dynamically assembled).
   • Field data flow: intra-program READS/WRITES Method→Field, props.via=field.

8b. TRACE a Flow from every entry point (entry-point-discovery.md): root a
    traversal at each non-synthetic Endpoint's handler, walk CALLS* to the
    reachable method set, record data/external touchpoints. Emit Flow + ENTERS_AT
    (Flow→Endpoint), FLOWS_THROUGH (Flow→Method), TOUCHES (Flow→Table/ExternalSystem);
    follow cross-repo contracts so a flow continues into other repos (props.crossRepo).
    Every non-synthetic entry point MUST yield ≥1 Flow — zero-flow ⇒ discovery Gap,
    never dropped. Flows are L0 facts owned here; DISCOVER+later stages read, don't re-trace.

9. WRITE with provenance. Stamp every node/edge: provenance.stage="semantic-indexer",
   provenance.tool, provenance.runId, and (nodes) source-span hash. Honor invariants
   before commit (§Quality). For any edge with from.repo != to.repo set
   props.crossRepo=true + props.contract=shared contract GID (edge-types §Cross-repo).
   Write live (Bolt) or portable (nodes.ndjson/edges.ndjson).

10. INCREMENTAL re-index via hash (when prior snapshot exists): recompute each
    node's span hash; reuse unchanged; for a changed hash invalidate the node +
    all derived facts referencing it via provenance back-links (schema §5), so
    downstream recomputes only the affected subgraph. Re-resolve edges only for
    touched nodes + direct dependents. Deleted spans ⇒ tombstone + cascade-invalidate.
    Idempotent: re-running on unchanged source is a no-op that re-stamps runId.
```

## Resolving hard cases (and recording confidence)

Rule: provable from source+bytecode ⇒ `confidence = 1.0`; inferred ⇒ `< 1.0`
with `provenance.tool` naming the inference. Present but honestly scored — never drop.

- **Framework DI (Spring).** Injected field/ctor has no source call to its impl.
  Resolve the bean to concrete `Class`/`Method` from
  `@Component`/`@Service`/`@Configuration @Bean` + `@Qualifier`/`@Primary`; emit
  `CALLS`/usage to it at `confidence ≈ 0.8`, `props.dynamic = true`. Multiple
  candidate beans ⇒ one edge each at lower confidence (split among candidates).
- **Reflection.** `Class.forName`, `Method.invoke`, proxies, service loaders:
  resolve target from string/constant analysis when possible ⇒ `CALLS`,
  `props.dynamic = true`, `confidence ≈ 0.5`. Unresolvable ⇒ record the call site
  on the calling Method (don't invent a target edge).
- **Interface dispatch / polymorphism.** Virtual call ⇒ all known implementors
  (via `IMPLEMENTS`/`EXTENDS`/`OVERRIDES`). Emit `CALLS` to each candidate at
  `confidence ≈ 1/N` (higher when CHA/RTA narrows). Single implementor ⇒ 1.0.
- **Generics.** Resolve type args so `List<Invoice>.get(i)` binds element types
  for `RETURNS`/`PARAM_OF` + downstream `CALLS`. Erased-only (no instantiation)
  ⇒ keep edge, `confidence < 1.0`.
- **Dynamic SQL & string-built routes.** Best-effort table/path extraction at
  `confidence < 1.0`; never claim 1.0 for a string you couldn't fully evaluate.

## Graph writes (exact type mapping)

| Source fact | Node `kind` | Edge `type` (from → to) | key props |
|-------------|-------------|-------------------------|-----------|
| repository | `Repo` | — | `vcsUrl`, `defaultBranch`, `buildSystem`, `lang` |
| build module | `Module` | `CONTAINS` Repo→Module | `coordinates`, `packaging` |
| package | `Package` | `CONTAINS` Module→Package | `classCount` |
| type | `Class` | `CONTAINS` Package→Class | `classKind`, `modifiers[]`, `isPublicApi`, `annotations[]` |
| method/ctor | `Method` | `DECLARES` Class→Method | `signature`, `returnType`, `params[]`, `cyclomatic`, `isEntryPoint` |
| field | `Field` | `DECLARES` Class→Field | `type`, `modifiers[]`, `initializer` |
| invocation | — | `CALLS` Method→Method | `callSite`, `dynamic`, `confidence` |
| superclass | — | `EXTENDS` Class→Class | — |
| implemented iface | — | `IMPLEMENTS` Class→Class | — |
| override | — | `OVERRIDES` Method→Method | — |
| return/param type | — | `RETURNS` / `PARAM_OF` Method→Class | — |
| declared throw | — | `THROWS` Method→Class | — |
| field/table read | — | `READS` Method→Field/Table | `via` (jdbc\|jpa\|field) |
| field/table write | — | `WRITES` Method→Field/Table | `via` |
| entry point (any class) | `Endpoint` | `HANDLED_BY` Endpoint→Method | `protocol`, `entryClass`, `verb`, `path`/`topic`/`schedule`, `synthetic` |
| execution slice from an entry point | `Flow` | `ENTERS_AT` Flow→Endpoint, `FLOWS_THROUGH` Flow→Method, `TOUCHES` Flow→Table/ExternalSystem | `rootEntryPoint`, `kind`, `methodSet`, `crossRepo`, `coverage`, `synthetic` |
| persistent store | `Table` | (target of `READS`/`WRITES`) | `schema`, `columns[]`, `pk[]` |
| external dependency (runtime) | `ExternalSystem` | `DEPENDS_ON_EXTERNAL` Method/Component→ExternalSystem | `sysKind`, `name`, `known` (false when undiscovered) |
| external library | `Dependency` | `DEPENDS_ON_LIB` Module→Dependency | `coordinates`, `version`, `scope`, `direct` |

## Quality & invariants

Enforce before commit; a violation is an L0 bug, not a downstream problem.

- **Containment is a tree.** Each Module/Package/Class/Method/Field has exactly
  one `CONTAINS`/`DECLARES` parent of the next-coarser kind (ontology §3.1). No
  orphan structural nodes (except `Repo`).
- **Every node has provenance + source-span hash.** `stage = "semantic-indexer"`,
  concrete `tool`, `runId`, `sha256` span `hash`. Non-source nodes
  (Repo/Module/Dependency) hash their defining descriptor (pom/gradle coordinate block).
- **GID format is exact** (`<repo>::<kind>::<fqn>[#<disambiguator>]`),
  repo-prefixed, parameter-typed disambiguators on Methods. Stable across
  re-index so edges/downstream layers keep pointing at the same node.
- **Confidence discipline.** Resolved = 1.0; inferred < 1.0 with the inferring
  tool named. Never upgrade an inferred edge to 1.0.
- **Cross-repo edges explicit.** `props.crossRepo = true` + `props.contract`
  whenever `from.repo != to.repo`.
- **L0 only.** No L1–L5 kinds/types. Treat unknown kinds/types defensively on
  read (ignore, don't fail) for forward-compat (ontology §5).
- **Read-only on source.** Never modify the worktree; parsing only.

## Querying contract

Downstream stages (and you, on re-index) MUST query the graph rather than
re-parse for structural questions. Promote broadly useful queries (e.g.
"transitive callers of method X", "endpoints touching table T") into
`graph/cypher-queries.md` rather than inlining per skill.

## Definition of done

The repo's L0 subgraph is complete and internally consistent: every build
module, package, type, method, field is a node under a single containment tree;
the call graph, inheritance/override, data-access, and entry-point edges are
resolved (hard cases present with honest `confidence`); **the entry-point
inventory is complete across all classes** (HTTP, messaging, scheduled, batch,
CLI/main, webhook, startup, file, library — `entryClass` set, test fixtures
marked `synthetic`); endpoints, tables, dependencies cataloged; every node and
edge carries full provenance and (nodes) a source-span hash; cross-repo edges
name their contract; on re-index only changed spans were recomputed while
unchanged nodes were reused and dependent derived facts invalidated. Downstream
stages can now answer structural questions purely by querying the graph.
