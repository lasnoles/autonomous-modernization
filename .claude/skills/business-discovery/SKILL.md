---
name: business-discovery
description: >-
  DISCOVER stage of the modernization autopilot. Owns graph layer L2
  (Business). Reads the L0 structural graph and L1 architecture and infers
  the behavior worth preserving: business Capabilities, discrete BusinessRules
  (invariants, validations, guards), and DomainEntities distinct from the
  classes that implement them. Mines rules from validation logic, conditionals,
  exceptions, test assertions, naming, and comments; uses the LLM to lift them
  into plain-language statements. This is the most LLM-driven stage, so every
  inferred fact records confidence + derivation (test|code|comment|llm) and
  cites evidence (source spans / test names). Its output becomes the
  `businessRefs`/PRESERVES targets the planner and compiler must keep — so
  quality here gates safe transformation. Use this after RECOVER, before DEBT.
---

# Business Discovery — L2 (Business) Layer

You are the archaeologist of intent: recover what the code *does for users* and
which behaviors must survive modernization. You don't transform code — read the
graph, read source only for evidence, write L2 nodes/edges. Every reconstruction
is a hypothesis with a confidence and a citation, never ground truth (business
rules in legacy Java are smeared across `if`s, exceptions, validation
annotations, magic constants, and comments — you reconstruct intent, not a spec).

Authoritative contracts (read first): `architecture/semantic-graph-schema.md`
(envelopes, GID format, layered facts) · `graph/ontology.md` (invariants, esp.
§3.3: L2 edges carry `confidence` + `provenance.tool` when inferred) ·
`graph/node-types.md` §L2 / `graph/edge-types.md` §L2 (exact shapes you may
write) · `architecture/modernization-ir.md` §2 (how `businessRefs`/`PRESERVES`
consume your output).

## When to use
- Orchestrator advances a repo RECOVER → DISCOVER (L0 + L1 present).
- "What business rules does this service enforce?" / "What must we not break?"
- Re-discovery on incremental re-index: re-infer only over the subgraph whose
  source-span `hash` changed (schema §5).

## Inputs
- **L0 graph** (required): Class, Method, Field, Endpoint, Table nodes; `CALLS`,
  `HANDLED_BY`, `READS`/`WRITES`, `THROWS`, `DECLARES` edges. Method nodes carry
  `signature`, `annotations[]`, `cyclomatic`, `loc`.
- **L1 graph** (required): Component, Layer, Seam; `BELONGS_TO`, `DEPENDS_ON`.
  Components are your strongest capability candidates.
- **Source** (read-only, evidence only): extract exact spans, test-assertion
  text, comments you cite. Per schema §6 MUST NOT re-parse source for structural
  questions — query the graph; open source only for human-readable evidence.
- **Production traces** (required — Target Spec §1 `inputs.productionTraces`,
  validated present at INTAKE): recorded real request/response + event traffic.
  Behavioral evidence — observed inputs/outputs along each `Flow` corroborate or
  contradict code-mined rules and let you rank confidence on what production
  actually does. Same traces are reused by tester/replay for E2E equivalence, so
  capabilities/flows anchored here become the journeys proven later.

## Outputs (L2 — all appended, never mutating prior layers)

Reads (not written here): **Flow** nodes + `ENTERS_AT`/`FLOWS_THROUGH`/`TOUCHES`
are L0 facts from `semantic-indexer` at INDEX — consume them (step 0), don't write.

Nodes (`graph/node-types.md` §L2):
- **Capability** — `description`, `criticality` (low|med|high), `owners[]`.
- **BusinessRule** — `statement` (plain language), `derivation`
  (test|code|comment|llm), `confidence` (0–1), `evidence` (source spans / test
  names justifying the rule).
- **DomainEntity** — `name`, `aliases[]`, `lifecycleStates[]`.

Edges (`graph/edge-types.md` §L2 — all inference-derived edges set
`confidence < 1.0` and record `provenance.tool`, per ontology §3.3):
- **REALIZES_CAPABILITY** (Component/Class → Capability) — `confidence`.
- **IMPLEMENTS_RULE** (Method → BusinessRule) — `confidence`, `derivation`.
- **MANIPULATES** (Method → DomainEntity) — `op` (create|read|update|delete).
- **MAPS_TO** (Class → DomainEntity) — `confidence`.

GIDs per schema §1: `billing-svc::Capability::Invoicing`,
`billing-svc::BusinessRule::invoice-amount-non-negative`,
`billing-svc::DomainEntity::Invoice`.

## Procedure

> **One capability/flow at a time, methods in batches.** Don't load the repo's
> whole method/flow set. Iterate capabilities; within one, page methods in
> batches of `execution.batchSize`, write resulting L2 rules/edges per batch,
> drop the batch before the next (`architecture/scalability-and-retrieval.md`
> §1a). Fetch only the **exact evidence spans** you cite (never whole files);
> checkpoint a cursor (capability, lastMethod) so a timeout resumes mid-discovery.

```
0. START FROM EVERY FLOW (no single front door). Flow slices traced at INDEX
   (L0, owned by semantic-indexer — one per entry point across all surfaces;
   architecture/entry-point-discovery.md). READ via graph (cypher: "flow trace
   from an entry point") — do not re-trace/re-parse source. Verify completeness:
   every non-synthetic Endpoint has ≥1 Flow; if any lacks one, raise a discovery
   Gap (never silently skip). Flows are the substrate for rules/capabilities and
   are reused by tester E2E scenarios + planner blast-radius sizing.

1. IDENTIFY CAPABILITY CANDIDATES. Query L1 Components (responsibility + member
   classes), L0 Endpoints (HANDLED_BY = surface verbs an external actor invokes),
   naming clusters (Invoice*, charge, refund). Group into a small set of coarse
   capabilities (Invoicing, Payments, Dunning). Prefer one Capability per cohesive
   Component; split/merge via endpoint grouping + shared domain vocabulary. Write
   Capability nodes; link realizing Components/Classes with REALIZES_CAPABILITY
   (confidence = how directly code maps to the named capability).

2. MINE RULES, ranked by evidence strength (descending trust):
   • Tests — assertions, parameterized cases, esp. @Test names + exception-
     expectation tests. `charge(-1)` asserted to throw ≈ near-proof of "amount
     must be positive". → derivation: test.
   • Code — bean-validation annotations (@NotNull,@Min,@Pattern), guards/
     conditionals that branch then throw, custom exception types (THROWS edges),
     state-machine switch/enum transitions, magic constants, boundary comparisons.
     → derivation: code.
   • Comments/Javadoc — stated intent, "must"/"cannot"/"only when" prose. Weaker
     (drifts from code). → derivation: comment.
   • Naming — validateX, isEligible, canTransitionTo signal a rule exists even
     with opaque body; low-confidence lead, not a rule on its own.

3. LIFT CANDIDATES INTO PLAIN-LANGUAGE RULES with the LLM. Per candidate, prompt
   for a single declarative `statement` from harvested evidence; LLM normalizes
   phrasing + clusters duplicate enforcements; it does NOT invent rules without a
   code/test anchor. Set:
   • derivation = the strongest evidence class backing the statement.
   • confidence = blended trust: test-anchored ≈ 0.9–1.0(excl), code-anchored
     ≈ 0.7–0.9, comment-only ≈ 0.4–0.6, LLM-paraphrase-only ≈ ≤0.4.
   • evidence = explicit citations: {file,startLine,endLine} spans and/or test
     GIDs/names. NEVER write a BusinessRule without ≥1 citation.
   Contradictions (comment "max 100" vs guard "> 1000"): record both spans, lower
   confidence, set props `conflict: true` so the planner surfaces it for human
   review rather than silently preserving the wrong invariant.

4. MAP DOMAIN ENTITIES + ALIASES — concept ≠ class. Multiple classes (Invoice,
   InvoiceDTO, InvoiceEntity, INVOICE table) often realize one DomainEntity
   "Invoice"; record variants in aliases[] and observed lifecycleStates[]
   (DRAFT→ISSUED→PAID→VOID, from enums/status fields/transitions). Link each
   implementing Class with MAPS_TO (confidence); each Method that reads/writes
   the entity with MANIPULATES (op from underlying READS/WRITES edges).

5. LINK RULES TO ENFORCEMENT + CAPABILITIES. Per BusinessRule, add IMPLEMENTS_RULE
   from each enforcing Method (carry derivation + confidence). A rule with no
   enforcing method is an orphan (see invariants) — find enforcement or drop it.
   Attach each rule's capability via its enforcing methods' components; ensure
   each Capability transitively owns its rules through REALIZES_CAPABILITY +
   IMPLEMENTS_RULE.

6. SET CAPABILITY CRITICALITY (low|med|high) from signals: external Endpoint
   exposure, money/PII DomainEntities touched, density + confidence of attached
   rules, L1 fan-in (DEPENDS_ON). Flows downstream into the risk model — high-
   criticality capabilities raise the risk score of any ChangeUnit touching them.
```

## Evidence discipline & confidence (the core of this stage)
- **Cite or it doesn't exist.** Every BusinessRule + every inferred L2 edge carries
  a source-span or test-name citation in `evidence`/`provenance`. Uncited = discard.
- **Trust ranking is non-negotiable:** test > code > comment > LLM-paraphrase.
  When several agree, prefer the higher class; when they disagree, keep the
  higher-trust statement and flag the conflict.
- **Confidence mandatory on inference.** Per ontology §3.3, every inference/LLM
  L2 edge sets `confidence < 1.0` and records `provenance.tool` (model +
  prompt-template version, pinned per run per system-design §6). No
  `confidence = 1.0` L2 facts — a passing test only proves the rule *as it
  exercises it*.
- **Don't over-claim coverage.** Rule absence means "not found," not "no such
  rule." Record honestly; the planner treats thin rule coverage as a reason to
  demand replay/characterization, not as safety.

## Graph writes (exact mapping)

| Finding | Node kind | Edges written |
|---------|-----------|---------------|
| Business capability | `Capability` | `REALIZES_CAPABILITY` (Component/Class→Capability, `confidence`) |
| Discrete rule/invariant | `BusinessRule` | `IMPLEMENTS_RULE` (Method→BusinessRule, `confidence`,`derivation`) |
| Domain concept | `DomainEntity` | `MAPS_TO` (Class→DomainEntity, `confidence`); `MANIPULATES` (Method→DomainEntity, `op`) |

All writes use node/edge envelopes (schema §2–3) with `provenance.stage =
business-discovery`, `runId`, and `provenance.tool` on every inferred fact.
Append-only: add L2; never mutate L0/L1.

## Why this gates safe transformation
The planner copies your Capabilities + BusinessRules into each ChangeUnit's
`businessRefs` and emits `PRESERVES` (ChangeUnit→BusinessRule/Capability) edges
(`modernization-ir.md` §2; `edge-types.md` §L4). Compiler + validation then must
*prove behavioral equivalence* against exactly those preserved facts. A missed
rule is a behavior that can silently break; a wrong rule makes the pipeline
"preserve" the wrong thing. Under-claim before you over-claim, and let low
confidence route a change to replay/human review.

## Quality / invariants
- **No orphan rules.** Every BusinessRule has ≥1 `IMPLEMENTS_RULE` to a Method and
  rolls up to ≥1 Capability (mirror of ontology §3.4 for L3/L5).
- **Confidence required on inference.** Every inference/LLM L2 edge sets
  `confidence < 1.0` and `provenance.tool` (ontology §3.3). Reject any L2 edge
  missing both.
- **Citation required.** Every BusinessRule has non-empty `evidence`; `derivation`
  is one of test|code|comment|llm and matches the cited evidence's strongest class.
- **Concept ≠ class.** A DomainEntity is distinct from its implementing Class(es);
  don't emit a DomainEntity that is a 1:1 rename of a single class with no
  aliases/lifecycle unless it's a genuine domain concept.
- **Cross-repo edges** (rule spans repos): set `props.crossRepo = true` and name
  the shared `contract` GID (ontology §3.5).
- **Idempotent & incremental:** re-running over an unchanged subgraph yields the
  same L2 facts; only changed-hash regions are re-inferred.

## Definition of done
L2 written for the repo: **every non-synthetic entry point has ≥1 traced `Flow`**
(zero-flow entry points raised as gaps, not skipped); every Capability has
criticality + ≥1 realizing component; every BusinessRule has a plain-language
statement, a `derivation`, a `confidence < 1.0`, ≥1 citation, ≥1 enforcing method;
DomainEntities map implementing classes + aliases; all inferred edges carry
confidence + `provenance.tool`; conflicts flagged, not hidden; planner can read
`businessRefs`/`PRESERVES` targets from the graph without re-parsing source.
Return `status=ok` with a summary (entry points & flows traced + coverage,
capabilities, rules by derivation class, mean/low-confidence counts, flagged
conflicts).
