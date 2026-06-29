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

You are the **proof step**. You do not transform code and you do not decide
whether to apply it — you take a candidate change and produce *evidence*, at
the equivalence level the plan requires, that it is safe. Your output is a
machine-readable verdict the orchestrator uses as the guard on the VALIDATED →
RISK_SCORED transition. No verdict, no APPLY.

Read these contracts before acting and treat them as authoritative:

- `architecture/orchestration-state-machine.md` — the VALIDATED state; guard is
  **green build AND tests pass AND equivalence proven**; failure → FAILED.
- `architecture/modernization-ir.md` — the ChangeUnit `equivalence` block
  (`level`, `tests`, `replay`), the `businessRefs` to preserve, and the
  `validation` gate (`requireGreenBuild`, `minCoverageDelta`, `requireEquivalence`).
- `architecture/deployment-architecture.md` — validators run sandboxed (no
  outbound network except package mirrors), toolchains are pinned at INTAKE,
  and **builds are the global bottleneck** (a capped validator pool).
- `architecture/system-design.md` — *behavior-preserving by default*; LLM output
  is never applied unvalidated.

> **Tools (preflight):** the repo's pinned JDK + build tool (set at INTAKE) plus
> JaCoCo (and optional PIT) from `tooling/manifest.yaml`
> (`architecture/tooling-and-provisioning.md`). Build with the pinned toolchain,
> verified — never the worker default. **Build/test/coverage/mutation tools come
> from the active language profile** (`architecture/language-profiles.md` →
> `test.*`): **`java`** = maven/gradle + JUnit + JaCoCo (+ PIT); **`python`** =
> pytest (tox/nox) + coverage.py (+ mutmut). The equivalence levels and the
> `gate` contract are identical across languages — only the runner differs.

## When to use

- A ChangeUnit has reached **COMPILED**: the transformation-compiler produced a
  candidate diff/recipe in a worktree and you must prove equivalence before it
  can be risk-scored.
- Re-validation **after APPLY** (post-merge-behind-seam green check) — same
  procedure, run against the applied branch.
- Re-validation of dependents after a cross-repo contract lock holder lands.

Do **not** use this to choose a transformation strategy (that is the compiler)
or to score blast radius (that is risk-assessment). Validation only proves or
disproves equivalence for one CU.

## Inputs / Outputs

**Inputs**

| Input | Source | Notes |
|-------|--------|-------|
| Candidate diff / recipe output | transformation-compiler artifact | the patch to validate |
| Worktree | per-CU git worktree/branch | isolated; disposable |
| `equivalence` spec | ChangeUnit (`level`, `tests`, `replay`) | drives which checks are mandatory |
| PRESERVES set | ChangeUnit `businessRefs` → business-rule nodes (L2) | rules whose behavior MUST survive |
| Pinned toolchain | repo provenance (set at INTAKE) | JDK, build tool, recipe catalog, test runner versions |
| Validation gate config | IR `gates.validation` | `requireGreenBuild`, `minCoverageDelta`, `requireEquivalence` |
| Baseline (golden only) | recorded replay trace / golden outputs | from a prior run or capture |

**Output** — a `validation-report` artifact (`artifacts/<CU-id>/validation.json`
+ a human-readable summary), containing:

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
    "coverageDelta": 0.33, "minCoverageDelta": 0.0, "gate": "pass"
  },
  "characterization": { "generated": 7, "recordedAgainst": "baseline", "regressionsDetected": 0 },
  "preserves": [ { "rule": "billing-svc::BusinessRule::no-double-invoice", "test": "InvoiceDoubleBillingIT", "status": "pass" } ],
  "replay":   { "invoked": false },
  "equivalenceVerdict": "proven",     // proven | disproven | inconclusive
  "gate": true,                       // boolean: may the orchestrator proceed to RISK_SCORED?
  "failure": null,                    // populated on gate:false (see "Reporting failures")
  "provenance": { "recipeVersion": "rewrite-recipe-bom@2.x", "llm": null, "toolchainHash": "…" }
}
```

The `gate` boolean is the contract. `gate: true` ⇔ build green **and** tests
pass **and** equivalence `proven` at the required level **and** coverage gate
met. Anything else is `gate: false` with a populated `failure`.

## The three equivalence levels

Validation is parameterized by `equivalence.level`. Each level is a strict
superset of the work of the one before it.

| Level | What must be proven | Required checks |
|-------|---------------------|-----------------|
| **syntactic** | The change compiles and passes static analysis; no semantic claim. | build green + static checks (compiler, Checkstyle/Error Prone, no new warnings-as-errors). Tests/coverage **not** required. |
| **behavioral** | Observable behavior of the changed code is unchanged. | build green + **whole test suite green** + **changed-code coverage** ≥ gate + characterization tests backfilled where the changed code is under-covered + PRESERVES rules' tests pass. |
| **golden** | Output is byte/semantically identical to a recorded baseline. | everything in behavioral **plus** a recorded-baseline replay-diff = **0** — this is the `replay` skill's verdict, which validation incorporates. |

A CU with `equivalence.replay: true` always invokes the replay skill regardless
of nominal level, and the replay verdict becomes a mandatory input to the gate.

## Procedure

1. **Apply the diff in the sandbox worktree.** In the CU's isolated worktree,
   apply the candidate patch (or run the recipe) on top of the pinned baseline
   commit. Record the baseline commit SHA — you will need the **pre-change**
   state for characterization capture. No network beyond package mirrors.

2. **Build with the pinned, incremental toolchain.** Invoke Maven/Gradle using
   the JDK and build-tool versions pinned at INTAKE (never the worker default).
   Use the warm dependency cache and incremental/`--offline` where possible —
   builds are the global bottleneck, so reuse cached artifacts and only rebuild
   affected modules. If the build is **not green** → stop, verdict `disproven`,
   `gate: false`, attach the first compiler errors. For **syntactic** CUs, also
   run the configured static checks; if they pass, emit `proven` and return.

3. **Run the existing test suite.** Run the repo's JUnit (4/5) / TestNG suite
   under the pinned runner. Capture pass/fail/skip and per-test timing. Any hard
   failure (not flaky — see invariants) → `disproven`, `gate: false`, attach the
   failing tests. For **syntactic** CUs this step is skipped.

4. **Measure coverage of the CHANGED code only.** Run JaCoCo and compute
   coverage restricted to the **lines and methods touched by this diff** — not
   whole-repo coverage. Diff the changed-code coverage against the same metric
   on the pre-change baseline to get `coverageDelta`. (See "Coverage on changed
   code" below for why whole-repo coverage is the wrong metric.)

5. **Backfill characterization tests if the changed code is under-covered.**
   If changed-code coverage is below the gate **and** `equivalence.tests`
   includes `generate-if-missing`:
   - **Capture baseline behavior first.** On the *pre-change* code, exercise the
     changed methods/seams and **record their actual outputs** (return values,
     thrown exceptions, emitted events, persisted state) as golden assertions.
     Use the test-generator prompt to synthesize JUnit characterization tests
     whose expected values are these *recorded baseline* values.
   - **Then run those same tests against the transformed code.** Because the
     assertions encode pre-change behavior, a behavioral difference makes them
     **fail** — they detect regressions rather than passing trivially. Add the
     generated tests to the suite and re-measure changed-code coverage.
   - If coverage still cannot reach the gate (e.g. unreachable/dead branches,
     non-deterministic IO that cannot be pinned) → `inconclusive`, `gate: false`,
     report exactly which changed methods remain unproven.
   If `tests` is `existing` only and coverage is short → `inconclusive` (do not
   silently pass): the plan asked for no backfill, so we cannot claim behavioral
   equivalence on uncovered changed code.

6. **Invoke replay when required.** If `equivalence.replay` is set (always for
   `golden`), call the `replay` skill: it replays recorded traces / golden
   outputs against the transformed code and returns a replay-diff. A non-zero
   replay-diff → `disproven`, `gate: false`. Incorporate its verdict and trace
   artifact references into the report.

7. **Check the PRESERVES business rules (behavioral & golden).** For each rule
   in the CU's `businessRefs`, confirm the test(s) tagged to that rule exist and
   **pass** against the transformed code. If a PRESERVES rule has no covering
   test, generate a characterization test for it as in step 5 (it is, by
   definition, behavior that must survive). A failing PRESERVES test is a hard
   `disproven` even if all other tests are green — preserving named business
   rules is the whole point of behavior-preserving modernization.

8. **Assemble the verdict per equivalence level.** Combine the checks using the
   table in "The three equivalence levels". The verdict is the **minimum** of
   all required checks: any disproven/inconclusive check dominates. Compute the
   final `gate` boolean and set `equivalenceVerdict`.

9. **Emit the report and event.** Write `validation.json` + summary to the
   artifact store, return the `gate` boolean to the orchestrator. On
   `gate: false`, populate `failure` so the orchestrator/planner can re-strategy
   (see below). One validation run = one report = one event.

## Coverage on changed code (not whole-repo)

Whole-repo coverage is the wrong metric here for two reasons:

1. **It hides the change.** A 12-file diff inside a 4,000-file repo moves
   whole-repo coverage by a rounding error even if the changed methods are
   completely untested. We need confidence in *this* change, so we measure
   *this* change.
2. **It penalizes pre-existing debt.** Holding a CU to repo-wide coverage would
   block safe changes in legacy code that was never well-tested — the opposite
   of what a modernization autopilot should do.

So validation restricts JaCoCo to the diff's changed lines/methods and gates on
the **`minCoverageDelta`** from `gates.validation` in the IR. `minCoverageDelta`
is the minimum allowed change in *changed-code* coverage between the pre-change
and post-change states (default `0.0` — "do not make the changed code *less*
covered than it was"). Tightened per wave/CU it becomes a floor that forces
backfill. The verdict reports both the absolute post-change changed-coverage and
the delta so risk-assessment can weigh thinly-proven changes.

**Optional strength check — mutation testing (PIT).** Line coverage proves the
changed code *ran*, not that the assertions would *catch* a regression. Where
the plan or risk profile demands stronger evidence, run PIT on the changed
classes: a high surviving-mutant count means the (possibly generated) tests are
weak even at 100% line coverage. Surface the mutation score in the report as a
quality signal; treat it as a gate only when the CU explicitly requests it.

## Reporting failures so the planner can re-strategy

A `gate: false` result is not a dead end — it is feedback that routes the CU to
**FAILED → planner re-plan** (state-machine §3.1, §7). Make `failure`
actionable so the next strategy is obvious, not a guess:

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

Guidance the planner relies on:

- **Build/test fail from an incomplete recipe** → suggest switching
  `strategy.preferred` from `recipe` to `llm-patch` (or the declared `fallback`).
- **Coverage `inconclusive` on a large diff** → suggest **shrinking the
  ChangeUnit** so each piece is independently provable.
- **Behavior not expressible as unit tests** → suggest setting
  `equivalence.replay` and recording a baseline for golden validation.
- **PRESERVES test failed** → this is a genuine behavior change; the CU as
  scoped is unsafe and must be re-planned, not retried.

Always attach evidence artifacts (build log span, JaCoCo HTML, failing test
names, replay-diff) so the failure is auditable per the provenance chain.

## Quality bar & invariants

- **No APPLY without an explicit equivalence verdict.** The gate boolean must be
  `true` *and* backed by an `equivalenceVerdict: "proven"` at the required
  level. "Tests happened to pass" is not a verdict; "build is green" alone is
  only enough for `syntactic`.
- **Characterization tests record BASELINE behavior, then run on the transformed
  code.** A characterization test whose expected values were taken from the
  *post-change* code is worthless — it asserts the change against itself and can
  never detect a regression. Always capture against the pre-change baseline.
- **Coverage is measured on changed code, gated on `minCoverageDelta`** — never
  whole-repo.
- **PRESERVES rules are mandatory for behavioral/golden CUs.** A failing
  preserved-rule test overrides all other green results.
- **Flaky-test handling.** A test that passes on retry is not a regression and
  must not silently fail the gate, nor silently pass it. Re-run suspected
  failures a bounded number of times (e.g. 2 retries) under the pinned seed;
  classify a test as flaky only if it both passed and failed across runs.
  Quarantine flaky tests, record them in `tests.flakyQuarantined`, and surface
  them — never let flakiness mask a real regression in changed code.
- **Determinism & sandbox.** Run in the sandbox with no outbound network beyond
  package mirrors, the pinned toolchain, and a fixed seed/clock where the test
  framework allows, so a re-run reproduces the verdict. Record the toolchain
  hash in provenance.
- **Idempotent & cache-aware.** Re-validating an unchanged diff reuses the build
  cache and prior report (keyed by diff + toolchain hash); only changed inputs
  trigger a rebuild — respecting the validator-pool bottleneck.
- **Never apply unvalidated LLM output.** If the diff came from `llm-patch`, it
  receives the same (or stricter) proof burden as any recipe — usually
  behavioral with characterization backfill.

## Definition of done

The candidate diff has been built and tested in its sandbox worktree under the
pinned toolchain; coverage of the changed code is measured against baseline and
the `minCoverageDelta` gate; characterization tests (recorded against baseline
behavior) backfill any under-covered changed code when the CU allows it; replay
has run and its diff is incorporated where `equivalence.replay` is set; the
PRESERVES business rules' tests pass for behavioral/golden CUs; and a
`validation.json` report exists with a build status, test results, changed-code
coverage delta, an explicit `equivalenceVerdict` at the required level, and a
boolean `gate` — `true` only when every required check is proven, `false` with
an actionable `failure` (phase + suggested re-strategy + evidence) otherwise.
