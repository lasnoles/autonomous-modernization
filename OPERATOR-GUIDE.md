# Operator Guide

For the human running the autopilot: how to start a run, watch where it is,
answer its questions, approve/redirect it, resume it after a crash, roll back,
and read what it did. The `modernize` commands below are the operator interface
to the system described in `architecture/` — one verb per thing you'll actually
do.

> Mental model: **you write the intent, it runs the gated pipeline, you monitor
> and answer/approve when it pauses.** It resolves what it safely can on its own
> (`SOURCES_OF_TRUTH.md`, `architecture/open-question-resolution.md`) and stops
> to ask only when it must — and when it does, it tells you exactly what to type.

---

## 1. Start a run

1. Copy the intent template and fill it in:
   ```
   cp input/target-spec.md input/billing-2026q3.md
   # edit: repos, objective, constraints, target.language, autonomy, environment
   ```
   The fields that most change behavior:
   - `scope.repos`, `scope.objective`, `scope.constraints` (hard rules)
   - **`target.language`** — same as source → modernization; different (e.g. `go`)
     → a cross-language **port** (`architecture/multi-language-and-porting.md`)
   - `autonomy.autoApplyCeiling` / `blockAbove` / `requireHumanFor`
   - `autonomy.questionPolicy` — `autonomous` | `interactive` | `manual`
     (how much it asks vs assumes — see §4)
   - `run.environment` — `dry-run` | `shadow` | `staged` | `prod-rollout` (§2)

2. Launch:
   ```
   modernize run --spec input/billing-2026q3.md
   # → prints a runId, e.g. run_2026_06_28_001
   ```

**Always do a `dry-run` first** — it analyzes, plans, compiles, and validates
against your spec and opens **no PRs**, so you can review the plan it derives
before anything is applied.

---

## 2. Environments (how far it's allowed to go)

| `environment` | What it does | Opens PRs? |
|---------------|--------------|-----------|
| `dry-run` | analyze → plan → compile → validate only | no |
| `shadow` | apply to a throwaway fork; measure equivalence | no |
| `staged` | real PRs behind seams/flags, default-off, up to risk ceiling | yes (gated) |
| `prod-rollout` | progressive flag enablement gated on SLOs | yes (gated) |

---

## 3. Watch where it is

Three views of the **same** source of truth (the event stream) — they never
disagree (`architecture/run-state-observability.md`):

```
modernize watch <runId>      # live console (tree of repos → stages → CUs, tokens, gaps, questions)
modernize status <runId>     # one-shot status (prints status.json summary)
```

- **`STATUS.md`** = the OKF wiki's `wiki/<runId>/index.md` — human-readable, refreshed
  every transition. Open it in any Markdown viewer.
- **`status.json`** — machine-readable mirror for CI/scripts (current stage per
  repo/CU, % complete, tokens, `openQuestions`, SLO summary).
- Symbols: `✔` done · `◐` running · `⏸` paused (needs you) · `✖` failed.

Alerts fire on `PAUSED` / `BLOCKED` / `NEEDS_HUMAN` / `ROLLED_BACK` / `FAILED`
and on any **new open question**, with a deep link to the relevant wiki page.

---

## 4. Answer its questions (it waits for you)

When the agent can't safely resolve something (or your `questionPolicy` says to
ask), it **raises a blocking Question, shows it at the top of the status file,
pauses the dependent scope, and waits** — it will not guess past it
(`architecture/open-question-resolution.md` §5).

See what it's waiting on, then answer:
```
modernize questions <runId>                      # list open questions
modernize answer <runId> Q-001 "Java 21"         # answer one (primary path)
```
Other ways to answer the same thing:
- Add the answer to `input/<run>.answers.md` (keyed by question id), or
- Fill the missing spec field directly (e.g. set `target.language: java`) — that
  answers the corresponding question.

Your answer is **authoritative Intent** — it overrides any assumption — and the
blocked scope resumes from its checkpoint. Tune how often you're asked with
`autonomy.questionPolicy`: `autonomous` (ask only on blocking+unsafe),
`interactive` (also ask on unknown interfaces & ambiguous intent), `manual`
(ask on everything). `onOpenQuestion: pause-scope` lets the rest of the run keep
moving; `halt-run` stops everything until you answer.

---

## 5. Approve, redirect, or reject pauses

A ChangeUnit can pause for two reasons:

| State | Meaning | What you do |
|-------|---------|-------------|
| `PAUSED` (risk in the middle band, or `kind ∈ requireHumanFor`) | safe enough to one-click approve | `modernize approve <runId> <CU-id>` or `modernize reject <runId> <CU-id> "reason"` |
| `BLOCKED` (risk ≥ `blockAbove`) | needs a *plan change*, not just approval | adjust the spec/constraints and `modernize replan <runId>` |
| `NEEDS_HUMAN` (open question) | waiting on an answer | answer it (§4) |

`seam-extraction` and `security-fix` always pause for approval by default
(`requireHumanFor`). Review the CU's wiki page (`wiki/<runId>/change-units/<CU>.md`)
— it shows the diff, validation verdict, risk factors, and rollback — before approving.

---

## 6. Gaps & placeholders (unknown integrations)

The agent never skips an integration; unknown interfaces become **typed
placeholders + tracked gaps**, mocked so the system still runs and is tested.
```
modernize gaps <runId>      # the gaps & assumptions register
```
To upgrade a placeholder to the real thing, supply the info (e.g. add the
provider's OpenAPI to the spec, or fill the `externalSystems` entry) — the real
interface **supersedes** the placeholder and everything built on it is
re-validated. Open gaps appear in `wiki/<runId>/gaps/` with `whatWouldResolveIt`.

---

## 7. Resume after a crash / interruption

```
modernize run --resume <runId>
```
Loads the last checkpoint (or replays the event log), continues each repo/CU from
where it stopped. Guarantees: **no recompute of unchanged work, no duplicate
applies, tokens keep accumulating, open questions re-surface unanswered.** You can
kill a run at any point and resume with no data loss
(`architecture/run-state-observability.md` §2).

---

## 8. Rollback

Regressions roll back automatically (post-apply re-validation / SLO breach →
`workflows/rollback.yaml`), preferring O(1) seam flag-flips over code reverts. To
roll back manually:
```
modernize rollback <runId> <CU-id> --reason "..."
```
The system restores the last-good behavior and feeds the failure back to the
planner for re-strategy.

---

## 9. Cost (token usage)

Every step's token use is attributed and rolled up:
```
modernize cost <runId>      # tokens by step → CU → stage → repo → run
```
Also in `wiki/<runId>/tokens.md`. Set optional per-stage token budgets in the run
config to pause a stage that exceeds its ceiling (a gate, like risk).

---

## 10. Read what happened (the run wiki)

The run continuously emits an **OKF bundle** at `wiki/<runId>/` — the live monitor
*and* the permanent record (`architecture/okf-run-wiki.md`). Start at
`index.md` and follow links:
- `log.md` — the full state-machine transition + token history (incl. `ASK`/`ANSWER`)
- `stages/`, `entry-points/`, `flows/`, `change-units/`, `decisions/`, `gaps/`,
  `questions/`, `environment/`, `e2e/`, `key-results/`, `tokens.md`

It's plain Markdown with stable paths — commit it, or publish to GitBook (OKF-native).
A worked example: `examples/run_2026_06_28_001/wiki/index.md`.

---

## 11. Command cheat-sheet

| Command | Does |
|---------|------|
| `modernize run --spec input/<run>.md` | start a run |
| `modernize run --resume <runId>` | resume after stop/crash |
| `modernize watch <runId>` | live console |
| `modernize status <runId>` | one-shot status |
| `modernize questions <runId>` | list open questions |
| `modernize answer <runId> <Q-id> "…"` | answer a question (unblocks) |
| `modernize approve / reject <runId> <CU-id>` | clear a paused ChangeUnit |
| `modernize replan <runId>` | re-plan after a BLOCKED CU / spec change |
| `modernize gaps <runId>` | gaps & assumptions register |
| `modernize rollback <runId> <CU-id>` | manual rollback |
| `modernize cost <runId>` | token/cost rollup |
| `modernize report <runId>` | final run report + audit manifest |

---

## 12. A typical session

1. `dry-run` the spec; read `wiki/<runId>/index.md` to review the derived plan.
2. Answer any questions it surfaced (`modernize questions` → `modernize answer`).
3. Re-run in `staged`; `modernize watch` it.
4. When it pauses on a `seam-extraction`/`security-fix`, review the CU page and
   `approve` (or `reject`).
5. Let INTEGRATE stand the system up under Podman and verify E2E from prod traces.
6. Read `report` / the OKF wiki; supply real specs for any open gaps and re-run to
   supersede placeholders.

> All commands are the operator interface to the design in `architecture/`. Pair
> this with `README.md` (what the system is) and `SOURCES_OF_TRUTH.md` (who wins
> when two things disagree).
