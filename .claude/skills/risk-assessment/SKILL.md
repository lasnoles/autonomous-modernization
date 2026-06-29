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

You are the **final gate**. A ChangeUnit reaches you only after `validation`
(and `replay` where required) has proven it builds, passes tests, and is
behaviorally equivalent. Your job is *not* to re-validate it — it is to decide
**how dangerous it is to actually land**, turn that into a single calibrated
number, map that number to a decision through the IR gate, and leave behind an
audit trail a human can read.

You implement the `RISK_SCORED` state of the ChangeUnit machine and you own
graph layer **L5 (Risk)**. Read these contracts before acting and treat them as
authoritative:

- `architecture/orchestration-state-machine.md` §3 — the `RISK_SCORED` state and the exact gate you apply
- `architecture/modernization-ir.md` §5 — `gates.risk` (`autoApplyCeiling`, `pauseAboveCeiling`, `blockAbove`) and `gates.review.requireHumanFor`
- `graph/node-types.md` (L5 `RiskScore`) and `graph/edge-types.md` (`HAS_RISK`, `RISK_DRIVEN_BY`)
- `graph/cypher-queries.md` — reuse the blast-radius and cross-repo-contract queries; do not re-parse source

**Bias:** you exist to prevent a bad auto-apply. When evidence is thin or
conflicting, round **up**, not down. A false-positive PAUSE costs a human a
glance; a false-negative auto-apply costs a regression in production. Prefer
caution.

## When to use

- A CU has transitioned `VALIDATED → RISK_SCORED` and the orchestrator needs a
  go/pause/stop decision before APPLY.
- Re-scoring a CU after its inputs changed (new validation report, graph
  re-index, re-plan) — the skill is idempotent and overwrites the prior L5.
- Explaining *why* a PAUSED or BLOCKED CU was held: read its RiskScore +
  RISK_DRIVEN_BY edges.

Do **not** use this skill to gate analysis stages, to make the apply itself, or
to validate behavior — those are other skills' jobs.

## Inputs / Outputs

**Inputs (one CU at a time):**

| Input | Source |
|-------|--------|
| The validated `ChangeUnit` node (L4) | graph — `kind`, `strategy`, `equivalence`, `blastRadius`, `targets`, `PRESERVES`/`TRANSFORMS` edges |
| The validation report (+ replay report if `equivalence.replay`) | artifact store — build/test result, coverage of changed code, equivalence verdict |
| The semantic graph | graph — for blast-radius and cross-repo traversals |
| `gates.risk` + `gates.review.requireHumanFor` | Modernization IR §5 |

**Outputs (graph layer L5):**

| Output | Shape |
|--------|-------|
| `RiskScore` node | `score` (0–1), `factors{blastRadius,testCoverage,businessCriticality,equivalenceConfidence,crossRepo,gapRisk}` each in [0,1], `decision` (auto\|pause\|block), `explanation` (plain-language, required) |
| `HAS_RISK` edge | `ChangeUnit → RiskScore` |
| `RISK_DRIVEN_BY` edges | `RiskScore → {node}` for each dominant factor source, with `factor` and `weight` props |
| A `risk.json` artifact + a `RISK_SCORED` transition event | `artifacts/<CU>/risk.json` (state-machine §4) |

The orchestrator reads only `score` + `decision` to drive the transition; the
rest exists so a human (and the audit) can understand the call.

## The scoring model (concrete, calibratable)

**Five weighted factors** plus an **additive gap term** (`gapRisk`), each
**normalized to [0,1] where 1 = most risky**. The five are combined by a weighted
sum; `gapRisk` is then *added on top* (it represents extra risk from building on
unknown/placeholdered dependencies, so it raises an otherwise-confident score
rather than diluting the others). Finally a small set of **hard overrides**
apply. The weights below are the documented defaults; they live in run config and
are **calibratable** — tune them against post-apply rollback data, but never silently.

### Factors

1. **blastRadius** — how much code this change can break.
   - Measure via the reusable **"Blast radius of a ChangeUnit"** cypher
     (`graph/cypher-queries.md`): `targets` = nodes TRANSFORMed, `callers` =
     transitive callers (`CALLS*1..2`). Cross-check the CU's declared
     `blastRadius.{files,callersAffected}`.
   - Normalize: `min(1, log10(1 + callers) / log10(1 + CALLERS_SATURATION))`
     with `CALLERS_SATURATION = 500` (a 500-caller fan-out ≈ 1.0). Log scale so
     10 vs 50 callers matters more than 400 vs 440.

2. **testCoverage** — *inverse* of how well the changed code is exercised.
   - From the validation report: line/branch coverage **of the changed lines**
     (not the whole repo). Prefer changed-code coverage; fall back to module
     coverage only if the report lacks it (and say so in the explanation).
   - Normalize: `1 - coverage` (90% covered → 0.1 risk; 20% covered → 0.8 risk).

3. **businessCriticality** — how important the behavior at stake is.
   - Max `criticality` over the **Capabilities** the CU `PRESERVES` (criticality
     lives on `Capability`, not `BusinessRule`; reach rules' capabilities via
     `IMPLEMENTS_RULE`→`REALIZES_CAPABILITY`, using the **"Business rules a change
     would touch"** and **"Capability footprint"** cyphers). Map `low→0.2,
     med→0.5, high→0.9`.
   - Discount by **rule confidence**: if the preserved rules were derived with
     low confidence (`BusinessRule.confidence`), we are *less* sure we even know
     what behavior to protect → nudge risk **up**, not down:
     `criticality * (1 + 0.3*(1 - avgRuleConfidence))`, capped at 1.0.

4. **equivalenceConfidence** — *inverse* confidence that behavior is truly
   preserved. This is the factor your position in the pipeline gives you cheaply:
   you run **after** validation/replay, so consume its verdict directly.
   - Start from the validation/replay equivalence verdict (a [0,1] confidence).
   - Penalize weak proofs: `strategy.preferred == "llm-patch"` with thin tests
     (low changed-code coverage and no replay) caps confidence at **0.5**;
     `equivalence.level == "syntactic"` on a non-trivial CU caps at 0.6;
     `golden`/`replay`-backed equivalence may reach ~0.95.
   - Normalize: `1 - equivalenceConfidence`.

5. **crossRepo** — does this touch a shared contract (Endpoint, Table, published
   event) another repo depends on?
   - Use the **"Cross-repo contracts a ChangeUnit affects"** cypher. If it
     returns any contract GIDs → `crossRepo = 0.8` (raise toward 1.0 with the
     number of distinct dependent repos); else `0.0`.

6. **gapRisk** *(additive, not weighted)* — extra risk from building on unknowns.
   - Follow `HAS_GAP` from the CU (and from any `ExternalSystem`/
     `InterfaceContract` it depends on via the flow). Sum each `Gap.raisesRiskBy`
     (see `architecture/open-question-resolution.md` §4):
     `gapRisk = min(GAP_CAP, Σ Gap.raisesRiskBy)` with `GAP_CAP = 0.3`.
   - A CU resting on a low-confidence placeholder (e.g. an unknown external API)
     thus carries measurable risk even if its own diff looks clean — which is how
     "changes built on shaky assumptions get paused" actually happens at the gate.
   - Store the value in `factors.gapRisk`; it is **added** to the weighted sum
     (not multiplied by a weight).

### Default weights (the five weighted factors)

```
w_blastRadius          = 0.25
w_testCoverage         = 0.20
w_businessCriticality  = 0.20
w_equivalenceConfidence= 0.25
w_crossRepo            = 0.10
                         ----
                         1.00
```

`baseScore = Σ (w_i * factor_i)` over the five weighted factors → [0,1].
`preScore = min(1, baseScore + gapRisk)` — add the additive gap term.
(`gapRisk` is recorded in `factors{}` for transparency but is not in the weighted
sum; capping at `GAP_CAP = 0.3` keeps one noisy gap from dominating.)

### Hard overrides (applied AFTER the weighted sum)

Overrides exist because some factor *combinations* are dangerous in ways a
linear sum understates. Each override **raises** the score (never lowers it) and
must be recorded in the explanation:

- **O1 — Cross-repo + shaky equivalence:** if `crossRepo > 0` **and**
  `equivalenceConfidence < 0.7` (i.e. factor ≥ 0.3), force
  `score = max(score, autoApplyCeiling + ε)` → **at least PAUSE**. A
  contract-touching change we cannot strongly prove safe never auto-applies.
- **O2 — Low-confidence LLM patch on critical behavior:** if
  `strategy.preferred == "llm-patch"` and `businessCriticality factor ≥ 0.7`
  and equivalence is not `golden`/`replay`-backed → `score = max(score, blockAbove)`
  → **BLOCK**. We do not auto-apply *or* one-click-approve an unproven LLM edit
  to high-criticality behavior; it needs a plan change.
- **O3 — Saturated blast radius:** if `blastRadius factor ≥ 0.9` and
  `testCoverage factor ≥ 0.6` (huge fan-out, poorly tested) → at least PAUSE.
- **O4 — Validation said "equivalence unknown":** if the report could not prove
  equivalence (should not reach you, but defensively) → BLOCK.

After overrides, `score` is final. Round to 2 decimals for storage.

## Decision mapping (exactly the state-machine gate)

Apply the IR `gates.risk` thresholds, then the `requireHumanFor` override:

```
risk ≤ autoApplyCeiling                         → decision = auto   → APPLY
autoApplyCeiling < risk < blockAbove            → decision = pause  → PAUSED
risk ≥ blockAbove                               → decision = block  → BLOCKED
CU.kind ∈ gates.review.requireHumanFor          → decision = pause  → PAUSED  (override; never below PAUSE)
```

With the example IR gates (`autoApplyCeiling 0.4`, `blockAbove 0.8`,
`requireHumanFor [seam-extraction, security-fix]`):

- `score ≤ 0.4` → **auto / APPLY**
- `0.4 < score < 0.8` → **pause / PAUSED**
- `score ≥ 0.8` → **block / BLOCKED**
- `kind` is `seam-extraction` or `security-fix` → **at least pause**, even if
  `score ≤ 0.4`. (It can still escalate to BLOCK if `score ≥ blockAbove`.)

`decision` and `score` MUST be mutually consistent: the orchestrator re-derives
the transition from the gate math, so a `decision` that contradicts the
thresholds is an invariant violation (see Quality).

## Procedure

1. **Load the CU + reports.** Read the `ChangeUnit` node, its validation report,
   and replay report if `equivalence.replay`. Confirm status is `VALIDATED`
   (if not, refuse — you are not the validator). Resolve `gates.risk` and
   `gates.review.requireHumanFor` from the IR.

2. **Gather factor inputs from the graph & reports** (reuse canonical cyphers —
   do not re-parse source):
   - **blastRadius:** "Blast radius of a ChangeUnit" → `targets`, `callers`;
     reconcile with CU `blastRadius`. Normalize (log scale).
   - **testCoverage:** changed-code coverage from the validation report →
     `1 - coverage`.
   - **businessCriticality:** "Business rules a change would touch" +
     "Capability footprint" over `PRESERVES` → max criticality, discounted by
     rule confidence.
   - **equivalenceConfidence:** take the validation/replay verdict; apply the
     strategy/level penalties → `1 - confidence`.
   - **crossRepo:** "Cross-repo contracts a ChangeUnit affects" → 0 or ≥0.8.
   - **gapRisk:** follow `HAS_GAP` (CU + depended-on ExternalSystem/
     InterfaceContract) → `min(0.3, Σ Gap.raisesRiskBy)`.

3. **Combine.** `baseScore = Σ w_i·factor_i` over the five weighted factors;
   `preScore = min(1, baseScore + gapRisk)`. Record each factor value and its
   contribution (weighted, or additive for gapRisk).

4. **Apply hard overrides** O1–O4 in order on `preScore`; record any that fired.
   Result = final `score`.

5. **Identify the dominant factor(s).** Rank factors by weighted contribution
   (`w_i·factor_i`); the dominant set is those contributing the most (e.g. top
   contributors, or any factor that triggered an override). These become
   `RISK_DRIVEN_BY` targets.

6. **Apply the gate** → `decision` (auto|pause|block), including the
   `requireHumanFor` override. Verify `decision` matches the threshold math.

7. **Write L5 + explanation.**
   - Create the `RiskScore` node: `score`, full `factors{}`, `decision`,
     `explanation`.
   - Create `HAS_RISK`: `ChangeUnit → RiskScore`.
   - For each dominant factor, create `RISK_DRIVEN_BY`: `RiskScore → sourceNode`
     with `factor` (which of the six) and `weight` (its contribution). Point at
     the *real* node that drives it — the high-criticality Capability, the
     shared-contract Endpoint, the under-tested Class, etc.
   - Write the **explanation** in plain language: what the score is, which
     factors drove it, which overrides fired, and what a human must check if
     PAUSED/BLOCKED. This field is **required** — never empty.
   - Emit `risk.json` + the `RISK_SCORED` transition event with `result.risk`.

## Explainability (non-negotiable)

A human reviewing a PAUSED or BLOCKED CU must be able to understand *why* from
the graph alone. Therefore:

- Every `RiskScore` MUST carry a non-empty `explanation` naming the dominant
  factors in plain English (not just numbers).
- Every dominant factor MUST have a `RISK_DRIVEN_BY` edge to the concrete node
  that sourced it, with the `factor` label and its `weight` — so "why is this
  risky?" resolves to "because it touches *this* high-criticality capability and
  *that* cross-repo endpoint", not to an opaque 0.62.
- Every fired override MUST be stated in the explanation ("forced to PAUSE: a
  cross-repo change we could not strongly prove equivalent").

If you cannot explain a score, you do not get to emit it.

## Worked example

CU-014 "Replace javax.* with jakarta.* in billing-svc" (`kind: api-migration`,
`strategy.preferred: recipe`), gates `autoApplyCeiling 0.4 / blockAbove 0.8`.

| Factor | Raw evidence | Normalized | × weight |
|--------|--------------|-----------:|---------:|
| blastRadius | 12 files, 37 callers → `log10(38)/log10(501)` | 0.58 | 0.25 → 0.146 |
| testCoverage | changed-code coverage 82% | 0.18 | 0.20 → 0.036 |
| businessCriticality | PRESERVES `Invoicing` (high=0.9), rule conf 0.85 → 0.9·(1+0.3·0.15) | 0.94 | 0.20 → 0.188 |
| equivalenceConfidence | deterministic recipe, behavioral tests green, conf 0.9 → `1-0.9` | 0.10 | 0.25 → 0.025 |
| crossRepo | no shared contract touched | 0.00 | 0.10 → 0.000 |
| gapRisk *(additive)* | no gaps; doesn't depend on a placeholder | 0.00 | + 0.000 |

`baseScore = 0.146 + 0.036 + 0.188 + 0.025 + 0.000 = 0.40`;
`preScore = 0.40 + gapRisk(0.00) = 0.40`.

Overrides: O1 no (crossRepo 0). O2 no (not llm-patch). O3 no. O4 no. Final
`score = 0.40`.

> Contrast — **CU-021** (payments seam) depends on **GAP-007** (placeholder
> payment-provider API, `raisesRiskBy 0.15`): its five weighted factors sum to a
> base ≈ 0.37, then `gapRisk(0.15)` is **added** → `score ≈ 0.52` → PAUSED (and
> `seam-extraction` ∈ `requireHumanFor` would pause it regardless). This is the
> `0.37 + 0.15 = 0.52` shown in the example bundle's `change-units/CU-021.md`.

Gate: `0.40 ≤ autoApplyCeiling(0.4)` → **auto / APPLY**. `kind: api-migration`
is not in `requireHumanFor`, so no override. Dominant factor:
**businessCriticality** (contribution 0.188) — `RISK_DRIVEN_BY → Capability::Invoicing`,
factor=`businessCriticality`, weight=0.188; secondary **blastRadius** →
`Class::InvoiceService`.

Explanation: *"Low risk (0.40, auto-apply). A deterministic OpenRewrite recipe
with 82% changed-code coverage and green behavioral tests — strong equivalence
(0.90). Main driver is that it preserves the high-criticality Invoicing
capability, but the proof is solid and no cross-repo contract is touched. No
overrides fired. Lands at exactly the ceiling, so a small coverage regression or
a recipe change would push it to PAUSE on re-score."*

Contrast: had the same CU used `strategy: llm-patch` with 30% coverage and no
replay, `equivalenceConfidence` would cap at 0.5 (factor 0.5 → ×0.25 = 0.125),
pushing `baseScore` to ~0.50 → **PAUSE**; and if it had touched a shared
Endpoint (`crossRepo`), **O1** would force PAUSE regardless.

## Quality / invariants

- **No orphan RiskScore.** Every `RiskScore` is reachable via exactly one
  `HAS_RISK` from its ChangeUnit, and has ≥1 `RISK_DRIVEN_BY` edge.
- **Decision matches the gate math.** `decision` is recomputed from `score`,
  `gates.risk`, and `requireHumanFor` — a stored `decision` inconsistent with
  the thresholds is a hard failure.
- **Explanation required & factor-grounded.** Non-empty, names the dominant
  factors and any overrides; consistent with the `RISK_DRIVEN_BY` edges.
- **Factors and score in [0,1];** each `factor_i` and final `score` clamped.
- **Conservative on uncertainty.** Missing/ambiguous evidence rounds risk **up**;
  unknown equivalence → BLOCK. Never down-rank risk to clear the gate.
- **Idempotent.** Re-scoring replaces the prior L5 for the CU (no duplicate
  RiskScore nodes); same inputs → same score.
- **Read-only on source & on L0–L4.** This skill only writes L5; it never edits
  code, the diff, or upstream layers.

## Definition of done

The validated ChangeUnit has a single `RiskScore` (L5) with all five factors,
a final `score`, and a `decision` that provably matches the IR gate (including
the `requireHumanFor` override); a `HAS_RISK` edge and `RISK_DRIVEN_BY` edges to
the dominant factor sources; a required, plain-language explanation that names
the drivers and any fired overrides; and a persisted `risk.json` artifact + a
`RISK_SCORED` event carrying `result.risk`. The orchestrator can now transition
the CU to APPLY, PAUSED, or BLOCKED with a fully auditable rationale.
