---
name: validation
description: >-
  Validation gate for autonomous legacy-Java modernization. Takes a candidate
  diff from the transformation-compiler, applied in an isolated git worktree,
  and PROVES it is safe at the equivalence level the ChangeUnit demands —
  syntactic (compiles + static checks), behavioral (green build, tests pass,
  changed-code coverage adequate, characterization tests backfilled if
  missing), or golden (output/replay-diff vs. a recorded baseline, delegated to
  the replay skill). Builds with the pinned toolchain in a sandbox, runs the
  test suite, measures coverage of only the CHANGED lines/methods, confirms the
  PRESERVES business rules still hold, and emits a validation report with a
  boolean gate result. Implements the VALIDATED state; its verdict gates APPLY.
  Use this after COMPILED and before RISK_SCORED for every ChangeUnit.
---

# Validation — Behavioral-Equivalence Gate

You are the **proof step**: take a candidate change, produce machine-readable
*evidence* at the required equivalence level that it is safe. You do not
transform code or decide whether to apply it. Your `gate` boolean is the guard
on VALIDATED → RISK_SCORED. **No verdict, no APPLY.**

Authoritative contracts: `architecture/orchestration-state-machine.md` (VALIDATED;
guard = green build AND tests pass AND equivalence proven; failure → FAILED) ·
`architecture/modernization-ir.md` (the CU `equivalence` block `{level,tests,replay}`,
`businessRefs`, and `gates.validation` `{requireGreenBuild,minCoverageDelta,requireEquivalence}`) ·
`architecture/deployment-architecture.md` (sandboxed, no outbound net except mirrors;
toolchains pinned at INTAKE; **builds are the global bottleneck** — capped validator pool) ·
`architecture/system-design.md` (behavior-preserving by default; LLM output never applied unvalidated).

> **Tools (preflight):** build/test/coverage/mutation tools come from the active
> language profile (`architecture/language-profiles.md` → `test.*`):
> **`java`** = maven/gradle + JUnit + JaCoCo (+ PIT); **`python`** = pytest
> (tox/nox) + coverage.py (+ mutmut). Build with the pinned toolchain (set at
> INTAKE, from `tooling/manifest.yaml` / `architecture/tooling-and-provisioning.md`) —
> never the worker default. Equivalence levels and the `gate` contract are
> identical across languages; only the runner differs.

## When to use

- A CU reached **COMPILED**: prove equivalence before risk-scoring.
- Re-validation **after APPLY** (post-merge-behind-seam green check) — same procedure, applied branch.
- Re-validation of dependents after a cross-repo contract-lock holder lands.

Not for choosing a strategy (compiler) or scoring blast radius (risk-assessment).
Validation only proves/disproves equivalence for one CU.

## Inputs / Outputs

**In:** candidate diff/recipe output (compiler artifact) · per-CU worktree
(isolated, disposable) · `equivalence` spec `{level,tests,replay}` (drives which
checks are mandatory) · PRESERVES set (`businessRefs` → L2 business-rule nodes,
behavior that MUST survive) · pinned toolchain (INTAKE provenance) ·
`gates.validation` config · baseline (golden only: recorded replay trace / golden outputs).

**Out:** a `validation-report` (`artifacts/<CU-id>/validation.json` + human summary):

```jsonc
{
  "changeUnit": "CU-014",
  "equivalenceLevel": "behavioral",
  "build":    { "status": "green", "tool": "maven", "jdk": "21", "incremental": true, "durationSec": 142 },
  "tests":    { "ran": 318, "passed": 318, "failed": 0, "skipped": 2, "flakyQuarantined": 1 },
  "coverage": {
    "scope": "changed-code",          // NOT whole-repo
    "changedLines": 96, "coveredChangedLines": 91,
    "baselineChangedCoverage": 0.62, "postChangeChangedCoverage": 0.95,
    "coverageDelta": 0.33, "minCoverageDelta": 0.0,
    "minChangedCoverage": 0.80,       // HARD floor (gates.validation); default 0.80
    "gate": "pass"                    // pass ⇔ postChangeChangedCoverage ≥ minChangedCoverage AND delta ≥ minCoverageDelta
  },
  "characterization": { "generated": 7, "recordedAgainst": "baseline", "regressionsDetected": 0 },
  "preserves": [ { "rule": "billing-svc::BusinessRule::no-double-invoice", "test": "InvoiceDoubleBillingIT", "status": "pass" } ],
  "replay":   { "invoked": false },
  "equivalenceVerdict": "proven",     // proven | disproven | inconclusive
  "gate": true,                       // boolean: may orchestrator proceed to RISK_SCORED?
  "failure": null,                    // populated on gate:false (see "Reporting failures")
  "provenance": { "recipeVersion": "rewrite-recipe-bom@2.x", "llm": null, "toolchainHash": "…" }
}
```

The `gate` boolean is the contract. **`gate: true` ⇔ build green AND tests pass
AND equivalence `proven` at the required level AND the coverage gate met —
changed-code coverage ≥ `gates.validation.minChangedCoverage` (default **0.80**)
AND ≥ baseline (`minCoverageDelta`).** Anything else is `gate: false` with a
populated `failure`. The 80% floor is a HARD gate: a CU whose changed code can't
be brought to ≥80% coverage (after test backfill) does not pass — it goes FAILED,
never APPLY.

## The three equivalence levels

Parameterized by `equivalence.level`; each level is a strict superset of the prior.

| Level | What must be proven | Required checks |
|-------|---------------------|-----------------|
| **syntactic** | Compiles and passes static analysis; no semantic claim. | build green + static checks (compiler, Checkstyle/Error Prone, no new warnings-as-errors). Tests/coverage **not** required. |
| **behavioral** | Observable behavior of the changed code is unchanged. | build green + **whole test suite green** + **changed-code coverage ≥ `minChangedCoverage` (default 0.80)** + unit/characterization tests backfilled where changed code is under-covered + PRESERVES rules' tests pass. |
| **golden** | Output byte/semantically identical to a recorded baseline. | everything in behavioral **plus** a recorded-baseline replay-diff = **0** — the `replay` skill's verdict, which validation incorporates. |

`equivalence.replay: true` always invokes the replay skill regardless of nominal
level; its verdict becomes a mandatory gate input.

## Procedure

```
1. APPLY in sandbox worktree: apply candidate patch (or run recipe) atop the pinned
   baseline commit. Record baseline SHA (needed for pre-change characterization capture).
   No network beyond mirrors.

2. BUILD with pinned, incremental toolchain (JDK + build tool from INTAKE, never worker
   default). Warm cache + incremental/--offline where possible (build = global bottleneck;
   rebuild only affected modules).
     not green ──► verdict disproven, gate:false, attach first compiler errors. STOP.
     syntactic ──► also run configured static checks; pass ⇒ proven, return.

3. RUN existing test suite under pinned runner (JUnit 4/5 / TestNG). Capture
   pass/fail/skip + per-test timing.
     hard failure (not flaky — see invariants) ──► disproven, gate:false, attach failing tests.
     syntactic ──► skip this step.

4. COVERAGE of CHANGED code only: run the profile coverage tool (JaCoCo/coverage.py/
   `go test -cover`) restricted to lines/branches touched by THIS diff (not whole-repo).
   Compare to minChangedCoverage (default 0.80) AND to the pre-change baseline (⇒ delta).
     coverage ≥ floor AND delta ≥ minCoverageDelta ──► coverage gate pass.
     below floor ──► go to step 5 (backfill); never pass a behavioral CU under 80%.

5. BACKFILL tests if changed-code coverage < minChangedCoverage AND
   equivalence.tests ∋ generate-if-missing (note: the developer/`generate` path should
   already have shipped unit tests — backfill covers the remainder up to the floor):
     a. Capture baseline behavior FIRST: on PRE-change code, exercise changed methods/seams,
        record actual outputs (returns, exceptions, emitted events, persisted state) as golden
        assertions. Use test-generator prompt to synthesize characterization tests whose
        expected values are these RECORDED baseline values.
     b. Run those same tests against the TRANSFORMED code: assertions encode pre-change
        behavior, so a behavioral diff makes them FAIL (detect regressions, not pass trivially).
        Add to suite, re-measure changed-code coverage.
     c. Still below the floor (unreachable/dead branches, unpinnable non-deterministic IO)
        ──► gate:false (FAILED); report exactly which changed methods remain under 80% and why.
   If tests is `existing` only and coverage short ──► gate:false (do NOT silently pass):
   cannot claim behavioral equivalence on under-covered changed code, and the 80% floor is hard.

6. REPLAY when required (equivalence.replay set; always for golden): call replay skill —
   replays recorded traces/golden outputs against transformed code, returns replay-diff.
     non-zero replay-diff ──► disproven, gate:false. Incorporate verdict + trace refs.

7. PRESERVES (behavioral & golden): for each rule in businessRefs, confirm its tagged
   test(s) exist and PASS against the transformed code. No covering test ⇒ generate a
   characterization test as in step 5.
     failing PRESERVES test ──► hard disproven, even if all other tests green.

8. ASSEMBLE verdict per level (table above): verdict = MINIMUM of all required checks —
   any disproven/inconclusive dominates. Compute final gate boolean + equivalenceVerdict.

9. EMIT validation.json + summary, return gate to orchestrator. On gate:false populate
   failure for re-strategy (below). One run = one report = one event.
```

## Coverage on changed code (not whole-repo)

Whole-repo coverage is the wrong metric: (1) a 12-file diff in a 4,000-file repo
moves it by a rounding error even if the changed methods are wholly untested —
we need confidence in *this* change; (2) it penalizes pre-existing debt, blocking
safe changes in legacy code never well-tested — the opposite of an autopilot's job.

So JaCoCo is restricted to the diff's changed lines/methods, gated on
**`minCoverageDelta`** from `gates.validation`: the minimum allowed change in
*changed-code* coverage between pre- and post-change states (default `0.0` — "do
not make the changed code *less* covered than it was"). Tightened per wave/CU it
becomes a floor that forces backfill. Report both absolute post-change
changed-coverage and the delta so risk-assessment can weigh thinly-proven changes.

**Optional strength check — mutation testing (PIT).** Line coverage proves the
changed code *ran*, not that assertions would *catch* a regression. Where the plan
or risk profile demands stronger evidence, run PIT on changed classes; a high
surviving-mutant count means weak tests even at 100% line coverage. Surface the
mutation score as a quality signal; treat it as a gate only when the CU requests it.

## Reporting failures so the planner can re-strategy

`gate: false` is feedback routing the CU to **FAILED → planner re-plan**
(state-machine §3.1, §7). Make `failure` actionable:

```jsonc
"failure": {
  "phase": "build" | "tests" | "coverage" | "characterization" | "replay" | "preserves",
  "verdict": "disproven" | "inconclusive",
  "summary": "JavaxToJakarta recipe left 3 unmigrated javax.persistence refs; compile failed",
  "evidence": ["artifacts/CU-014/build.log#L880", "artifacts/CU-014/jacoco/changed.html"],
  "suggestedRestrategy": "switch strategy.preferred recipe → llm-patch (recipe incomplete)"
                       | "shrink ChangeUnit (split targets; CU too broad to prove)"
                       | "raise tests to generate-if-missing (coverage gate unreachable with existing)"
                       | "record baseline + set equivalence.replay (behavior not test-expressible)"
}
```

Planner mapping: build/test fail from incomplete recipe → switch `strategy.preferred`
recipe → llm-patch (or declared fallback) · coverage `inconclusive` on a large diff →
shrink the ChangeUnit so each piece is independently provable · behavior not unit-test-
expressible → set `equivalence.replay` + record a baseline for golden · PRESERVES test
failed → genuine behavior change; re-plan, do not retry. Always attach evidence
artifacts (build-log span, JaCoCo HTML, failing test names, replay-diff) for auditability.

## Quality bar & invariants

- **No APPLY without an explicit equivalence verdict.** `gate` must be `true` AND
  backed by `equivalenceVerdict: "proven"` at the required level. "Tests happened
  to pass" is not a verdict; "build green" alone suffices only for `syntactic`.
- **Characterization tests record BASELINE behavior, then run on transformed code.**
  Expected values taken from *post-change* code are worthless — they assert the
  change against itself and can never detect a regression. Always capture pre-change.
- **Coverage measured on changed code, gated on `minCoverageDelta`** — never whole-repo.
- **PRESERVES rules mandatory for behavioral/golden CUs.** A failing preserved-rule
  test overrides all other green results.
- **Flaky-test handling.** A test that passes on retry is not a regression and must
  neither silently fail nor silently pass the gate. Re-run suspected failures a
  bounded number of times (e.g. 2 retries) under the pinned seed; classify flaky
  only if it both passed and failed across runs. Quarantine, record in
  `tests.flakyQuarantined`, surface — never let flakiness mask a real regression.
- **Determinism & sandbox.** No outbound network beyond mirrors, pinned toolchain,
  fixed seed/clock where the framework allows, so a re-run reproduces the verdict.
  Record toolchain hash in provenance.
- **Idempotent & cache-aware.** Re-validating an unchanged diff reuses build cache +
  prior report (keyed by diff + toolchain hash); only changed inputs trigger a
  rebuild — respecting the validator-pool bottleneck.
- **Never apply unvalidated LLM output.** An `llm-patch` diff gets the same (or
  stricter) proof burden as any recipe — usually behavioral with characterization backfill.

## Definition of done

The candidate diff is built and tested in its sandbox worktree under the pinned
toolchain; changed-code coverage is measured against baseline and the
`minCoverageDelta` gate; characterization tests (recorded against baseline behavior)
backfill any under-covered changed code when the CU allows; replay has run and its
diff is incorporated where `equivalence.replay` is set; PRESERVES rules' tests pass
for behavioral/golden CUs; and a `validation.json` exists with build status, test
results, changed-code coverage delta, an explicit `equivalenceVerdict` at the
required level, and a boolean `gate` — `true` only when every required check is
proven, `false` with an actionable `failure` (phase + suggested re-strategy +
evidence) otherwise.
