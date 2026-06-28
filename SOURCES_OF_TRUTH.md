# Sources of Truth

An autonomous system must know, unambiguously, *which artifact wins* when two
disagree. This document defines the authoritative sources, their precedence,
and how conflicts are resolved. Every skill is bound by this hierarchy.

There are **three kinds** of truth in this system, and conflating them is the
classic failure mode:

| Kind | Question it answers | Authoritative artifact |
|------|---------------------|------------------------|
| **Intent** | What do we *want*? | `input/<run>.md` (Target Spec) |
| **Reality** | What *is* the code today? | The **source in each repo** (git) |
| **Rules** | How does *this system itself* work? | `architecture/` + `graph/` contracts |

Everything else — the semantic graph, the Modernization IR, diffs, reports — is
**derived** and therefore *not* a source of truth. Derived artifacts are caches
or projections; if one disagrees with its source, the source wins and the
derived artifact is regenerated.

---

## 1. Authority precedence (highest → lowest)

When two artifacts conflict, the higher one prevails.

1. **Human approval / gate decisions** (run-time)
   A human pausing, blocking, or approving a ChangeUnit overrides any
   automated decision. Recorded as gate events.

2. **Intent — the Target Spec** (`input/<run>.md`)
   *Authoritative for goals, constraints, behavior-to-preserve, autonomy
   ceilings, and acceptance criteria.* The objective and `constraints` here are
   hard; the planner may not violate them. If the spec is silent, defaults in
   `workflows/modernization.yaml` apply.

3. **Reality — repository source code** (git, the pinned commit)
   *Authoritative for what the system currently does.* No analysis output may
   override observed source/behavior. If the graph says X and the code says Y,
   **the code is right** and the graph is reindexed.

4. **System contracts** (`architecture/` + `graph/`)
   *Authoritative for how the autopilot is built* — the IR format, the state
   machine, the graph schema/ontology. Skills conform to these; if a skill's
   behavior diverges from a contract, the contract wins (fix the skill).

5. **Skill instructions** (`skills/*/SKILL.md`) and **prompts** (`prompts/*`)
   Authoritative for *how a given step is performed*, within the contracts above.

6. **Derived artifacts** (NOT sources of truth — regenerable):
   semantic graph (L0–L5), Modernization IR, candidate diffs, validation/replay
   reports, risk scores, run reports.

> Rule of thumb: **Intent > Reality > Contracts > Derived.** A run exists to
> move *Reality* toward *Intent* without violating *Contracts*, leaving an
> auditable trail of *Derived* facts.

---

## 2. The two "source of truth" extremes you must not confuse

- **Source code is the source of truth for behavior.** The semantic graph is a
  *queryable projection* of the code, kept fresh by content-hash invalidation
  (`architecture/semantic-graph-schema.md` §5). Never treat the graph as ground
  truth — when in doubt, re-derive from source.

- **The Target Spec is the source of truth for intent.** The Modernization IR is
  a *derived plan* that operationalizes the spec. If the IR conflicts with the
  spec (e.g. proposes a change a constraint forbids), the IR is wrong and the
  planner must re-plan. The IR never silently overrides the spec.

---

## 3. Per-layer authority in the semantic graph

Within the derived graph, each layer has exactly one **producing skill** that is
authoritative for that layer; no other skill may mutate it (they add their own
layer instead — `architecture/semantic-graph-schema.md` §4).

| Layer | Authoritative producer | Derived from |
|-------|------------------------|--------------|
| L0 Structural | `semantic-indexer` | repo source (Reality) |
| L1 Architectural | `architecture-recovery` | L0 |
| L2 Business | `business-discovery` | L0, L1, tests, comments |
| L3 Debt | `technical-debt` | L0, L1, deps, git history |
| L4 Plan | `planner` | L0–L3 + Target Spec (Intent) |
| L5 Risk | `risk-assessment` | L4 + validation/replay reports |

A consumer that needs an L1 fact reads it; it never recomputes or overrides it.

---

## 4. Conflict resolution rules

| Conflict | Resolution |
|----------|------------|
| Graph fact vs. source code | Source wins; reindex the affected subgraph. |
| IR plan vs. Target Spec constraint | Spec wins; planner re-plans the offending ChangeUnit. |
| Two skills want to edit the same graph layer | Only the layer's producer may write; others add their own layer. |
| Inferred (LLM/heuristic) fact vs. resolved fact | Resolved (`confidence=1.0`) wins; inferred facts carry `confidence<1.0` and must record `provenance.tool`. |
| Test-derived rule vs. comment/LLM-derived rule | Higher-confidence derivation wins (`test > code > comment > llm`). |
| Candidate diff (incl. LLM) vs. behavior-to-preserve | Preserve list wins; diff is rejected/blocked unless §4 of the spec authorizes the change. |
| Cross-repo contract edit, producer vs. consumer | Serialize via the contract lock; both change together or neither does. |
| Automated apply vs. human gate | Human gate wins, always. |
| Open question with no answer in any source | Resolve down the ladder; if still unknown, **placeholder + Gap**, never skip. |
| Placeholder vs. later-provided real interface | Real interface wins; it `SUPERSEDES` the placeholder and dependents re-validate. |

### Resolving open questions & unknowns

The system answers its own open questions by walking this same precedence
hierarchy top-down (human decision → Intent → Reality → Contracts → reasoned
inference), recording how each was answered. When nothing answers and the
unknown is an **external interface/dependency**, it creates a **typed
placeholder + a tracked Gap** and builds the integration against it — it does
**not** silently skip the integration. Human escalation (`NEEDS_HUMAN`) is the
last resort, only for unknowns that are both *blocking* and *unsafe to assume*.
Full procedure: `architecture/open-question-resolution.md`.

---

## 5. Provenance makes truth auditable

Every node, edge, and diff records `provenance` (stage, tool, model+template
version, runId, source-span hash). This lets any derived fact be traced back to
its source of truth and reproduced (`skills/replay`), satisfying the audit
requirement in `architecture/deployment-architecture.md` §6. If you can't trace
a fact to an authoritative source, treat it as untrusted.

---

## 6. Quick reference

```
WANT  ─ Target Spec (input/<run>.md) ─────────────► drives the run
IS    ─ Repo source (git @ pinned commit) ────────► ground truth for behavior
RULES ─ architecture/ + graph/ contracts ─────────► how the autopilot works
HOW   ─ skills/*/SKILL.md + prompts/* ────────────► how each step runs
CACHE ─ graph, IR, diffs, reports ────────────────► derived; regenerate on doubt
```
