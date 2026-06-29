---
name: risk-assessment
description: >-
  Risk gate for autonomous legacy-Java modernization. For ONE already-validated
  ChangeUnit, compute a calibrated risk score in [0,1] from explicit factors
  (blast radius, test coverage, business criticality, equivalence confidence,
  cross-repo contract exposure), aggregate them with a documented weighting plus
  hard safety overrides, and apply the IR risk gate to decide auto | pause |
  block. Writes graph layer L5: a RiskScore node (factors + decision +
  explanation), a HAS_RISK edge, and RISK_DRIVEN_BY edges to the nodes that
  dominate the score. This is the LAST gate before APPLY — it must be
  conservative and every score must be explainable in plain language. Use it
  when a CU has reached RISK_SCORED (build green, tests pass, equivalence
  proven) and the orchestrator needs a go/pause/stop decision.
---

# Risk Assessment — Last Gate Before Apply

You are the **final gate**: take ONE already-`VALIDATED` (and replayed where
required) ChangeUnit, decide **how dangerous it is to land**, turn that into one
calibrated [0,1] score, map it through the IR gate to `auto|pause|block`, and
leave an audit trail a human can read. You implement `RISK_SCORED` and own graph
layer **L5 (Risk)**. You do NOT re-validate, apply, or check behavior.

**Bias up.** Thin or conflicting evidence rounds **up**, never down — a false
PAUSE costs a glance, a false auto-apply costs a production regression.

Authoritative contracts: `architecture/orchestration-state-machine.md` §3
(`RISK_SCORED` + the exact gate), §4 (transition event) · `architecture/modernization-ir.md`
§5 (`gates.risk{autoApplyCeiling,pauseAboveCeiling,blockAbove}`,
`gates.review.requireHumanFor`) · `graph/node-types.md` (L5 `RiskScore`),
`graph/edge-types.md` (`HAS_RISK`, `RISK_DRIVEN_BY`) · `graph/cypher-queries.md`
(reuse blast-radius / cross-repo cyphers — never re-parse source) ·
`architecture/open-question-resolution.md` §4 (`Gap.raisesRiskBy`).

## When to use

- A CU went `VALIDATED → RISK_SCORED` and the orchestrator needs go/pause/stop.
- Re-scoring after inputs changed (new report, re-index, re-plan) — idempotent,
  overwrites the prior L5.
- Explaining a held CU: read its RiskScore + RISK_DRIVEN_BY edges.

Not for gating analysis stages, doing the apply, or validating behavior.

## Inputs / Outputs

**In (one CU):**

| Input | Source |
|-------|--------|
| Validated `ChangeUnit` node (L4) | graph — `kind`, `strategy`, `equivalence`, `blastRadius`, `targets`, `PRESERVES`/`TRANSFORMS` |
| Validation report (+ replay report if `equivalence.replay`) | artifact store — build/test, changed-code coverage, equivalence verdict |
| Review report (`code-review`) | artifact store — intent/fail-loud findings; unresolved med/high findings **raise** the score (high = a hard override toward pause/block) |
| Semantic graph | graph — blast-radius + cross-repo traversals |
| `gates.risk` + `gates.review.requireHumanFor` | Modernization IR §5 |

**Out (graph layer L5):**

| Output | Shape |
|--------|-------|
| `RiskScore` node | `score`[0–1], `factors{blastRadius,testCoverage,businessCriticality,equivalenceConfidence,crossRepo,gapRisk}` each [0,1], `decision`(auto\|pause\|block), `explanation` (plain-language, **required**) |
| `HAS_RISK` edge | `ChangeUnit → RiskScore` |
| `RISK_DRIVEN_BY` edges | `RiskScore → sourceNode` per dominant factor, props `factor` + `weight` |
| `risk.json` + `RISK_SCORED` event | `artifacts/<CU>/risk.json` (state-machine §4), event carries `result.risk` |

Orchestrator reads only `score` + `decision`; the rest is for humans + audit.

## Scoring model

**Five weighted factors** (sum to 1.0) + **one additive gap term**, each normalized
to [0,1] where 1 = most risky. `gapRisk` is *added on top* (extra risk from building
on unknowns — it raises a confident score rather than diluting the rest), then **hard
overrides** apply. Weights are documented defaults living in run config — **calibratable
against post-apply rollback data, never silently**.

### Factors

1. **blastRadius** — code this change can break. Run "Blast radius of a ChangeUnit"
   cypher: `targets` = TRANSFORMed nodes, `callers` = transitive (`CALLS*1..2`);
   cross-check CU `blastRadius.{files,callersAffected}`.
   `factor = min(1, log10(1+callers) / log10(1+CALLERS_SATURATION))`,
   `CALLERS_SATURATION = 500` (500-caller fan-out ≈ 1.0; log scale so 10 vs 50 > 400 vs 440).

2. **testCoverage** — *inverse* of how well changed code is exercised. From the
   validation report, line/branch coverage **of the changed lines** (fall back to
   module coverage only if absent — say so in the explanation). `factor = 1 - coverage`.

3. **businessCriticality** — max `criticality` over Capabilities the CU `PRESERVES`
   (criticality lives on `Capability`, reach via `IMPLEMENTS_RULE`→`REALIZES_CAPABILITY`;
   use "Business rules a change would touch" + "Capability footprint" cyphers).
   Map `low→0.2, med→0.5, high→0.9`. Discount by rule confidence — low-confidence
   preserved rules mean we are *less* sure what to protect, so nudge **up**:
   `criticality * (1 + 0.3*(1 - avgRuleConfidence))`, capped at 1.0.

4. **equivalenceConfidence** — *inverse* confidence behavior is preserved; consume the
   validation/replay verdict directly (you run after them). Start from its [0,1]
   confidence, then penalize weak proofs:
   - `strategy.preferred == "llm-patch"` + thin tests (low changed-code coverage, no replay) → cap 0.5
   - `equivalence.level == "syntactic"` on a non-trivial CU → cap 0.6
   - `golden`/`replay`-backed → may reach ~0.95
   `factor = 1 - equivalenceConfidence`.

5. **crossRepo** — touches a shared contract (Endpoint, Table, published event) another
   repo depends on? Run "Cross-repo contracts a ChangeUnit affects": any contract GIDs
   → `0.8` (raise toward 1.0 with # distinct dependent repos); else `0.0`.

6. **gapRisk** *(additive, unweighted)* — extra risk from building on unknowns. Follow
   `HAS_GAP` from the CU (and from depended-on `ExternalSystem`/`InterfaceContract` via
   the flow): `gapRisk = min(GAP_CAP, Σ Gap.raisesRiskBy)`, `GAP_CAP = 0.3`. A CU resting
   on a low-confidence placeholder thus carries measurable risk even with a clean diff.
   Stored in `factors.gapRisk`; **added** to the weighted sum, not multiplied by a weight.

### Weights + combination

```
w_blastRadius = 0.25 · w_testCoverage = 0.20 · w_businessCriticality = 0.20
w_equivalenceConfidence = 0.25 · w_crossRepo = 0.10            (= 1.00)

baseScore = Σ (w_i · factor_i)         over the five weighted factors  → [0,1]
preScore  = min(1, baseScore + gapRisk)   (gapRisk recorded in factors{}, not weighted)
```

### Hard overrides (after the weighted sum; each only **raises** score, must be in explanation)

- **O1 — Cross-repo + shaky equivalence:** `crossRepo > 0` AND `equivalenceConfidence < 0.7`
  (factor ≥ 0.3) → `score = max(score, autoApplyCeiling + ε)` → **≥ PAUSE**.
- **O2 — Low-confidence LLM patch on critical behavior:** `strategy.preferred == "llm-patch"`
  AND `businessCriticality factor ≥ 0.7` AND equivalence not `golden`/`replay`-backed →
  `score = max(score, blockAbove)` → **BLOCK** (needs a plan change).
- **O3 — Saturated blast radius:** `blastRadius factor ≥ 0.9` AND `testCoverage factor ≥ 0.6`
  → **≥ PAUSE**.
- **O4 — Equivalence unknown:** report could not prove equivalence (shouldn't reach you;
  defensive) → **BLOCK**.

After overrides, `score` is final; round to 2 decimals for storage.

## Decision mapping (the state-machine gate)

```
risk ≤ autoApplyCeiling                  → auto   → APPLY
autoApplyCeiling < risk < blockAbove     → pause  → PAUSED
risk ≥ blockAbove                        → block  → BLOCKED
CU.kind ∈ gates.review.requireHumanFor   → pause  → PAUSED  (override, never below PAUSE; can still escalate to BLOCK if risk ≥ blockAbove)
```

Example IR gates (`autoApplyCeiling 0.4`, `blockAbove 0.8`,
`requireHumanFor [seam-extraction, security-fix]`): `≤0.4`→auto · `0.4–0.8`→pause ·
`≥0.8`→block · `seam-extraction`/`security-fix`→≥pause even at `≤0.4`.

`decision` and `score` MUST be consistent — the orchestrator re-derives the
transition from the gate math, so a contradicting `decision` is an invariant violation.

## Procedure

```
1. LOAD: CU node + validation report (+ replay if equivalence.replay). Confirm status
   == VALIDATED, else REFUSE (you are not the validator). Resolve gates.risk +
   gates.review.requireHumanFor from IR.

2. GATHER factor inputs (reuse canonical cyphers — never re-parse source):
   blastRadius          → "Blast radius of a ChangeUnit", reconcile w/ CU blastRadius, log-normalize
   testCoverage         → changed-code coverage → 1 - coverage
   businessCriticality  → "Business rules…" + "Capability footprint" over PRESERVES → max crit, discount by rule conf
   equivalenceConfidence→ validation/replay verdict + strategy/level penalties → 1 - confidence
   crossRepo            → "Cross-repo contracts…" → 0 or ≥0.8
   gapRisk              → HAS_GAP (CU + depended-on ExternalSystem/InterfaceContract) → min(0.3, Σ raisesRiskBy)

3. COMBINE: baseScore = Σ w_i·factor_i; preScore = min(1, baseScore + gapRisk).
   Record each factor value + its contribution (weighted, or additive for gapRisk).

4. OVERRIDES O1–O4 in order on preScore; record any that fired → final score.

5. DOMINANT factors: rank by weighted contribution (w_i·factor_i); dominant set =
   top contributors + any factor that fired an override → RISK_DRIVEN_BY targets.

6. GATE → decision (incl. requireHumanFor override). Verify decision matches threshold math.

7. WRITE L5:
   • RiskScore node: score, full factors{}, decision, explanation.
   • HAS_RISK: ChangeUnit → RiskScore.
   • RISK_DRIVEN_BY: RiskScore → the REAL source node per dominant factor (the
     high-crit Capability, the shared-contract Endpoint, the under-tested Class…),
     props factor (which of six) + weight (its contribution).
   • explanation (required, never empty): score, drivers, fired overrides, what a
     human must check if PAUSED/BLOCKED.
   • Emit risk.json + RISK_SCORED transition event with result.risk.
```

## Explainability (non-negotiable)

A human must understand a PAUSE/BLOCK from the graph alone:
- Every `RiskScore` carries a non-empty `explanation` naming dominant factors in plain
  English — "why risky?" resolves to "touches *this* high-crit capability + *that*
  cross-repo endpoint", not an opaque 0.62.
- Every dominant factor has a `RISK_DRIVEN_BY` edge to the concrete source node, with
  `factor` + `weight`.
- Every fired override is stated in the explanation.

If you cannot explain a score, you do not emit it.

## Worked example

CU-014 "Replace javax.* with jakarta.* in billing-svc" (`kind: api-migration`,
`strategy.preferred: recipe`), gates `0.4 / 0.8`.

| Factor | Raw evidence | Normalized | × weight |
|--------|--------------|-----------:|---------:|
| blastRadius | 12 files, 37 callers → `log10(38)/log10(501)` | 0.58 | 0.25 → 0.146 |
| testCoverage | changed-code coverage 82% | 0.18 | 0.20 → 0.036 |
| businessCriticality | PRESERVES `Invoicing` (high=0.9), rule conf 0.85 → 0.9·(1+0.3·0.15) | 0.94 | 0.20 → 0.188 |
| equivalenceConfidence | deterministic recipe, behavioral tests green, conf 0.9 → `1-0.9` | 0.10 | 0.25 → 0.025 |
| crossRepo | no shared contract touched | 0.00 | 0.10 → 0.000 |
| gapRisk *(additive)* | no gaps; no placeholder dep | 0.00 | + 0.000 |

`baseScore = 0.146+0.036+0.188+0.025+0.000 = 0.40`; `preScore = 0.40 + 0.00 = 0.40`.
Overrides: O1 no (crossRepo 0), O2 no (not llm-patch), O3 no, O4 no → final `score = 0.40`.
Gate: `0.40 ≤ 0.4` → **auto / APPLY**; `api-migration` ∉ `requireHumanFor`. Dominant:
**businessCriticality** (0.188) → `RISK_DRIVEN_BY → Capability::Invoicing`; secondary
**blastRadius** → `Class::InvoiceService`.

Explanation: *"Low risk (0.40, auto-apply). Deterministic OpenRewrite recipe, 82%
changed-code coverage, green behavioral tests — strong equivalence (0.90). Main driver
is preserving the high-criticality Invoicing capability, but the proof is solid and no
cross-repo contract is touched. No overrides. Lands at exactly the ceiling, so a small
coverage regression or recipe change would push it to PAUSE on re-score."*

Contrasts:
- Same CU as `strategy: llm-patch`, 30% coverage, no replay → `equivalenceConfidence`
  caps at 0.5 (factor 0.5 → ×0.25 = 0.125), `baseScore` ~0.50 → **PAUSE**; if it touched
  a shared Endpoint (`crossRepo`), **O1** forces PAUSE regardless.
- **CU-021** (payments seam) depends on **GAP-007** (placeholder payment-provider API,
  `raisesRiskBy 0.15`): five weighted factors base ≈ 0.37, `+ gapRisk(0.15)` →
  `score ≈ 0.52` → PAUSED (and `seam-extraction` ∈ `requireHumanFor` pauses regardless).
  Matches `change-units/CU-021.md`.

## Quality / invariants

- **No orphan RiskScore** — reachable via exactly one `HAS_RISK`, has ≥1 `RISK_DRIVEN_BY`.
- **Decision matches gate math** — recomputed from `score`, `gates.risk`, `requireHumanFor`;
  an inconsistent stored `decision` is a hard failure.
- **Explanation required & factor-grounded** — non-empty, names dominant factors + overrides,
  consistent with the `RISK_DRIVEN_BY` edges.
- **Factors and score in [0,1]**, clamped.
- **Conservative on uncertainty** — missing/ambiguous evidence rounds **up**; unknown
  equivalence → BLOCK; never down-rank to clear the gate.
- **Idempotent** — re-scoring replaces the prior L5 (no duplicates); same inputs → same score.
- **Read-only on source & L0–L4** — writes L5 only.

## Definition of done

The validated ChangeUnit has a single `RiskScore` (L5) with all five factors, a final
`score`, and a `decision` that provably matches the IR gate (incl. the `requireHumanFor`
override); a `HAS_RISK` edge and `RISK_DRIVEN_BY` edges to the dominant factor sources; a
required, plain-language explanation naming the drivers and any fired overrides; and a
persisted `risk.json` + a `RISK_SCORED` event carrying `result.risk`. The orchestrator can
now transition the CU to APPLY, PAUSED, or BLOCKED with a fully auditable rationale.
