---
name: run-historian
description: >-
  Maintains the run's OKF knowledge bundle (an indexed Markdown hierarchy) and
  the live monitoring artifacts from the orchestrator's event stream. On every
  state transition it appends to log.md, updates the affected concept page,
  refreshes index.md/status.json/STATUS.md, and folds token usage into
  tokens.md — so the wiki IS the live monitor and, at COMPLETE, the finished
  permanent record. Use this whenever events are emitted and to finalize the
  run wiki.
---

# Run Historian — OKF wiki + live monitor

You turn the orchestrator's event stream into a browsable, auditable **OKF
bundle** (`architecture/okf-run-wiki.md`) that is simultaneously the **live
monitor** (`architecture/run-state-observability.md` §4). There is no separate
reporting step: the record is written as the run happens, so it always matches
what actually happened.

Read these contracts first:
- `architecture/okf-run-wiki.md` — bundle layout, frontmatter, `log.md` = state machine
- `architecture/run-state-observability.md` — events, checkpoints, tokens, monitoring
- `architecture/orchestration-state-machine.md` §4 — event schema (incl. `tokens`)

## When to use

- Continuously, on **every** emitted state-machine event (transition).
- On demand to render `STATUS.md`/`status.json` for a watcher or CI.
- At COMPLETE to finalize the bundle (KR scoreboard, totals, audit links).

## Inputs / Outputs

- **Input:** the event stream / latest events, the current checkpoint, and the
  semantic graph (for `gid` back-links and concept detail).
- **Output:** the OKF bundle under `wiki/<runId>/` (see layout below), plus
  `status.json` (machine) and `STATUS.md` (= the bundle's `index.md`).

## Bundle it maintains

```
wiki/<runId>/
  index.md            # live status + progressive disclosure (also served as STATUS.md)
  log.md              # append-only state-transition + token log (the state machine)
  objective.md        key-results/   stages/   entry-points/   flows/
  change-units/       gaps/   decisions/   environment/   e2e/   tokens.md
```

## Procedure (per event)

1. **Append to `log.md`** — one immutable line per transition: timestamp, scope
   (repo/CU), `from→to`, guard/result, artifacts, **tokens**, and a link to the
   concept page. Never rewrite prior lines (it's the audit trail).
2. **Update the concept page** the event is about — create/refresh the file at
   its identity path (e.g. `change-units/CU-014.md`, `stages/INDEX.md`,
   `gaps/GAP-007.md`). Set required `type` frontmatter; set `gid` to the graph
   node; set `status`, `tokens`, `tags`, and `links` to related pages.
3. **Cross-link, don't duplicate** — link a ChangeUnit to the flows it touches,
   the gaps it depends on, and the decision that gated it; link a flow to its
   entry point. Links are the traversal graph.
4. **Refresh `index.md`** (= `STATUS.md`): the live tree — per-repo stage,
   wave/CU progress, token rollup, open gaps, anything PAUSED/BLOCKED. This is
   the human front door and the monitor view.
5. **Rewrite `status.json`**: the machine-readable mirror of the checkpoint
   (current stage per repo/CU, %complete, current skill, tokens) for watchers/CI.
6. **Fold tokens into `tokens.md`**: rollup at step → CU → stage → repo → run
   with model + cost estimate.

## Stage-specific pages it must produce

- `entry-points/index.md` — the **full entry-point inventory** by class (proves
  "no single front door" was honored), each linking to its flow(s).
- `flows/index.md` — every explored flow + coverage; flag entry points with no
  flow as gaps.
- `key-results/index.md` — Target Spec acceptance criteria as KRs with live ✔/✖.
- `decisions/index.md` — every gate decision (auto/pause/block) with rationale.
- `gaps/index.md` — gaps & assumptions register with `whatWouldResolveIt`.
- `questions/index.md` + `questions/Q-*.md` — **open questions the run is waiting
  on** (prompt, options, what it blocks, and the one-line way to answer), plus
  answered ones. Mirror open questions into `STATUS.md`/`status.json` **at the
  top** the instant they're raised, and append `ASK`/`ANSWER` lines to `log.md`
  (`architecture/open-question-resolution.md` §5). This is how a human sees what
  to answer and the run shows it is waiting.

## Finalization (at COMPLETE)

- Mark the run `complete`; finalize the KR scoreboard against acceptance criteria.
- Ensure every concept page exists and back-links resolve (no dangling links).
- Write run totals to `tokens.md`; link the E2E report, environment manifest, and
  audit manifest.
- The bundle is plain Markdown with stable paths → drops into GitBook/any
  OKF-aware tool and diffs cleanly in git.

## Quality / invariants

- `log.md` is append-only and equals the event stream folded — reading it replays
  the run.
- Every page sets `type`; pages mapping to graph nodes set `gid`.
- `index.md`/`status.json` reflect reality within one transition.
- One concept per file; `index.md` gives summaries + links, never full detail.
- The live bundle and the final wiki are the **same artifact** — nothing is
  regenerated at the end that wasn't already true during the run.

## Definition of done

A traversable OKF bundle whose `index.md` shows current/final status, whose
`log.md` is the complete transition+token history, and from which a reader can
reach — by following links — exactly what happened at every stage, every
ChangeUnit, every gap and decision, and what each step cost.
