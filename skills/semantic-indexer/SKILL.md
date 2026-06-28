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

You are the **ground-truth surveyor**. Every later skill queries the graph
instead of re-parsing source, so the correctness of the entire pipeline rests
on the facts you write here. You own **layer L0 (Structural)** and only L0: you
add nodes and edges, you never mutate another stage's nodes. Read these
contracts before acting and treat them as authoritative:

- `architecture/semantic-graph-schema.md` — GID format, node/edge envelopes,
  layered fact model, change detection (§5), serialization
- `graph/ontology.md` — controlled vocabulary, invariants enforced on write
- `graph/node-types.md` — L0 node `kind`s and required `props`
- `graph/edge-types.md` — L0 edge `type`s and required `props`
- `architecture/system-design.md` — where you sit in the flow (§5 step 1)

> **Tools (preflight):** requires `tree-sitter` (+ grammars) and the symbol
> resolver at their pinned versions — resolved at INTAKE from `tooling/manifest.yaml`
> (`architecture/tooling-and-provisioning.md`). Use the pinned tool; never an
> ad-hoc/floating install.

## When to use

- "Index repo X" / orchestrator drives a repo into the **INDEX** state.
- Re-index after source changed (incremental — only the touched subgraph).
- Any time a downstream stage reports missing or stale L0 facts for a node.

Do **not** use this to recover architecture, infer business rules, or score
debt — those are L1–L5 and read your output. If asked a structural question
that the graph already answers, query it (see §Querying); do not re-parse.

## Inputs / Outputs

**Inputs**
- A **cloned repo worktree** at a pinned commit (provided by the orchestrator;
  this skill never clones or mutates source — read-only).
- A **pinned toolchain**: JDK version, build tool (Maven/Gradle), parser and
  resolver versions. These are recorded in `provenance.tool` for reproducibility.
- The `runId` and target `repo` slug (the GID prefix).
- Optional prior graph snapshot for the same repo (enables incremental re-index).

**Outputs** — L0 nodes and edges, written live (Bolt) or portable
(`nodes.ndjson` / `edges.ndjson`), per schema §7.

| L0 nodes | L0 edges |
|----------|----------|
| `Repo`, `Module`, `Package`, `Class`, `Method`, `Field`, `Endpoint`, `Table`, `Dependency`, `Flow`, `ExternalSystem` | `CONTAINS`, `DECLARES`, `CALLS`, `EXTENDS`, `IMPLEMENTS`, `OVERRIDES`, `READS`, `WRITES`, `RETURNS`/`PARAM_OF`, `HANDLED_BY`, `DEPENDS_ON_LIB`, `THROWS`, `ENTERS_AT`, `FLOWS_THROUGH`, `TOUCHES`, `DEPENDS_ON_EXTERNAL` |

Every node carries the common envelope (gid, kind, repo, fqn, name, lang, loc,
props, provenance with a **source-span `hash`**). Every edge carries the common
envelope (type, from, to, props, provenance). Resolved facts get
`confidence = 1.0`; inferred ones (reflection, DI, interface dispatch) get
`confidence < 1.0`.

## Procedure

1. **Detect the build system.** Scan the worktree root and subdirectories for
   `pom.xml` (Maven), `build.gradle[.kts]` / `settings.gradle` (Gradle), or fall
   back to a source-tree heuristic if none. Determine the module reactor /
   subproject set. Emit one **`Repo`** node (`props.buildSystem`, `vcsUrl`,
   `defaultBranch`, `lang`) and one **`Module`** per build module
   (`props.coordinates = group:artifact:version`, `packaging`).

2. **Resolve the classpath & dependencies.** Use the build tool itself
   (`mvn dependency:list -DoutputType=…` / `mvn dependency:build-classpath`,
   `gradle dependencies` / `:dependencies` resolution) to get the *exact*
   resolved graph — do not parse coordinates by hand. For each resolved
   artifact emit a **`Dependency`** node (`coordinates`, `version`, `scope`,
   `direct`) and a **`DEPENDS_ON_LIB`** edge `Module → Dependency`
   (`props.scope`, `props.direct`). The resolved classpath jars feed the
   bytecode resolver in step 3.

3. **Parse to ASTs + resolve symbols/types.** Recommended stack, in order of
   precision:
   - **tree-sitter** (`tree-sitter-java` and per-language grammars) for fast,
     fault-tolerant, multi-language AST coverage — survives partially broken
     code, which legacy repos always have.
   - A **symbol/type resolver / language server** (JavaParser+SymbolSolver,
     Eclipse JDT, or `jdtls`) for binding names to declarations, generic
     instantiation, and overload resolution.
   - **Bytecode** of compiled classes / dependency jars (ASM) for *precise*
     resolution of inherited members, bridge/synthetic methods, and call
     targets the source alone cannot disambiguate. Prefer bytecode-confirmed
     targets where available.
   Record the actual versions used in `provenance.tool`
   (e.g. `"tree-sitter-java@0.21 + JDT 1.39 + ASM 9.7"`).

4. **Build the symbol table.** Construct a per-module symbol table keyed by
   `fqn`: packages, types (class/interface/enum/record/annotation), methods
   (by erased + generic signature), fields. This table is the resolution index
   for every edge in step 6 and is what makes call/override resolution
   deterministic rather than textual.

5. **Emit structural nodes (containment first).** Walk the symbol table and emit:
   - **`Package`** (`classCount`), **`Class`** (`classKind`, `modifiers[]`,
     `isPublicApi`, `annotations[]`), **`Method`** (`signature`, `returnType`,
     `params[]`, `modifiers[]`, `cyclomatic`, `annotations[]`, `isEntryPoint`),
     **`Field`** (`type`, `modifiers[]`, `initializer`).
   - GID per schema §1: `<repo>::<kind>::<fqn>[#<disambiguator>]`. Methods MUST
     include the parameter-typed disambiguator,
     e.g. `billing-svc::Method::com.acme.billing.InvoiceService#charge(java.math.BigDecimal)`.
   - Set `loc` (file/startLine/endLine) and compute `provenance.hash =
     sha256(normalized source span)` for change detection (step 9).
   - Emit containment edges as you go: **`CONTAINS`** Repo→Module, Module→Package,
     Package→Class; **`DECLARES`** Class→Method, Class→Field. Containment is a
     tree — exactly one parent of the next-coarser kind (ontology §3.1).

6. **Resolve & emit relationship edges.** Using the symbol table + bytecode:
   - **`CALLS`** Method→Method per call site (`props.callSite` = line,
     `dynamic`, `confidence`). Bytecode/source-confirmed static target ⇒ 1.0.
   - **`EXTENDS`** Class→Class, **`IMPLEMENTS`** Class→interface,
     **`OVERRIDES`** Method→Method (overridden supertype method).
   - **`RETURNS`** / **`PARAM_OF`** Method→Class for type usage,
     **`THROWS`** Method→exception Class (declared *and* documented throws).
   - **Hard cases get edges at `confidence < 1.0`** — never drop them, never
     fake 1.0 (see §Resolving hard cases). Always set `provenance.tool` to the
     resolver that produced an inferred edge.

7. **Enumerate ALL entry points (no single front door).** Legacy systems are
   entered through many surfaces; you must build a **complete inventory** across
   every class below, not just HTTP. Emit one **`Endpoint`** node + **`HANDLED_BY`**
   Endpoint→Method edge each, set the handler Method's `props.isEntryPoint = true`,
   and set `entryClass` + `protocol` (see `architecture/entry-point-discovery.md`):
   - **HTTP/REST**: Spring `@RestController`/`@RequestMapping`/`@GetMapping…`,
     JAX-RS `@Path`/`@GET…`, servlet mappings ⇒ `entryClass=http`, `verb`, `path`.
   - **GraphQL / SOAP / gRPC**: resolvers, `@WebService`, proto service impls.
   - **Messaging**: `@KafkaListener`, `@RabbitListener`, JMS `@JmsListener`/
     `MessageListener`, SQS pollers ⇒ `entryClass=messaging`, `topic`.
   - **Scheduled**: `@Scheduled`, Quartz `Job`, cron ⇒ `entryClass=scheduled`, `schedule`.
   - **Batch**: Spring Batch `Job`/`Step`, batch `CommandLineRunner`.
   - **CLI / main**: `public static void main`, picocli/`@Command`.
   - **Webhooks / callbacks**: inbound callback/OAuth-redirect handlers.
   - **Startup / lifecycle**: `@PostConstruct`, `ApplicationRunner`,
     `@EventListener(ApplicationReadyEvent)` ⇒ `entryClass=startup`.
   - **File / FS watchers**: `WatchService`, directory/FTP/S3 event handlers.
   - **Public library API**: exported `isPublicApi` methods consumed by other repos.
   - **Test fixtures**: detected but set `props.synthetic = true` (excluded from
     production flows, retained for coverage).
   Resolve **dynamic/implicit** entries too (reflection, DI/annotation-driven
   registration, config-declared handlers); where unresolved, emit a
   low-`confidence` Endpoint + a discovery `Gap` rather than missing it.

8. **Detect data access (JPA/JDBC).** Catalog persistent stores and the
   read/write flow into them:
   - **JPA/Hibernate**: `@Entity`/`@Table` ⇒ **`Table`** node (`schema`,
     `columns[]` from `@Column`/fields, `pk[]` from `@Id`). Repository/EM calls
     ⇒ **`READS`**/**`WRITES`** Method→Table with `props.via = jpa`.
   - **JDBC / MyBatis / Spring JDBC**: parse SQL in string literals,
     `@Query`, and mapper XML; map SELECT⇒`READS`, INSERT/UPDATE/DELETE⇒`WRITES`
     Method→Table with `props.via = jdbc` (`confidence < 1.0` when the SQL is
     dynamically assembled).
   - **Field data flow**: intra-program `READS`/`WRITES` Method→Field with
     `props.via = field`.

8b. **Trace a Flow from every entry point.** With the call graph, entry-point
    inventory, and data access in place, materialize an execution slice per
    entry point (`architecture/entry-point-discovery.md`): root a traversal at
    each non-synthetic `Endpoint`'s handler, walk `CALLS*` to the reachable
    method set, and record the data/external touchpoints. Emit one **`Flow`**
    node + **`ENTERS_AT`** (Flow→Endpoint), **`FLOWS_THROUGH`** (Flow→Method),
    and **`TOUCHES`** (Flow→Table/ExternalSystem) edges; follow cross-repo
    contracts so a flow continues into other repos (`props.crossRepo`). **Every
    non-synthetic entry point must yield ≥1 Flow** — a zero-flow entry point is a
    discovery `Gap`, never dropped. Flows are L0 facts owned here; DISCOVER and
    later stages **read** them, they don't re-trace.

9. **Write to the graph with provenance.** Stamp every node and edge with
   `provenance.stage = "semantic-indexer"`, `provenance.tool`, `provenance.runId`,
   and (nodes) the source-span `hash`. Honor invariants before commit
   (§Quality & invariants). For any edge whose `from.repo != to.repo` (e.g. an
   HTTP call resolved to another repo's Endpoint) set `props.crossRepo = true`
   and `props.contract` = the shared contract GID (edge-types §Cross-repo).
   Write live (Bolt) or portable (`nodes.ndjson` / `edges.ndjson`).

10. **Incremental re-index via hash.** When a prior snapshot exists: recompute
    each node's source-span `hash`; **reuse** nodes whose hash is unchanged, and
    for a changed hash **invalidate the node and all derived facts that
    reference it** via provenance back-links (schema §5), so downstream stages
    recompute only the affected subgraph. Re-resolve edges only for touched
    nodes and their direct dependents. Deleted spans ⇒ tombstone the node and
    cascade-invalidate. Re-index is idempotent: re-running on unchanged source
    is a no-op that re-stamps `runId`.

## Resolving hard cases (and how to record confidence)

The graph is only useful if hard edges are *present but honestly scored*. Rule:
a fact you can prove from source+bytecode is `confidence = 1.0`; a fact you
infer is `< 1.0`, and `provenance.tool` names the inference.

- **Framework DI (Spring).** A field/constructor injected by type/qualifier has
  no source call to its concrete impl. Resolve the bean to its concrete
  `Class`/`Method` from `@Component`/`@Service`/`@Configuration @Bean`
  definitions and `@Qualifier`/`@Primary`; emit the `CALLS`/usage edge to the
  resolved bean with `confidence ≈ 0.8` and `props.dynamic = true`. If multiple
  candidate beans satisfy the type, emit an edge to each at lower confidence
  (split among candidates).

- **Reflection.** `Class.forName`, `Method.invoke`, proxies, service loaders:
  resolve the target from string/constant analysis when possible and emit
  `CALLS` with `props.dynamic = true`, `confidence ≈ 0.5`. If unresolvable,
  record the call site on the calling Method (do not invent a target edge).

- **Interface dispatch / polymorphism.** A virtual call to an interface or
  abstract method resolves to *all* known implementors (via `IMPLEMENTS`/
  `EXTENDS`/`OVERRIDES` in the symbol table). Emit a `CALLS` edge to each
  candidate target with `confidence` ≈ `1/N` of the candidate set (or higher
  when CHA/RTA narrows it). Single implementor ⇒ `confidence = 1.0`.

- **Generics.** Resolve type arguments through the resolver so
  `List<Invoice>.get(i)` binds element types correctly for `RETURNS`/`PARAM_OF`
  and downstream `CALLS`. Erased-only resolution (no instantiation available)
  ⇒ keep the edge but set `confidence < 1.0`.

- **Dynamic SQL & string-built routes.** Concatenated SQL or computed paths:
  best-effort table/path extraction at `confidence < 1.0`; never claim 1.0 for
  a string you could not fully evaluate.

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

Enforce before commit; a violation is a bug in L0, not a downstream problem.

- **Containment is a tree.** Each Module/Package/Class/Method/Field has exactly
  one containment (`CONTAINS`/`DECLARES`) parent of the next-coarser kind
  (ontology §3.1). No orphan structural nodes (except `Repo`).
- **Every node has provenance + source-span hash.** `provenance.stage =
  "semantic-indexer"`, a concrete `provenance.tool`, the `runId`, and a
  `sha256` span `hash`. Non-source nodes (Repo/Module/Dependency) hash their
  defining descriptor (pom/gradle coordinate block).
- **GID format is exact** (`<repo>::<kind>::<fqn>[#<disambiguator>]`), repo-
  prefixed, with parameter-typed disambiguators on Methods. GIDs are stable
  across re-index so edges and downstream layers keep pointing at the same node.
- **Confidence discipline.** Resolved = 1.0; inferred < 1.0 with the inferring
  tool named. Never upgrade an inferred edge to 1.0.
- **Cross-repo edges are explicit.** `props.crossRepo = true` + `props.contract`
  whenever `from.repo != to.repo`.
- **L0 only.** Do not emit L1–L5 kinds/types. Treat unknown kinds/types
  defensively on read (ignore, don't fail) for forward-compat (ontology §5).
- **Read-only on source.** Never modify the worktree; parsing only.

## Querying contract

Downstream stages (and you, on re-index) MUST query the graph rather than
re-parse for structural questions. Promote broadly useful queries (e.g.
"transitive callers of method X", "endpoints touching table T") into
`graph/cypher-queries.md` rather than inlining them per skill.

## Definition of done

The L0 subgraph for the repo is complete and internally consistent: every
build module, package, type, method, and field is a node under a single
containment tree; the call graph, inheritance/override, data-access, and
entry-point edges are resolved (hard cases present with honest `confidence`);
**the entry-point inventory is complete across all classes** (HTTP, messaging,
scheduled, batch, CLI/main, webhook, startup, file, library — with `entryClass`
set and test fixtures marked `synthetic`); endpoints, tables, and dependencies
are cataloged; every node and edge carries
full provenance and (nodes) a source-span hash; cross-repo edges name their
contract; and on re-index only changed spans were recomputed while unchanged
nodes were reused and dependent derived facts invalidated. Downstream stages
can now answer structural questions purely by querying the graph.
