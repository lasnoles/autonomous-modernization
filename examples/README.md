# Examples

A fully worked example run of the modernization autopilot, materialized as the
artifacts a real run would emit. Use it to see what the system *produces* and how
the pieces link together.

## `run_2026_06_28_001/` — the "billing-jakarta-payments" run

Migrating `billing-svc` from Java 8 + Spring Boot 1.5 → Java 21 + Spring Boot 3.x
and extracting the **Payments** capability behind a strangler-fig seam, with
`web-app` as a cross-repo consumer. The run completed: 14 ChangeUnits applied
(two via human approval), end-to-end verified, one non-blocking gap left open.

### What's here

```
run_2026_06_28_001/
├── wiki/              ← the OKF run wiki (start here: wiki/index.md)
├── status.json        ← machine-readable live status (monitoring / CI)
├── checkpoint.json    ← the resumable checkpoint (crash → resume)
└── artifacts/
    └── environment.yaml  ← the rendered rootless-Podman topology (devops role)
```

### How to read it — start at [`wiki/index.md`](run_2026_06_28_001/wiki/index.md)

The wiki is an **OKF bundle** (Open Knowledge Format): cross-linked Markdown where
the file path is the concept's identity. Follow the links — you never need to read
a directory listing.

- [`wiki/index.md`](run_2026_06_28_001/wiki/index.md) — the front door / live STATUS:
  stage tree, KR scoreboard, open gaps, token rollup.
- [`wiki/log.md`](run_2026_06_28_001/wiki/log.md) — the **state machine** as a
  chronological transition + token log (incl. the two pause → human-approval →
  resume episodes). Reading it top-to-bottom replays the run.
- `wiki/stages/` — one page per pipeline stage (INDEX … INTEGRATE).
- `wiki/entry-points/` — the **full entry-point inventory** (37 across 8 classes —
  proving "no single front door") and `wiki/flows/` — the explored flows.
- `wiki/change-units/`, `wiki/decisions/` — every change and every gate decision.
- `wiki/gaps/GAP-007.md` — the unknown payment-provider interface, handled as a
  **typed placeholder + tracked gap** (never skipped), backed by a Podman mock.
- `wiki/e2e/e2e-report.md` — end-to-end verification (38/38 from production
  traces) and `wiki/environment/environment.md` — the Podman topology + monitoring.
- `wiki/tokens.md` — token cost rolled up step → CU → stage → run.

### What this example demonstrates (the four asks)

| Ask | Where to look |
|-----|---------------|
| **Discover all entry points & flows** (no single entry) | `wiki/entry-points/index.md`, `wiki/flows/index.md` |
| **Monitor which stage the agent is at** | `wiki/index.md` (= STATUS.md), `status.json`; live console in `architecture/run-state-observability.md` §4.1 |
| **Resume after failure + token usage per step** | `checkpoint.json`, `wiki/tokens.md` |
| **OKF wiki of exactly what happened** | the whole `wiki/` bundle; `wiki/log.md` is OKF-as-state-machine |

Plus: typed placeholders for unknown integrations (`wiki/gaps/`), role-based
delivery (developer/devops/tester), and end-to-end certification under Podman
(`wiki/e2e/`, `artifacts/environment.yaml`).

> These files are illustrative output. In a real run they are generated and kept
> continuously up to date by the `run-historian` skill from the orchestrator's
> event stream — the live monitor and the permanent wiki are the same artifact.
