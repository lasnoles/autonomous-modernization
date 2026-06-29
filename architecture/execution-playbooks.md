# Execution Playbooks & the Post-Discovery Approach Gate (L2 bounded adaptivity)

The macro pipeline is fixed for safety/audit/resume. But on a **complex,
heterogeneous estate** the right *execution method* genuinely differs per
module — a poorly-tested module needs characterization before any edit; a
tangled module needs decoupling before an upgrade; a component being moved to
another language needs generate→shadow→diff→cutover. This document makes that
adaptivity **first-class but bounded**: the planner *selects and composes vetted
playbooks* (it never invents control flow), and surfaces the chosen approach for
review **after discovery, before execution**.

This is "L2" — adapt the *plan*, not the control plane. It changes nothing in
the state machine, the gates, or the per-ChangeUnit `compile→validate→risk→apply`
core.

## 1. What a playbook is

A **playbook** is a named, pre-validated execution sub-workflow for a wave/
ChangeUnit, composed of phases that are themselves existing gated steps. The
planner picks one per CU/wave; EXECUTE runs the CU through its playbook's phases.
Playbooks live in `workflows/` and are the *only* execution shapes allowed — a
situation no playbook covers becomes a `manual` path + a `Question`, never
improvised control flow.

The existing `strangler-fig` workflow **is** the default playbook.

## 2. The catalog

| Playbook | Use when (selection trigger) | Phases (each = gated steps) | Rollback |
|----------|------------------------------|------------------------------|----------|
| `strangler-fig` *(default)* | normal in-place change behind a seam/flag | compile → validate → risk → apply (behind seam, default-off) | flag flip |
| `seam-extraction` | objective extracts a component; extractability ≥ 0.6 | introduce seam → build new path → shadow-verify → incremental cutover → retire | flip flag (O(1)) |
| `characterization-first` | changed-code coverage < threshold (thin tests) | **capture baseline (replay record + characterization tests) FIRST** → then strangler-fig | flag flip |
| `dependency-untangle` | cyclic / high coupling blocks the change | **break cycle via branch-by-abstraction FIRST** → then the upgrade/refactor | revert abstraction |
| `language-port` | `target.language` ≠ source for this scope | freeze interface → capture spec+traces → generate target → stand up both → replay/diff → cutover → retire | route back to source impl (O(1)) |
| `big-bang-with-replay` | small/leaf component where incremental isn't worth it | replace wholesale → prove via golden replay | revert commit |

Playbooks **compose**: `characterization-first` and `dependency-untangle` are
*prefix* phases that run before an inner playbook (usually `strangler-fig`).

## 3. Selection (the planner, from discovery facts)

After L0–L3 are written, the planner selects a playbook per wave/ChangeUnit by
querying the graph — deterministic, evidence-based:

```
for each ChangeUnit / wave scope:
  if target.language ≠ source for this scope        → language-port
  else if changed-code coverage < coverageFloor      → characterization-first (+ inner)
  else if scope on a dependency cycle / coupling>cap  → dependency-untangle (+ inner)
  else if kind == seam-extraction                     → seam-extraction
  else if leaf & tiny & low-risk                       → big-bang-with-replay
  else                                                 → strangler-fig (default)
```

Thresholds (`coverageFloor`, coupling cap, "tiny") live in run config and are
calibratable. The selection + its trigger + rationale are recorded on the IR
(`ChangeUnit.playbook` and the `approach` block, see `modernization-ir.md`).

## 4. The approach gate (post-discovery, pre-execution)

PLAN now emits — alongside the ChangeUnits — an **approach proposal**: which
playbook governs which scope and *why* (the discovery fact that triggered it).
Between PLAN and EXECUTE the orchestrator applies the **approach gate**:

```
autonomy.approachGate:
  auto     → proceed with the selection (logged, not surfaced)
  review   → (default) if any NON-default playbook was selected, surface the
             proposal as a Question and WAIT for approval; all-default → proceed
  always   → always surface and wait, even for an all-default approach
```

When surfaced, it uses the existing question/wait machinery
(`open-question-resolution.md` §5): the proposal appears in `STATUS.md` /
`status.json` / the OKF `decisions/` (and `questions/`) pages, EXECUTE waits, and
you **approve or redirect the method once** — before it spends tokens executing.
Redirect → the planner re-selects (e.g. "use big-bang here, not strangler-fig").

This is the concrete answer to "the actual flow might differ after the code is
scanned": you see and approve the *method* the scan implies, as one reviewable
decision, instead of it emerging ChangeUnit-by-ChangeUnit.

## 5. How a playbook maps onto EXECUTE

EXECUTE dispatches each ChangeUnit through its `playbook`'s phases. Every phase is
an existing gated unit (compile / validate / risk / apply / replay / seam
cutover / role steps), so the safety properties are unchanged — a playbook just
*orders and bundles* them (e.g. adds a characterization phase up front, or a
shadow-diff phase before cutover). The risk gate, validation gate, rollback, and
resume all behave exactly as before.

## 6. Why this helps a complex estate (and when it doesn't)

- **Right method per module:** characterize-once-up-front instead of reactively
  per-CU; decouple-before-upgrade guaranteed, not hand-wired → fewer FAILED CUs
  and less rework.
- **One approval point for "how":** review the approach after the scan, not as it
  unfolds.
- **Reusable, auditable patterns:** "we used `dependency-untangle` here because
  discovery found a cycle" is recorded, repeatable, and reviewable.
- **Not worth it for a simple, well-tested, same-language single repo:** the
  approach is all-default (`strangler-fig`), the gate auto-approves under
  `review`, and nothing changes. The cost is borne only when complexity warrants.

## 7. Invariants (the "bounded" in bounded adaptivity)

- The planner selects **only from the catalog**; it never authors new control flow.
- A scope no playbook fits → `manual` + a `Question`, logged — never improvised.
- Every selection records its **trigger + rationale** on the IR and the wiki.
- The macro state machine, gates, and per-CU core are unchanged; a playbook only
  composes existing gated phases.
- A run never enters EXECUTE with an **unapproved** approach when the gate requires
  approval.
