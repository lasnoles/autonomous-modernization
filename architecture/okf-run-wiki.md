# OKF Run Wiki — the run as an indexed knowledge bundle

Every run produces (and continuously updates) an **OKF bundle**: a directory of
cross-linked Markdown files with YAML frontmatter, where **the file path is the
concept's identity** and ordinary Markdown links turn the directory into a
traversable knowledge graph. This single bundle is **both the live monitor and
the permanent wiki** — and its `log.md` is the state-machine transition record,
so the OKF bundle *is* the state machine made browsable.

OKF reference: Open Knowledge Format (Google Cloud), markdown + YAML frontmatter,
`index.md` for progressive disclosure, `log.md` for chronological history,
plain-markdown + stable paths + git-versioned. The `run-historian` skill
authors and maintains the bundle.

## 1. Bundle layout

```
wiki/<runId>/
├── index.md                      # entry point — progressive disclosure (= live STATUS.md)
├── log.md                        # chronological state-transition + token log (THE state machine)
├── objective.md                  # the run objective (from Target Spec Intent)
├── key-results/
│   ├── index.md                  # KR scoreboard (acceptance criteria → status)
│   └── kr-01-build-green.md      # one file per key result
├── stages/
│   ├── index.md                  # the pipeline, each stage's status + tokens
│   ├── stage-INDEX.md, stage-RECOVER.md, … stage-INTEGRATE.md  # one file per stage
│   │                             # NOTE: prefix `stage-` so a stage page never
│   │                             # case-collides with index.md on case-insensitive
│   │                             # filesystems (Windows/macOS-default).
├── entry-points/
│   ├── index.md                  # full entry-point inventory (all surfaces)
│   └── http-post-invoices.md     # one file per entry point → links its flow(s)
├── flows/
│   ├── index.md                  # all explored flows + coverage
│   └── flow-charge-invoice.md    # one file per flow → entry point, methods, data, externals
├── change-units/
│   ├── index.md                  # all ChangeUnits + status/risk
│   └── CU-014.md                 # one file per CU → diff, validation, risk, decision
├── gaps/
│   ├── index.md                  # gaps & assumptions register
│   └── GAP-007.md                # one file per gap → placeholder, confidence, whatWouldResolveIt
├── questions/
│   ├── index.md                  # open questions the run is WAITING on (+ answered ones)
│   └── Q-001.md                  # one file per question → prompt, options, blocks, answer
├── decisions/
│   ├── index.md                  # every gate decision (auto/pause/block) + rationale
│   └── d-CU-021-pause.md
├── environment/
│   └── environment.md            # the Podman topology + monitoring (links environment.yaml)
├── e2e/
│   └── e2e-report.md             # end-to-end verification result from prod traces
└── tokens.md                     # token usage rollup (step → CU → stage → repo → run)
```

Paths are stable and git-versioned; links between files are the traversal graph
(e.g. `CU-014.md` links to the `gaps/GAP-007.md` it depends on, the
`flows/*.md` it touches, and the `decisions/*.md` that gated it).

## 2. Frontmatter conventions

Per OKF, `type` is the only required field; we reserve a `type` vocabulary so
agents and humans can navigate deliberately:

```yaml
---
type: change-unit            # run | run-objective | stage | key-result |
                             # entry-point | flow | change-unit | gap | question |
                             # decision | environment | e2e-scenario | tokens | log
title: "CU-014 — javax → jakarta in billing-svc"
description: "API-migration ChangeUnit; applied behind no seam, behavioral equivalence proven."
status: APPLIED              # custom field: mirrors state-machine status
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::ChangeUnit::CU-014        # back-link to the semantic graph
tokens: 47164
tags: [api-migration, jakarta, wave-2]
resource: ../../artifacts/CU-014/diff.patch  # optional URL/relpath to the artifact
links: [ gaps/GAP-007.md, flows/flow-charge-invoice.md, decisions/d-CU-014-auto.md ]
---
```

`gid` ties each wiki concept back to its semantic-graph node, so the wiki and the
graph are two views of one truth.

## 3. `log.md` = the state machine, chronologically

`log.md` is the append-only fold of the event stream — every transition with its
guard result, artifacts, and tokens. Reading it top-to-bottom *is* replaying the
state machine. This is the "use OKF as the state machine" idea: the format is the
canonical, human-browsable transition record.

```markdown
---
type: log
title: "Run log — run_2026_06_28_001"
---
# Run Log

- `10:00:02` **INTAKE** billing-svc → ok · toolchain pinned (JDK21, rewrite 2.x) · 0 tok
- `10:00:40` **INDEX** → ok · 37 entry points, 1,204 nodes · 48.1k tok · [stages/INDEX.md]
- `10:20:11` **PLAN** → ok · 14 ChangeUnits, 4 waves · 91.2k tok · [stages/PLAN.md]
- `11:02:55` **CU-014** VALIDATED→RISK_SCORED→**APPLIED** · risk 0.31 (auto) · 47.2k tok · [change-units/CU-014.md]
- `11:39:10` **CU-021** RISK_SCORED→**PAUSED** · requireHumanFor:seam-extraction · [decisions/d-CU-021-pause.md]
- `10:21:00` **ASK** Q-001 · target language Java vs Go · blocks PLAN · [questions/Q-001.md]
- `10:24:30` **ANSWER** Q-001 · "Java 21" by min_than · PLAN resumes · [questions/Q-001.md]
…
```

`ASK`/`ANSWER` lines record the human-in-the-loop questions and their answers, so
the log shows not just what the agent decided but what it *paused to ask you* and
what you said (`architecture/open-question-resolution.md` §5).

## 4. Live = permanent (no separate reporting step)

The historian updates the bundle on **every transition** (so `index.md` doubles
as `STATUS.md` from `run-state-observability.md` §4.2, and `tokens.md` is the
live token rollup). When the run reaches COMPLETE there is nothing extra to
generate — the wiki is already written. This guarantees the record matches what
actually happened, because it *was* what happened, as it happened.

## 5. Generation rules (for `run-historian`)

- One concept = one file; never bury two concepts in one page.
- Always set `type`; set `gid` whenever the concept maps to a graph node.
- Link, don't duplicate: reference other pages by relative path.
- `index.md` files give progressive disclosure (summary + links down), never the
  full detail.
- Append to `log.md`; never rewrite history (it's the audit trail).
- Keep it plain Markdown with stable paths so it drops straight into GitBook or
  any OKF-aware tool and is di‑able in git.

## 6. Publishing

The bundle is plain Markdown, so it can be: committed to the repo under
`wiki/<runId>/`, published to GitBook (OKF-native), or rendered by any static
site / wiki. The `index.md` is the front door; agents and humans traverse from
there.
