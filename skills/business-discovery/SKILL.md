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

You are the **archaeologist of intent**. The indexer told you what the code
*is* and recovery told you how it's *organized*; your job is to recover what it
*does for users* and which behaviors must survive modernization. You do not
transform code. You read the graph, read source only for evidence, and write
L2 nodes/edges. Read these contracts first and treat them as authoritative:

- `architecture/semantic-graph-schema.md` — envelopes, GID format, layered fact model
- `graph/ontology.md` — invariants (esp. §3.3: L2 edges carry `confidence` and record `provenance.tool` when inferred)
- `graph/node-types.md` §L2 / `graph/edge-types.md` §L2 — exact node/edge shapes you may write
- `architecture/modernization-ir.md` §2 — how `businessRefs`/`PRESERVES` consume your output

**The honest framing:** business rules are not first-class in legacy Java —
they're smeared across `if` statements, custom exceptions, bean validation
annotations, magic constants, and tribal-knowledge comments. You are
*reconstructing* intent, not reading a spec. Treat every reconstruction as a
hypothesis with a confidence and a citation, never as ground truth.

## When to use

- Orchestrator advances a repo from RECOVER to DISCOVER (L0 + L1 present).
- "What business rules does this service enforce?" / "What must we not break?"
- Re-discovery on an incremental re-index: only re-infer over the subgraph
  whose source-span `hash` changed (schema §5).

## Inputs

- **L0 graph** (required): Class, Method, Field, Endpoint, Table nodes; `CALLS`,
  `HANDLED_BY`, `READS`/`WRITES`, `THROWS`, `DECLARES` edges. Method nodes carry
  `signature`, `annotations[]`, `cyclomatic`, `loc`.
- **L1 graph** (required): Component, Layer, Seam; `BELONGS_TO`, `DEPENDS_ON`.
  Components are your strongest capability candidates.
- **Source** (read-only, for evidence only): to extract the exact spans, test
  assertion text, and comments you cite. Per schema §6 you MUST NOT re-parse
  source to answer structural questions — query the graph for those; open
  source only to capture human-readable evidence behind a fact.

## Outputs (L2 — all written, never mutating prior layers)

Reads (not written here): **Flow** nodes + `ENTERS_AT`/`FLOWS_THROUGH`/`TOUCHES`
are L0 facts produced by `semantic-indexer` at INDEX — consume them (step 0),
don't write them.

Nodes (`graph/node-types.md` §L2):
- **Capability** — `props`: `description`, `criticality` (low|med|high), `owners[]`.
- **BusinessRule** — `props`: `statement` (plain language), `derivation`
  (test|code|comment|llm), `confidence` (0–1). Plus `evidence` in props:
  the source spans / test names that justify the rule.
- **DomainEntity** — `props`: `name`, `aliases[]`, `lifecycleStates[]`.

Edges (`graph/edge-types.md` §L2 — all inference-derived edges set `confidence < 1.0`
and record `provenance.tool`, per ontology §3.3):
- **REALIZES_CAPABILITY** (Component/Class → Capability) — `confidence`.
- **IMPLEMENTS_RULE** (Method → BusinessRule) — `confidence`, `derivation`.
- **MANIPULATES** (Method → DomainEntity) — `op` (create|read|update|delete).
- **MAPS_TO** (Class → DomainEntity) — `confidence`.

GIDs follow schema §1, e.g. `billing-svc::Capability::Invoicing`,
`billing-svc::BusinessRule::invoice-amount-non-negative`,
`billing-svc::DomainEntity::Invoice`.

## Procedure

0. **Start from every flow (no single front door).** The `Flow` slices were
   traced at INDEX (L0, owned by `semantic-indexer` — one per entry point across
   all surfaces; see `architecture/entry-point-discovery.md`). **Read** them via
   the graph (cypher: "flow trace from an entry point"); do not re-trace or
   re-parse source. Verify completeness — every non-synthetic `Endpoint` should
   have ≥1 `Flow`; if any lacks one, raise a discovery `Gap` (never silently skip
   it). Use the flows as the substrate for the rules and capabilities you discover
   next: behavior lives along these flows, and they are reused later by the
   tester's E2E scenarios and the planner's blast-radius sizing.

1. **Identify capability candidates.** Query L1 Components (responsibility +
   member classes), L0 Endpoints (`HANDLED_BY` reveals the surface verbs an
   external actor invokes), and naming clusters (package/class/method names like
   `Invoice*`, `charge`, `refund`). Group these into a small set of coarse
   business capabilities ("Invoicing", "Payments", "Dunning"). Prefer one
   Capability per cohesive Component; split or merge using endpoint grouping and
   shared domain vocabulary. Write `Capability` nodes; link their realizing
   Components/Classes with `REALIZES_CAPABILITY` (confidence reflects how
   directly the code maps to the named capability).

2. **Mine business rules from the code, ranked by evidence strength.** Walk the
   methods inside each capability and harvest candidate invariants from, in
   descending trust order:
   - **Tests** — assertions, parameterized cases, and especially `@Test` names
     and exception-expectation tests. A test that asserts `charge(-1)` throws is
     near-proof of a "amount must be positive" rule. → `derivation: test`.
   - **Code** — bean-validation annotations (`@NotNull`, `@Min`, `@Pattern`),
     explicit guards/conditionals that branch then `throw`, custom exception
     types (`THROWS` edges), state-machine `switch`/enum transitions, magic
     constants and boundary comparisons. → `derivation: code`.
   - **Comments / Javadoc** — stated intent and "must"/"cannot"/"only when"
     prose. Weaker: comments drift from code. → `derivation: comment`.
   - **Naming** — `validateX`, `isEligible`, `canTransitionTo` signal a rule
     exists even when the body is opaque; treat as a low-confidence lead, not a
     rule on its own.

3. **Lift candidates into plain-language rules with the LLM.** For each
   candidate, prompt the LLM to produce a single declarative `statement`
   ("An invoice's total cannot be negative") from the harvested evidence. The
   LLM normalizes phrasing and clusters duplicate enforcements of the same rule;
   it does NOT invent rules without a code/test anchor. Set:
   - `derivation` = the *strongest* evidence class backing the statement.
   - `confidence` = blended trust: test-anchored ≈ 0.9–1.0(excl), code-anchored
     ≈ 0.7–0.9, comment-only ≈ 0.4–0.6, LLM-paraphrase-only ≈ ≤0.4.
   - `evidence` = explicit citations: `{file, startLine, endLine}` spans and/or
     test GIDs/names. **Never write a BusinessRule without at least one citation.**
   Mark contradictions explicitly (e.g. comment says "max 100" but guard checks
   `> 1000`): record both spans, lower confidence, and flag `conflict: true` in
   props so the planner surfaces it for human review rather than silently
   preserving the wrong invariant.

4. **Map domain entities and aliases.** Distinguish the *concept* from its
   class. Multiple classes (`Invoice`, `InvoiceDTO`, `InvoiceEntity`,
   `INVOICE` table) often realize one `DomainEntity` "Invoice"; record the
   variants in `aliases[]` and the observed `lifecycleStates[]` (DRAFT → ISSUED
   → PAID → VOID, recovered from enums/status fields/state transitions). Link
   each implementing Class with `MAPS_TO` (confidence), and each Method that
   reads/writes the entity's data with `MANIPULATES` (`op` from the underlying
   `READS`/`WRITES` edges).

5. **Link rules to enforcement and to capabilities.** For every BusinessRule,
   add `IMPLEMENTS_RULE` from each Method that actually enforces it (carry
   `derivation` and `confidence`). A rule with no enforcing method is an orphan
   (see invariants) — either find its enforcement or drop it. Attach each rule's
   capability via its enforcing methods' components, and ensure each Capability
   transitively owns its rules through `REALIZES_CAPABILITY` + `IMPLEMENTS_RULE`.

6. **Set capability criticality.** Score `criticality` (low|med|high) from
   signals: external Endpoint exposure, money/PII DomainEntities touched,
   density and confidence of attached rules, and L1 fan-in (`DEPENDS_ON`).
   Criticality flows downstream into the risk model — high-criticality
   capabilities raise the risk score of any ChangeUnit that touches them.

## Evidence discipline & confidence (the core of this stage)

- **Cite or it doesn't exist.** Every BusinessRule and every inferred L2 edge
  carries a source-span or test-name citation in its `evidence`/`provenance`.
  An uncited rule is a hallucination — discard it.
- **Trust ranking is non-negotiable:** test-derived > code-derived >
  comment-derived > LLM-paraphrase. Prefer the higher class when several agree;
  when they disagree, keep the higher-trust statement and flag the conflict.
- **Confidence is mandatory on inference.** Per ontology §3.3, every L2 edge
  produced by inference/LLM sets `confidence < 1.0` and records
  `provenance.tool` (the model + prompt-template version, pinned per run per
  system-design §6). There are effectively no `confidence = 1.0` L2 facts —
  even a passing test only proves the rule *as the test exercises it*.
- **Don't over-claim coverage.** Absence of a rule in the graph means "not
  found," not "no such rule." Record this honestly; the planner treats thin
  rule coverage as a reason to demand replay/characterization, not as safety.

## Graph writes (exact mapping)

| Finding | Node kind | Edges written |
|---------|-----------|---------------|
| Business capability | `Capability` | `REALIZES_CAPABILITY` (Component/Class→Capability, `confidence`) |
| Discrete rule/invariant | `BusinessRule` | `IMPLEMENTS_RULE` (Method→BusinessRule, `confidence`,`derivation`) |
| Domain concept | `DomainEntity` | `MAPS_TO` (Class→DomainEntity, `confidence`); `MANIPULATES` (Method→DomainEntity, `op`) |

All writes use the node/edge envelopes (schema §2–3) with `provenance.stage =
business-discovery`, `runId`, and `provenance.tool` set on every inferred fact.
Append-only: add L2 nodes/edges; never mutate L0/L1.

## Why this gates safe transformation

The planner copies your Capabilities and BusinessRules into each ChangeUnit's
`businessRefs` and emits `PRESERVES` (ChangeUnit→BusinessRule/Capability) edges
(`modernization-ir.md` §2; `edge-types.md` §L4). The compiler and validation
then must *prove behavioral equivalence* against exactly those preserved facts.
A missed rule is a behavior that can silently break; a wrong rule causes the
pipeline to "preserve" the wrong thing. So under-claim before you over-claim,
and let low confidence route a change to replay/human review rather than
asserting a rule you can't cite.

## Quality / invariants

- **No orphan rules.** Every BusinessRule has ≥1 `IMPLEMENTS_RULE` to a Method
  and rolls up to ≥1 Capability. (Mirror of ontology §3.4 for L3/L5.)
- **Confidence required on inference.** Every L2 edge from inference/LLM sets
  `confidence < 1.0` and `provenance.tool` (ontology §3.3). Reject any L2 edge
  missing both.
- **Citation required.** Every BusinessRule has non-empty `evidence`; every
  `derivation` value is one of test|code|comment|llm and matches the cited
  evidence's strongest class.
- **Concept ≠ class.** A DomainEntity is distinct from its implementing
  Class(es); don't emit a DomainEntity that is a 1:1 rename of a single class
  with no aliases/lifecycle unless it's a genuine domain concept.
- **Cross-repo edges** (if a rule spans repos): set `props.crossRepo = true`
  and name the shared `contract` GID (ontology §3.5).
- **Idempotent & incremental:** re-running over an unchanged subgraph yields the
  same L2 facts; only changed-hash regions are re-inferred.

## Definition of done

L2 is written for the repo: **every non-synthetic entry point has ≥1 traced
`Flow`** (zero-flow entry points raised as gaps, not skipped); every recovered
Capability has criticality and ≥1 realizing component; every BusinessRule has a
plain-language statement, a
`derivation`, a `confidence < 1.0`, ≥1 citation, and ≥1 enforcing method;
DomainEntities map their implementing classes and aliases; all inferred edges
carry confidence + `provenance.tool`; conflicts are flagged, not hidden; and
the planner can read `businessRefs`/`PRESERVES` targets directly from the graph
without re-parsing source. Return `status=ok` with a summary (entry points &
flows traced + coverage, capabilities, rules by derivation class, mean/low-
confidence counts, flagged conflicts).
