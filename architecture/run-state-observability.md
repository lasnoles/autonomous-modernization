# Run State, Resumability & Observability

How a human knows **where the agent is**, how the run **survives crashes and
resumes exactly where it left off**, and how **token cost is tracked per step**.
This makes the autopilot operable: monitorable in real time, resumable after any
failure, and accountable for its compute spend.

## 1. The run state store (event-sourced + checkpoints)

State is **event-sourced**: every state-machine transition emits an immutable
event (`orchestration-state-machine.md` §4), and a **checkpoint** is the folded
current state. The event log is the source of truth; checkpoints are a cache for
fast resume and live status.

Hierarchy: `run → repo → stage → changeUnit → role-step`. Each level carries
position, status, input hashes, artifacts, and **token usage**.

### Checkpoint schema

```jsonc
{
  "runId": "run_2026_06_28_001",
  "updatedAt": "2026-06-28T11:42:03Z",
  "status": "running",                 // running | needs-human | paused | blocked | complete | failed
  "openQuestions": [                   // blocking questions the run is waiting on (see open-question-resolution §5)
    { "id": "Q-001", "status": "open", "blocks": ["billing-svc::stage::PLAN"] }
  ],
  "repos": {
    "billing-svc": {
      "stage": "EXECUTE",              // current repo-machine state
      "stagesDone": ["INTAKE","INDEX","RECOVER","DISCOVER","DEBT","PLAN"],
      "changeUnits": {
        "CU-014": { "state": "APPLIED",   "tokens": { "in": 38120, "out": 9044, "total": 47164 } },
        "CU-021": { "state": "PAUSED",    "reason": "requireHumanFor:seam-extraction",
                    "tokens": { "in": 51002, "out": 12380, "total": 63382 } }
      },
      "tokens": { "in": 402110, "out": 98233, "total": 500343 }
    }
  },
  "tokens": { "in": 402110, "out": 98233, "total": 500343 },   // run rollup
  "cursor": { "event": 1487 }          // last folded event (resume point)
}
```

Checkpoints are written **after every transition** (atomic write + fsync), so the
worst-case loss is one in-flight step. An open blocking `Question` is part of the
checkpoint, so a killed-and-resumed run re-surfaces the same pending questions and
keeps waiting — answers are never lost across a restart. When the last blocking
question is answered, `status` flips off `needs-human` and the blocked scope
resumes from its checkpoint.

## 2. Resumability protocol

On (re)start with a `runId`:
1. Load the latest checkpoint; if absent/corrupt, **rebuild it by replaying the
   event log** from event 0 (event-sourcing guarantees this is lossless).
2. For each repo/CU, resume from its persisted state. Because every skill is
   **idempotent and graph-backed**, re-entering a state re-uses cached outputs
   when input hashes are unchanged (`semantic-graph-schema.md` §5) — no
   recomputation, no double-apply.
3. In-flight steps (started, not committed) are **re-run from their last
   checkpoint**, never resumed mid-step. APPLY uses idempotency keys (branch/PR
   per CU) so a retried apply doesn't create duplicates.
4. Continue the state machine. Token counters resume from the checkpoint and
   keep accumulating (so the rollup spans the whole run, across restarts).

A run can be killed at any point and resumed with `--resume <runId>` with no data
loss and no duplicated side effects.

## 3. Token accounting (per step)

Every LLM call is attributed to the (repo, stage, changeUnit, role) it served
and recorded in the event and folded into the checkpoint:

```jsonc
{ "ts":"…", "runId":"…", "repo":"billing-svc", "stage":"DISCOVER",
  "changeUnit": null, "role":"business-discovery",
  "model":"claude-opus-4-8", "tokens":{"in":21044,"out":6231,"total":27275},
  "costEstimate": 0.41 }
```

Rollups are available at every level (step → CU → stage → repo → run) and
rendered in the monitor and the final wiki (`tokens.md`). This answers
"what did each step cost?" and surfaces runaway stages early. Optional per-stage
**token budgets** can pause a stage that exceeds its ceiling (a gate, like risk).

## 4. Monitoring — recommended approach

> **Recommendation: a live, event-sourced "run console" backed by two always-
> current artifacts.** One source of truth (the event stream) drives a CLI/TUI,
> a machine-readable `status.json`, and a human-readable `STATUS.md` — no log
> scraping, works headless/CI, and the same artifacts power resume and the wiki.

### 4.1 Live console (primary)

`modernize watch <runId>` tails the event stream and renders a refreshing tree:

```
run_2026_06_28_001   ● running   elapsed 1h12m   tokens 500.3k (~$8.10)
└─ billing-svc                          EXECUTE  ▓▓▓▓▓▓▓░░ 71%   tokens 500.3k
   ├─ ✔ INDEX      37 entry points, 1,204 nodes        48.1k
   ├─ ✔ RECOVER    9 components, 3 seams               31.4k
   ├─ ✔ DISCOVER   6 capabilities, 41 rules            27.3k
   ├─ ✔ DEBT       58 debt items, 3 CVEs               19.0k
   ├─ ✔ PLAN       14 CUs, 4 waves                      91.2k
   └─ ◐ EXECUTE    wave 3/4
      ├─ ✔ CU-014  jakarta migration      APPLIED       47.2k
      ├─ ⏸ CU-021  payments seam          PAUSED (human) 63.4k
      └─ ◐ CU-022  remove field-injection COMPILED       12.1k
   GAPS: 1 open (GAP-007 payment-provider interface)
   ▶ AWAITING YOUR ANSWER (run paused on these):
     Q-001  Target language for billing-svc: Java 21, or port to Go?
            answer: modernize answer run_… Q-001 "Java 21"
```

It reads only the event stream, so it can attach to a running, paused, or
finished run and reconstruct the exact picture. **Open questions appear at the
top** so you see immediately what the run is waiting on.

### 4.2 `status.json` (machine) + `STATUS.md` (human)

Both are rewritten on every transition (so polling/CI and humans see the same
truth as the console). `STATUS.md` is the OKF bundle's live `index.md` —
monitoring and the wiki are the *same* artifact (`okf-run-wiki.md`).

Both carry an **`openQuestions`** section, listed first when non-empty
(`architecture/open-question-resolution.md` §5) — these are what the run is
blocked on and waiting for you to answer:

```jsonc
// status.json (excerpt)
"openQuestions": [
  { "id": "Q-001", "prompt": "Target language for billing-svc: Java 21, or port to Go?",
    "options": ["Java 21 (modernize in place)", "Go (port Payments)"],
    "default": "Java 21 (modernize in place)", "blocks": ["billing-svc::stage::PLAN"],
    "answerWith": "modernize answer run_2026_06_28_001 Q-001 \"<your choice>\"",
    "raisedBy": "planner", "askedAt": "2026-06-28T10:21:00Z" }
],
"status": "needs-human"        // run-level status flips while a blocking question is open
```

`STATUS.md` renders the same under a top **## Open questions (waiting on you)**
section with the prompt, options, and the one-line way to answer each.

### 4.3 Push alerts

A new **open blocking `Question`**, and transitions to `PAUSED`, `BLOCKED`,
`NEEDS_HUMAN`, `ROLLED_BACK`, or `FAILED`, fire a notification (Slack/email/
webhook) with the reason and a deep link to the OKF question page — humans are
pulled in only when needed, and told exactly what to answer.

### 4.4 Optional web dashboard

A thin web app subscribing to the same event stream gives multi-run overview,
historical token/cost charts, and gap/risk drill-down. It adds no new source of
truth — just another renderer of the event log.

### Why this design

- **One source of truth** (event log) → console, status files, dashboard, resume,
  and wiki can never disagree.
- **Event-sourced** → attach/replay anytime; survives crashes; lossless resume.
- **Headless-friendly** → `status.json` for CI gating, `STATUS.md` for humans,
  no terminal required.
- **Already the wiki** → no separate reporting step; the live view *is* the
  permanent record.

## 5. Invariants

- A checkpoint is written after every transition; the event log can always
  rebuild it.
- Resume causes no recomputation of unchanged work and no duplicated side
  effects (idempotency keys on APPLY).
- Every LLM call is attributed to a (repo, stage, CU, role) and counted.
- The monitor reflects state within one transition of reality.
