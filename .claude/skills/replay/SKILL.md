---
name: replay
description: >-
  Golden / record-replay behavioral-equivalence checker and run-reproducibility
  engine for the modernization autopilot. Two jobs: (1) CAPTURE real I/O traces
  (HTTP, DB, messaging, and the clock/random/UUID/env seams) by exercising the
  PRE-change baseline build against representative traffic, then REPLAY the same
  inputs against the POST-change build with externals stubbed from the recording
  and DIFF the outputs to prove behavioral equivalence; (2) SNAPSHOT the portable
  graph (nodes.ndjson/edges.ndjson) plus the pinned toolchain and LLM model+
  template versions, keyed by runId, so any decision or diff can be reproduced
  and audited later. Use when tests are thin, when a ChangeUnit sets
  `equivalence.replay`, when cutting over a high-risk seam, or to build the
  audit/repro snapshot for a run.
---

# Replay — Golden Equivalence & Run Reproducibility

You are the **behavioral truth-teller and the time machine**. You do not
transform code; you prove that the transformed build behaves like the baseline
on real I/O, and you freeze enough of each run that its decisions can be
re-derived later. You are invoked by the **validation** stage (CU machine state
`VALIDATED`) when test coverage is too thin to prove safety on its own or when
a ChangeUnit explicitly requests record-replay, and you contribute the
reproducibility snapshot that backs the run's audit manifest.

Read these contracts before acting and treat them as authoritative:

- `architecture/modernization-ir.md` — `equivalence.level == golden`, the
  `equivalence.replay` flag, and the `tests`/replay validation plan you fulfill
- `architecture/semantic-graph-schema.md` §7 — the **portable NDJSON** form
  (`nodes.ndjson`, `edges.ndjson`) you snapshot for reproducibility
- `architecture/deployment-architecture.md` §2/§6 — pinned toolchain
  (JDK, recipe catalog, LLM model+template), sandbox/no-egress rules, and the
  provenance/audit manifest your snapshot feeds
- `architecture/orchestration-state-machine.md` §3 — the `VALIDATED` guard
  ("equivalence proven") your verdict satisfies
- `architecture/system-design.md` §2/§6 — behavior-preserving-by-default and
  the determinism principle (recipe + prompt/template versions pinned per run)

## When to use

- **Thin tests.** A ChangeUnit's `equivalence.tests` resolve to little or no
  coverage of the targeted behavior — characterization via replay is the only
  way to prove the change is behavior-preserving.
- **Golden equivalence requested.** The CU has `equivalence.level == "golden"`
  or `equivalence.replay == true` — validation MUST call replay, not just
  build + unit tests.
- **High-risk seam cutover.** A `seam-extraction` CU moves behavior across a
  network/process/DB boundary (interface-facade, http-proxy, event-bridge).
  Record the seam's I/O on `v1`, replay it against `v2` before any flag flip.
- **Audit / reproducibility snapshot.** Building the run's provenance manifest:
  snapshot the portable graph + pinned versions so any decision/diff is
  reproducible and a reviewer can answer "prove this change is safe and
  reversible" months later.

## Inputs / Outputs

**Inputs**

- **Baseline build** — the PRE-change artifact (current `HEAD` of the repo, or
  the seam's `v1` implementation) runnable in a sandbox.
- **Transformed build** — the POST-change artifact: the compiler's diff applied
  in the CU's git worktree (the seam's `v2`).
- **Traffic source** — one of: recorded prod-like traffic (sampled, scrubbed),
  the existing integration-test suite, or generated load (contract-derived /
  fuzzed inputs over the affected endpoints from the graph).
- **ChangeUnit** — for `targets`, `businessRefs` (behavior that MUST survive),
  `equivalence.{level,replay}`, and `seams[]` (the boundaries to record).

**Outputs**

- **Recorded trace artifact** — VCR-style cassettes / WireMock mappings + DB
  query log + captured seam values, written to
  `artifacts/<CU>/replay/cassettes/` and content-hashed so the recording is
  itself reproducible.
- **Replay-diff report** — `artifacts/<CU>/replay/diff.json` (+ human-readable
  summary): per-interaction inputs, baseline vs transformed outputs, normalized
  diff, and each divergence classified benign vs behavioral.
- **Equivalence verdict** — `{ verdict: "equivalent" | "divergent",
  level: "golden", behavioralDivergences: N, benignDivergences: M,
  equivalenceConfidence: 0..1 }` consumed by the validation stage to satisfy
  the `VALIDATED` guard and by risk-assessment as a scoring factor.
- **Run snapshot** — `artifacts/<runId>/snapshot/`: `nodes.ndjson`,
  `edges.ndjson`, and a `pins.json` (JDK, recipe catalog version, LLM
  model+templateVersion, traffic-source hash, cassette hashes) for repro/audit.

## Procedure A — Behavioral record-replay (the golden check)

1. **Identify the seams.** From the ChangeUnit `targets` and `seams[]`, and by
   querying the graph for what the targeted code touches (cypher: "endpoints
   and tables reachable from these GIDs", outbound `CALLS`/`READS`/`WRITES`
   crossing a Component boundary), enumerate every nondeterministic or external
   seam to control:
   - **network** — outbound HTTP/gRPC clients, message producers/consumers;
   - **db** — JDBC/JPA queries and their result sets;
   - **clock** — `System.currentTimeMillis`, `Instant.now`, `LocalDate.now`;
   - **random / UUID** — `Random`, `SecureRandom`, `UUID.randomUUID`;
   - **env** — environment variables, system properties, config, locale/timezone.
   A seam you don't control is a seam that produces a false divergence.

2. **Instrument & record the baseline.** Run the baseline build in the sandbox
   against the traffic source with capture enabled:
   - Service virtualization in **record mode** — **WireMock** (record-and-
     playback) or VCR/**cassette**-style interceptors capture each outbound
     HTTP request → response pair as a replayable mapping.
   - DB capture — run against **Testcontainers** Postgres/MySQL (or a captured
     read-through proxy) and log every query + bind params + result set; this
     becomes the replay fixture.
   - Messaging — record consumed messages and produced messages (in/out) with
     their keys/headers.
   - Determinism seams — pin a **fixed clock** (`Clock.fixed`), a **seeded** RNG
     (fixed `Random`/`SecureRandom` seed), a deterministic UUID generator
     (counter/seeded), and a frozen env/locale/timezone. Record the seam values
     actually consumed so replay can re-inject the identical sequence.
   - Capture the baseline outputs (HTTP responses/status, emitted messages,
     final DB writes) as the **golden** set.
   Persist all of the above as the recorded trace artifact, content-hashed.

3. **Select / generate representative traffic.** Prefer recorded prod-like
   traffic (sampled and PII-scrubbed) for the affected endpoints; fall back to
   the integration suite; fill gaps with generated load derived from the
   endpoint/contract shapes in the graph (boundary + fuzzed inputs). Aim to
   exercise the `businessRefs` rules the CU must preserve. Record coverage of
   targeted behavior — it drives `equivalenceConfidence`.

4. **Replay against the transformed build.** Boot the transformed build in the
   sandbox with **no live external calls** (deployment-architecture §2 no-egress
   for validators):
   - WireMock / cassettes in **playback mode** serve the recorded responses;
     unexpected/unstubbed outbound calls are a hard failure (see invariants).
   - The DB is seeded from the captured fixtures; the recorded clock/seed/UUID/
     env sequences are re-injected so the same inputs hit the same seams.
   - Drive the identical inputs in the identical order; capture the transformed
     outputs.

5. **Normalize outputs.** Apply a declared normalization layer before diffing,
   to strip *known* non-determinism: timestamps and date-formatted fields,
   generated IDs/UUIDs, collection/JSON-key ordering, whitespace, ephemeral
   ports/hostnames, latency/duration fields, and trace/correlation IDs.
   Normalization is **allowlist-explicit** and recorded in the report — it may
   only canonicalize non-determinism, never mask a value change.

6. **Diff & classify divergences.** Diff normalized baseline vs transformed
   outputs interaction-by-interaction. Classify each divergence:
   - **benign** — fully explained by a declared normalization rule (e.g. a new
     UUID, reordered JSON map). Reported, not failing.
   - **behavioral** — a real difference in a value, status code, message,
     emitted query effect, or branch taken. Any behavioral divergence fails the
     check unless the CU's plan explicitly authorizes that behavior change.

7. **Emit verdict + diff.** Write `diff.json` and the verdict
   (`equivalent`/`divergent`, counts, `equivalenceConfidence`). Return to
   validation. The diff is the evidence; it is part of the provenance chain.

## Procedure B — Run snapshot & reproducibility

1. **Export the portable graph.** Serialize the current graph to the portable
   form (semantic-graph-schema §7): `nodes.ndjson` + `edges.ndjson`, envelopes
   intact, including provenance back-links.

2. **Capture the pins.** Record everything needed to re-derive a decision:
   pinned JDK version, recipe catalog version (`recipe-bom@x`), LLM
   `model + templateVersion` (deployment-architecture §7 toolchain), the
   traffic-source hash, and the cassette/fixture hashes. Write `pins.json`.

3. **Store keyed by runId.** Write `artifacts/<runId>/snapshot/` to the artifact
   store. The snapshot is immutable and content-addressed; it is referenced by
   the audit manifest (deployment-architecture §6) so source span → graph fact →
   IR → diff → validation → risk → PR is fully traceable.

4. **Expose `reproduce(runId[, decisionId])`.** Given a runId, rehydrate the
   portable graph into an in-memory/Neo4j instance, re-apply the recorded pins,
   and re-derive the requested decision or diff. Because each skill is
   idempotent and graph-backed (system-design §4), re-derivation with identical
   pins + inputs MUST reproduce the original output bit-for-bit (after
   normalization); a mismatch is itself a finding — it means a determinism seam
   leaked.

## Determinism seams — how non-determinism is controlled

| Seam | Baseline (record) | Replay (re-inject) |
|------|-------------------|--------------------|
| clock | `Clock.fixed(instant, zone)`; record value | same fixed clock |
| random | seeded `Random`/`SecureRandom`; record draws | identical seed/sequence |
| UUID | seeded/counter generator; record issued IDs | identical generator |
| env / props / locale / tz | frozen snapshot; record reads | identical snapshot |
| network | WireMock/cassette **record** | **playback**, no egress |
| db | Testcontainers + query/result capture | seeded fixtures, no live DB |
| ordering | record emission order | replay in same order; normalize unordered sets |

Determinism is what makes a diff *meaningful*: every difference left after
seam-pinning + normalization is attributable to the code change, not the
environment. This is the same per-run pinning the system-design determinism
principle requires (recipe + prompt/template versions pinned).

## How the verdict is consumed

- **Validation stage (golden equivalence).** When a CU has
  `equivalence.level == "golden"` or `equivalence.replay == true`, validation
  treats the replay `verdict` as the equivalence proof: `equivalent` satisfies
  the `VALIDATED` guard's "equivalence proven" clause; `divergent` fails the CU
  to `FAILED`, attaching `diff.json`, which feeds the planner for re-strategy.
- **Risk-assessment (`equivalenceConfidence` factor).** Risk-assessment reads
  `equivalenceConfidence` (a function of targeted-behavior coverage, traffic
  representativeness, and benign/behavioral divergence counts) as an input to
  the RiskScore: thin/low-confidence replay raises risk and can push a CU above
  `autoApplyCeiling` even when the verdict is nominally `equivalent`.

## Quality / invariants

- **Captures must be reproducible.** Recorded traces and snapshots are content-
  hashed and immutable; the same baseline + traffic + pins reproduces the same
  recording.
- **No live external calls during replay.** Replay runs under no-egress; any
  unstubbed outbound call (HTTP, DB, broker) is a hard failure, never a silent
  passthrough — a passthrough would make the diff a lie.
- **Normalization must not mask real changes.** Only declared, allowlisted
  non-determinism may be canonicalized; every normalization rule is recorded in
  the report and applied identically to both sides.
- **Both sides, same inputs, same order.** Baseline and transformed builds see
  byte-identical inputs and seam sequences; divergence in *inputs* is a harness
  bug, not a verdict.
- **Determinism leaks are findings.** A non-reproducible `reproduce(runId)` or a
  divergence with no determinism-seam explanation must be surfaced, not silenced.
- **Verdict is evidence-backed.** Every `equivalent`/`divergent` verdict ships
  with the diff and the coverage/confidence it was derived from.

## Definition of done

For a record-replay invocation: the baseline trace is captured and hashed,
representative traffic exercised the CU's `businessRefs`, the transformed build
was replayed with all seams controlled and no live egress, outputs were
normalized via a recorded allowlist, every divergence is classified
benign/behavioral, and a verdict with `equivalenceConfidence` plus the diff is
emitted for validation and risk-assessment. For a snapshot invocation: the
portable graph + pins are stored immutably under `runId`, and `reproduce(runId)`
rehydrates the graph and re-derives the decision identically — making the run
auditable and its every diff reproducible.
