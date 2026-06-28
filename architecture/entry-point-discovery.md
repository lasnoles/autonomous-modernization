# Entry-Point Discovery & Flow Exploration

Legacy systems rarely have a single `main()`. They are entered through dozens of
surfaces — HTTP routes, message listeners, scheduled jobs, batch runners, CLI
commands, webhooks, startup hooks, library APIs. **Discovery must enumerate
every entry point and trace a flow from each one** — otherwise whole behaviors
(and the business rules inside them) are invisible to the planner and never
get preserved or tested.

This refines the INDEX and DISCOVER stages and adds `Flow` to the graph.

## 1. Principle: completeness over a "front door"

There is no assumed front door. The indexer builds a **complete entry-point
inventory** (the `Endpoint` node kind covers every entry surface — see
`graph/node-types.md`), and discovery explores **all reachable flows**. Coverage is a
first-class, measured property: every detected entry point must have ≥1 traced
flow, and code reachable only from an untraced entry point is flagged, never
silently dropped.

## 2. Entry-point taxonomy (detect all of these)

| Class | How it's detected (Java-centric, multi-lang aware) |
|-------|----------------------------------------------------|
| HTTP / REST | `@RestController`/`@RequestMapping`/JAX-RS `@Path`, servlet mappings |
| GraphQL / SOAP / gRPC | schema resolvers, `@WebService`, proto service impls |
| Messaging consumers | `@KafkaListener`, `@RabbitListener`, JMS `@JmsListener`, SQS pollers |
| Scheduled / cron | `@Scheduled`, Quartz `Job`, cron entries |
| Batch jobs | Spring Batch `Job`/`Step`, `CommandLineRunner` batch mains |
| CLI / main | `public static void main`, picocli/`@Command` |
| Webhooks / callbacks | inbound callback routes, OAuth/redirect handlers |
| Startup / lifecycle hooks | `@PostConstruct`, `ApplicationRunner`, `@EventListener(ApplicationReadyEvent)` |
| File / FS watchers | directory pollers, `WatchService`, FTP/S3 event handlers |
| Public library API | `isPublicApi` exported classes/methods consumed by other repos |
| Test fixtures | detected but **tagged `synthetic`** — excluded from production flows, used for coverage |

Each becomes (or annotates) an `Endpoint` node (`graph/node-types.md`) with its
`protocol`, `verb`/`path`/`topic`/`schedule`, and `entryClass` from the table.

## 3. Flow exploration — trace from every entry point

A **Flow** is an execution slice rooted at one entry point: the transitive
call graph from that entry through methods, into the data it reads/writes and
the external systems it calls. One entry point may yield several flows
(branches, error paths); a flow may span repos (cross-repo contracts).

For each `Endpoint`:
1. Root a traversal at its handler method (`HANDLED_BY`).
2. Walk `CALLS*` to build the reachable method set (mark `dynamic`/DI/reflection
   edges at `confidence<1.0`).
3. Record data touchpoints (`READS`/`WRITES` → Field/Table) and external
   touchpoints (`DEPENDS_ON_EXTERNAL` → ExternalSystem, incl. placeholders).
4. Follow cross-repo contracts to continue the flow into consumer/producer repos.
5. Materialize a `Flow` node + edges (`ENTERS_AT`, `FLOWS_THROUGH`, `TOUCHES`).

Flows are the unit that downstream stages reason about:
- **business-discovery** lifts each flow into the capability/rules it realizes.
- **planner** uses flow blast-radius to size ChangeUnits and order waves.
- **tester** turns each significant flow into an E2E scenario from prod traces.
- **validation/risk** measure coverage and exposure per flow.

## 4. Graph additions

Node: **Flow** — `props`: `rootEntryPoint` (gid), `kind` (request|message|
scheduled|batch|cli|startup|lib), `methodSet` (count), `crossRepo` (bool),
`coverage` (0–1), `synthetic` (bool). See `graph/node-types.md`.

Edges: `ENTERS_AT` (Flow→Endpoint), `FLOWS_THROUGH` (Flow→Method),
`TOUCHES` (Flow→Table/ExternalSystem). See `graph/edge-types.md`.

## 5. Completeness invariants

- **Every `Endpoint` has ≥1 `Flow`.** A zero-flow entry point is a discovery
  gap → register a `Gap` (`open-question-resolution.md`), don't ignore it.
- **Reachability is reported.** Methods unreachable from any non-synthetic entry
  point are flagged (candidate dead code or a missed dynamic entry).
- **Dynamic/implicit entries are hunted, not assumed absent.** Reflection, DI
  wiring, annotation-driven registration, and config-declared handlers are
  resolved where possible and otherwise recorded as low-confidence entries +
  gaps.
- **Cross-repo flows are continued, not truncated** at the repo boundary.
- The entry-point inventory and per-flow coverage appear in the run wiki
  (`entry-points/index.md`, `flows/index.md`) and the monitoring status.

## 6. Reusable queries

Added to `graph/cypher-queries.md`: "all entry points (by class)", "flow trace
from an entry point", "entry points with no flow", "methods unreachable from any
entry point", "entry points touching table/external X".
