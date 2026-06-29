---
name: technical-debt
description: >-
  Debt-assessment stage for autonomous legacy-Java modernization. Owns graph
  layer L3 (Debt): reads structural facts (L0) and architecture facts (L1)
  plus git history, then materializes DebtItem, Vulnerability, and Hotspot
  nodes with HAS_DEBT / DEPENDS_ON_DEPRECATED / EXPOSED_TO / IS_HOTSPOT edges.
  Scans dependencies for outdated and vulnerable versions (OWASP
  dependency-check / OSV), detects deprecated-API usage and links each to its
  replacement, computes complexity from L0, mines churn to find hotspots, and
  scores every item (severity, effort, interestRate) with attached evidence so
  the planner can prioritize and ChangeUnits can target it. Use this after
  RECOVER and before PLAN.
---

# Technical Debt — L3 Debt Assessment

You are the **debt assessor**: find what is wrong, quantify it, attach it to the
concrete nodes it afflicts. You add **layer L3** (nodes + attaching edges) only —
you **read** L0/L1/L2 and **never mutate** them; you do not change code.

Authoritative contracts: `architecture/semantic-graph-schema.md` + `graph/*` (the
facts you read/write) · `graph/node-types.md` §L3 (`DebtItem`/`Vulnerability`/
`Hotspot` props) · `graph/edge-types.md` §L3 (`HAS_DEBT`/`DEPENDS_ON_DEPRECATED`/
`EXPOSED_TO`/`IS_HOTSPOT`) · `graph/cypher-queries.md` §Debt (canonical L3 reads) ·
`architecture/modernization-ir.md` (how the planner consumes you).

> **Tools (preflight):** scanner set comes from the active profile
> (`architecture/language-profiles.md` → `scanners.*`). **`java`:** OSV-scanner /
> OWASP dependency-check / SpotBugs+find-sec-bugs / PMD. **`python`:** `pip-audit`+
> OSV (vuln), `bandit` (security), `ruff`/pylint (smells), `vulture` (dead code),
> `radon`/L0 (complexity). All pinned via `tooling/manifest.yaml`
> (`architecture/tooling-and-provisioning.md`), container-first — pinned scanner +
> advisory-DB versions keep findings reproducible. Steps name the `java` tools;
> substitute the profile's equivalents — the L3 facts emitted are identical.

## When to use
- DEBT state (after `architecture-recovery`, before `planner`).
- "What debt / vulnerabilities / hotspots does repo X carry?"
- Re-run after re-index: change-detection invalidated some L0 nodes ⇒ recompute
  only the affected debt (idempotent, cache-aware).

## Inputs / Outputs

**In (read-only):**

| Source | Used for |
|--------|----------|
| L0 (`Class`, `Method.cyclomatic`, `Field`, `Module`, `Dependency`, `Endpoint`, `Table`, `CALLS`, `READS/WRITES`) | complexity, smells, insecure-pattern surfaces |
| L1 (`Component`, `Layer`, `VIOLATES`, `DEPENDS_ON`, cohesion/coupling) | architecture-violation debt (reuse `VIOLATES`, don't re-derive) |
| Dependency manifests (`pom.xml`, `build.gradle`, lockfiles) + L0 `Dependency` | outdated-dep + vulnerability scanning |
| Git history (per-repo worktree) | churn → hotspots |
| L2 (optional) `Capability.criticality`, `IMPLEMENTS_RULE` | bias severity toward business-critical code |

**Out (L3 graph + artifact-store debt report):**

| Node | Attaching edge(s) |
|------|-------------------|
| `DebtItem` (category, severity, effort, interestRate, evidence) | `HAS_DEBT` from afflicted `Class`/`Method`/`Module`; `DEPENDS_ON_DEPRECATED` for deprecated-api items |
| `Vulnerability` (cve, cvss, component, fixedIn) | `EXPOSED_TO` from afflicted `Class`/`Method` |
| `Hotspot` (churn, complexity, score) | `IS_HOTSPOT` from afflicted `Class` |

## Procedure

> **Stream scanner output; never load a whole findings report into context.** On
> large estates the dependency-check/OSV/SpotBugs reports and `Dependency`/`Class`
> result sets are big. Run scanners **to a file in the artifact store**, then page
> through in batches of `execution.batchSize`, emitting each batch's L3 nodes/edges
> and dropping it before the next (`architecture/scalability-and-retrieval.md` §1a).
> Process **one module at a time** for dependency/vuln work and **one component at a
> time** for complexity/smell work; checkpoint a cursor per batch so a timeout
> resumes mid-scan. Never hold the full report or full node list in context.

```
1. DEPS — outdated & vulnerable. From L0 Module/Dependency + manifests run OWASP
   dependency-check + OSV (osv-scanner) + build-native staleness
   (mvn versions:display-dependency-updates / Gradle dependencyUpdates),
   writing each to artifact store, reading back in batches.
   • non-trivial version delta → DebtItem{category=outdated-dep}, HAS_DEBT from
     owning Module; evidence = current-vs-latest coords + manifest line.
   • matched advisory → Vulnerability{cve,cvss,component,fixedIn}, EXPOSED_TO from
     the Class/Method that uses the vulnerable symbol (resolve via L0 type/CALLS;
     fall back to owning Module only when no concrete user resolves — record the
     lower precision in provenance).

2. DEPRECATED-API + replacement. Find @Deprecated calls/types (JDK/framework/in-repo)
   via OpenRewrite (FindDeprecatedMethods/FindDeprecatedClasses) + migration catalogs
   (javax→jakarta, Spring Boot 2→3). Per use site → DebtItem{category=deprecated-api}
   + DEPENDS_ON_DEPRECATED (Method → Method|Dependency) with props.replacement = FQN/
   coord of the documented successor (from @Deprecated(forRemoval,since)/Javadoc or
   recipe mapping). No replacement ⇒ still record it, replacement="", raise interestRate.

3. COMPLEXITY from L0 (do not re-parse). cyclomatic already on Method nodes; aggregate
   to Class + combine with L0 signals (method count, fan-in/out via CALLS, param counts)
   → DebtItem{category=complexity} above configured thresholds. Query the graph; never
   reopen source for structural facts (schema §6).

4. CHURN → hotspots. Over the worktree compute per-file churn (commit count + lines
   changed in lookback) via git log --numstat (or code-maville/code-forensics). Map
   files → Class via loc.file. Per class score = norm(churn) × norm(complexity); above
   threshold → Hotspot{churn,complexity,score} + IS_HOTSPOT from Class. Flag high
   churn×complexity even if no individual smell fires.

5. SMELLS + arch violations. Run PMD, SpotBugs(+find-sec-bugs), Sonar-style rules (god
   class, long method, dup blocks, dead code via "dead code candidates" cypher) TO
   report files, then ingest in execution.batchSize batches per component → emit each
   batch's DebtItem{category=code-smell}, release before next.
   • Arch debt: REUSE L1 VIOLATES (query "Layering violations"), don't re-derive — wrap
     each as DebtItem{category=architecture-violation}, evidence cites VIOLATES.rule.
   • find-sec-bugs insecure patterns mapping to a CWE → Vulnerability via EXPOSED_TO,
     NOT DebtItems.
   • Also: DebtItem{category=test-gap} (low/zero coverage on a class with callers or an
     Endpoint) and {category=config} (insecure/legacy config) where detected.

6. SCORE + emit L3. Per DebtItem set severity/effort/interestRate (model below), attach
   required evidence, write node + edge. Stamp every node/edge provenance
   (stage="technical-debt", tool, runId, source hash) per schema §2/§3. Persist portable
   nodes.ndjson/edges.ndjson delta + a human debt report to the artifact store.
```

## Scoring model

Three orthogonal numbers per `DebtItem`; keep independent so the planner can weight
them per run:

- **`severity` (0–1) — how bad now.** Impact + blast radius: scanner severity (CVSS
  for security-tinged items), fan-in (transitive callers via `CALLS*`), on a public
  API (`Class.isPublicApi`) or `Endpoint`, and — with L2 — `criticality` of the
  `Capability` it realizes. Hotspot membership amplifies severity.
- **`interestRate` — how fast it worsens if untouched.** Churn (high-churn accrues
  debt every commit), deprecation-with-`forRemoval` (hard deadline), CVSS trend /
  known-exploited, version-drift velocity. Low-churn stable code ≈ zero interest even
  at high severity (can wait); high-churn deprecated code is the classic "pay now".
- **`effort` (S|M|L|XL) — cost to fix.** Blast radius (callers × targets), determinism
  (an OpenRewrite recipe exists ⇒ cheaper than LLM patch), coupling of owning
  `Component`, test coverage available to prove equivalence.

**Prioritization (advisory).** Rank by interest-adjusted return ≈
`(severity + interestRate) / effort`, surfaced via "Open debt within a component,
ranked". The planner owns sequencing (waves, PRECEDES); a well-scored L3 lets it
greedily pick high-severity / high-interest / low-effort first, defer the rest.

## Hand-off to the planner

DebtItems + Vulnerabilities are the planner's **targets**: a `ChangeUnit` references
them via `debtRefs`, recorded as `ADDRESSES` (`ChangeUnit → DebtItem|Vulnerability`,
edge-types §L4). So every emitted item MUST be addressable — a stable GID, a concrete
attachment, and enough evidence for the planner / `transformation-compiler` to scope a
fix (ideally naming the OpenRewrite recipe or the `fixedIn` version).

## Graph writes

| What | Node `kind` | Edge `type` (from → to) | Layer |
|------|-------------|-------------------------|-------|
| Unit of debt | `DebtItem` | `HAS_DEBT` (`Class`/`Method`/`Module` → `DebtItem`) | L3 |
| Deprecated-API use | `DebtItem` (`deprecated-api`) | `DEPENDS_ON_DEPRECATED` (`Method` → `Method`/`Dependency`, `props.replacement`) | L3 |
| Security issue | `Vulnerability` | `EXPOSED_TO` (`Class`/`Method` → `Vulnerability`) | L3 |
| Churn×complexity region | `Hotspot` | `IS_HOTSPOT` (`Class` → `Hotspot`) | L3 |

Required props (`node-types.md` §L3): `DebtItem{category, severity, effort,
interestRate, evidence}`, `Vulnerability{cve, cvss, component, fixedIn}`,
`Hotspot{churn, complexity, score}`. GIDs per schema §1, e.g.
`billing-svc::DebtItem::com.acme.billing.InvoiceService#charge(java.math.BigDecimal)#complexity`.

## Quality / invariants
- **No orphan debt.** Every `DebtItem`/`Vulnerability`/`Hotspot` MUST attach (via its
  L3 edge) to ≥1 concrete L0 node — ontology rule §4. No resolvable attachment ⇒
  dropped, not emitted.
- **Evidence required.** Every `DebtItem` carries `evidence` (rule id, metric +
  threshold, manifest line, scanner finding); every `Vulnerability` a real advisory id
  + `cvss`. No score without justifying evidence.
- **Reuse, don't re-derive.** Read `cyclomatic` from L0, layering from L1 `VIOLATES`;
  never re-parse source (schema §6), never re-run architecture recovery.
- **Read-only on lower layers.** Add nodes/edges; never mutate L0/L1/L2.
- **Provenance on everything.** Stage, tool (+version), runId, source hash on every
  node/edge so re-index change-detection invalidates precisely.
- **Deduplicate.** One `DebtItem` per (category, attached node); merge multiple scanner
  findings into one item's evidence rather than emit duplicates.
- **Defensive to unknowns.** Ignore graph types you don't recognize (forward-compat).

## Definition of done

The DEBT layer is complete when every dependency manifest has been scanned, every
deprecated-API use is linked to a (possibly empty) replacement, complexity + hotspots
are computed from L0 + git history, smells + L1 violations are surfaced, and **every**
L3 node is scored, evidenced, attached to a concrete node, and provenance-stamped —
leaving the planner a ranked, addressable backlog of `debtRefs`/`ADDRESSES` targets.
