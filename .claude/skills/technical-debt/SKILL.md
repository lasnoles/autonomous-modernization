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

You are the **debt assessor**. You do not change code; you find what is wrong,
quantify it, and attach it to the concrete nodes it afflicts so the planner can
decide what is worth fixing and in what order. Everything you produce is a
graph fact in **layer L3**. Read these contracts before acting and treat them
as authoritative:

- `architecture/semantic-graph-schema.md` + `graph/*` — the facts you read/write
- `graph/node-types.md` §L3 — `DebtItem`, `Vulnerability`, `Hotspot` props
- `graph/edge-types.md` §L3 — `HAS_DEBT`, `DEPENDS_ON_DEPRECATED`, `EXPOSED_TO`, `IS_HOTSPOT`
- `graph/cypher-queries.md` §Debt — the canonical L3 read queries
- `architecture/modernization-ir.md` — how the planner consumes your output

You **read** lower, **never mutate** lower. Per the layered fact model you add
the L3 layer (nodes + attaching edges) and never edit L0/L1/L2 nodes in place.

> **Tools (preflight):** the scanner set comes from the active language profile
> (`architecture/language-profiles.md` → `scanners.*`). **`java`:** OSV-scanner /
> OWASP dependency-check / SpotBugs+find-sec-bugs / PMD. **`python`:** `pip-audit`
> + OSV (vuln), `bandit` (security), `ruff`/pylint (smells), `vulture` (dead
> code), `radon`/L0 (complexity). All at pinned versions from `tooling/manifest.yaml`
> (`architecture/tooling-and-provisioning.md`), container-first. Pinned scanner +
> advisory-DB versions keep findings reproducible. Steps below name the `java`
> tools; substitute the profile's equivalents — the L3 facts emitted are identical.

## When to use

- DEBT state of the repo machine (after `architecture-recovery`, before `planner`).
- "What debt / vulnerabilities / hotspots does repo X carry?"
- Re-run after a re-index: change detection invalidated some L0 nodes, so only
  the affected debt must be recomputed (idempotent, cache-aware).

## Inputs / Outputs

**Inputs (read-only):**

| Source | Used for |
|--------|----------|
| L0 graph (`Class`, `Method.cyclomatic`, `Field`, `Module`, `Dependency`, `Endpoint`, `Table`, `CALLS`, `READS/WRITES`) | complexity, smells, insecure-pattern surfaces |
| L1 graph (`Component`, `Layer`, `VIOLATES`, `DEPENDS_ON`, cohesion/coupling) | architecture-violation debt (reuse `VIOLATES`, do not re-derive) |
| Dependency manifests (`pom.xml`, `build.gradle`, lockfiles) + L0 `Dependency` nodes | outdated-dep and vulnerability scanning |
| Git history (per-repo worktree) | churn for hotspot computation |
| L2 (optional) `Capability.criticality`, `IMPLEMENTS_RULE` | bias severity toward business-critical code |

**Outputs (L3, written to the graph + an artifact-store debt report):**

| Node | Attaching edge(s) |
|------|-------------------|
| `DebtItem` (category, severity, effort, interestRate, evidence) | `HAS_DEBT` from the afflicted `Class`/`Method`/`Module`; `DEPENDS_ON_DEPRECATED` for deprecated-api items |
| `Vulnerability` (cve, cvss, component, fixedIn) | `EXPOSED_TO` from the afflicted `Class`/`Method` |
| `Hotspot` (churn, complexity, score) | `IS_HOTSPOT` from the afflicted `Class` |

## Procedure

> **Stream scanner output; never load a whole findings report into context.** On
> large estates the dependency-check / OSV / SpotBugs reports and the
> `Dependency`/`Class` result sets are big. Run scanners to a **file in the
> artifact store**, then page through that report in batches of
> `execution.batchSize`, emitting each batch's L3 nodes/edges and dropping it
> before the next (`architecture/scalability-and-retrieval.md` §1a). Process **one
> module at a time** for dependency/vuln work and **one component at a time** for
> complexity/smell work, checkpointing a cursor per batch so a timeout resumes
> mid-scan. Never hold the full report or the full node list in context.

1. **Scan dependencies for outdated & vulnerable versions.** From the L0
   `Module`/`Dependency` nodes and the on-disk manifests, run
   **OWASP dependency-check** and the **OSV** database (`osv-scanner`) — and the
   build-native staleness check (`mvn versions:display-dependency-updates` /
   Gradle `dependencyUpdates`) — **writing each scanner's output to the artifact
   store, then reading it back in batches** (do not inline a multi-thousand-line
   report into context).
   - Each library behind a non-trivial version delta → `DebtItem`
     `category="outdated-dep"`, attached `HAS_DEBT` to its owning `Module`,
     `evidence` = current vs latest coordinates and the manifest line.
   - Each matched advisory → a `Vulnerability` node (`cve`, `cvss`, `component`,
     `fixedIn`) with `EXPOSED_TO` from the `Class`/`Method` that actually uses
     the vulnerable symbol (resolve via L0 type/`CALLS` usage; fall back to the
     owning `Module` only when no concrete user can be resolved, and record the
     lower precision in provenance).

2. **Detect deprecated-API usage and link the replacement.** Find calls/types
   carrying `@Deprecated` (JDK, framework, or in-repo) using **OpenRewrite**
   recipes (`FindDeprecatedMethods` / `FindDeprecatedClasses`) and the migration
   catalogs (e.g. javax→jakarta, Spring Boot 2→3). For each use site emit a
   `DebtItem` `category="deprecated-api"` and a `DEPENDS_ON_DEPRECATED` edge
   `Method → Method|Dependency` with `props.replacement` = the FQN/coordinate of
   the documented successor (from `@Deprecated(forRemoval, since)` Javadoc or the
   recipe's mapping). No replacement found ⇒ still record it, replacement="" and
   raise interestRate.

3. **Compute complexity from L0 (do not re-parse).** `cyclomatic` already lives
   on `Method` nodes (semantic-indexer). Aggregate to `Class` level and combine
   with L0-derivable signals (method count, fan-in/fan-out via `CALLS`, param
   counts) to flag `category="complexity"` `DebtItem`s above the configured
   thresholds. Query the graph; never reopen source for structural facts
   (schema §6).

4. **Mine git history for churn → hotspots.** Over the worktree compute per-file
   churn (commit count and lines changed in the lookback window) with
   `git log --numstat` (or **code-maville**/`code-forensics`). Map files to
   `Class` nodes via `loc.file`. For each class compute
   `score = norm(churn) × norm(complexity)` and, above threshold, emit a
   `Hotspot` (`churn`, `complexity`, `score`) with `IS_HOTSPOT` from the `Class`.
   High churn × high complexity is where change risk concentrates — flag it even
   if no individual smell fires.

5. **Detect code smells & architecture violations.** Run **PMD**, **SpotBugs**
   (+ **find-sec-bugs** for insecure code patterns), and **Sonar**-style rules
   for smells (god class, long method, duplicated blocks, dead code via the
   "dead code candidates" cypher) **to report files, then ingest those reports in
   `execution.batchSize` batches per component** — emit each batch's
   `category="code-smell"` `DebtItem`s and release it before reading the next.
   For architecture debt, **reuse L1 `VIOLATES`** (query "Layering violations")
   rather than re-deriving layering — wrap each violation as a
   `category="architecture-violation"` `DebtItem` whose evidence cites the
   `VIOLATES.rule`. Insecure patterns from find-sec-bugs that map to a weakness
   (CWE) become `Vulnerability` nodes via `EXPOSED_TO`, not DebtItems.
   Also emit `category="test-gap"` (low/zero coverage on a class with callers or
   an `Endpoint`) and `category="config"` (insecure/legacy config) where detected.

6. **Score, attach evidence, and emit L3.** For every `DebtItem` set `severity`,
   `effort`, and `interestRate` (model below), attach required `evidence`, and
   write the node plus its edge. Stamp every node/edge `provenance`
   (`stage="technical-debt"`, `tool`, `runId`, source `hash`) per schema §2/§3.
   Persist the portable `nodes.ndjson`/`edges.ndjson` delta and a human debt
   report to the artifact store.

## Scoring model

Each `DebtItem` carries three orthogonal numbers; keep them independent so the
planner can weight them per run:

- **`severity` (0–1) — how bad it is now.** Driven by impact and blast radius:
  scanner severity (CVSS for security-tinged items), fan-in (transitive callers
  via `CALLS*`), whether it sits on a public API (`Class.isPublicApi`) or an
  `Endpoint`, and — when L2 is present — the `criticality` of the `Capability`
  it realizes. Hotspot membership amplifies severity.
- **`interestRate` — how fast it worsens if untouched.** Driven by churn (a
  high-churn file accrues debt every commit), deprecation-with-`forRemoval`
  (a hard deadline), CVSS trend / known-exploited status, and version drift
  velocity. Low-churn, stable code has near-zero interest even at high severity
  (it can wait); high-churn deprecated code is the classic "pay now" item.
- **`effort` (S|M|L|XL) — cost to fix.** Estimated from blast radius (callers ×
  targets), determinism of the fix (an OpenRewrite recipe exists ⇒ cheaper than
  an LLM patch), coupling of the owning `Component`, and test coverage available
  to prove equivalence.

**Prioritization for the planner.** Rank by an interest-adjusted return, roughly
`(severity + interestRate) / effort`, surfaced via the "Open debt within a
component, ranked" query. This is advisory: the planner owns sequencing (waves,
PRECEDES), but a well-scored L3 lets it greedily pick high-severity / high-
interest / low-effort items first and defer expensive low-interest ones.

## Hand-off to the planner

DebtItems and Vulnerabilities are the **targets** the planner addresses. In the
Modernization IR a `ChangeUnit` references them via its `debtRefs` and the graph
records this as `ADDRESSES` (`ChangeUnit → DebtItem|Vulnerability`, edge-types
§L4). So: every item you emit must be addressable — a stable GID, a concrete
attachment, and enough evidence that the planner and `transformation-compiler`
can scope a fix (ideally naming the OpenRewrite recipe or the `fixedIn` version).

## Graph writes

| What | Node `kind` | Edge `type` (from → to) | Layer |
|------|-------------|-------------------------|-------|
| Unit of debt | `DebtItem` | `HAS_DEBT` (`Class`/`Method`/`Module` → `DebtItem`) | L3 |
| Deprecated-API use | `DebtItem` (`deprecated-api`) | `DEPENDS_ON_DEPRECATED` (`Method` → `Method`/`Dependency`, `props.replacement`) | L3 |
| Security issue | `Vulnerability` | `EXPOSED_TO` (`Class`/`Method` → `Vulnerability`) | L3 |
| Churn×complexity region | `Hotspot` | `IS_HOTSPOT` (`Class` → `Hotspot`) | L3 |

Required props (see `node-types.md` §L3): `DebtItem{category, severity,
effort, interestRate, evidence}`, `Vulnerability{cve, cvss, component, fixedIn}`,
`Hotspot{churn, complexity, score}`. GIDs follow schema §1, e.g.
`billing-svc::DebtItem::com.acme.billing.InvoiceService#charge(java.math.BigDecimal)#complexity`.

## Quality / invariants

- **No orphan debt.** Every `DebtItem`, `Vulnerability`, and `Hotspot` MUST
  attach (via its L3 edge) to at least one concrete L0 node — ontology rule §4.
  An item with no resolvable attachment is dropped, not emitted.
- **Evidence required.** Every `DebtItem` carries `evidence` (rule id, metric +
  threshold, manifest line, scanner finding) and every `Vulnerability` a real
  advisory id + `cvss`. No score is emitted without the evidence that justifies it.
- **Reuse, don't re-derive.** Read `cyclomatic` from L0 and layering from L1
  `VIOLATES`; never re-parse source (schema §6) and never re-run architecture
  recovery.
- **Read-only on lower layers.** Add edges/nodes; never mutate L0/L1/L2 nodes.
- **Provenance on everything.** Stage, tool (+ version), runId, source hash on
  every node and edge, so re-index change-detection can invalidate precisely.
- **Deduplicate.** One `DebtItem` per (category, attached node); merge multiple
  scanner findings into one item's evidence rather than emitting duplicates.
- **Defensive to unknowns.** Ignore graph types you don't recognize (forward-compat).

## Definition of done

The DEBT layer for the repo is complete when every dependency manifest has been
scanned, every deprecated-API use is linked to a (possibly empty) replacement,
complexity and hotspots are computed from L0 + git history, smells and L1
violations are surfaced, and **every** L3 node is scored, evidenced, attached to
a concrete node, and provenance-stamped — leaving the planner a ranked,
addressable backlog of `debtRefs`/`ADDRESSES` targets.
