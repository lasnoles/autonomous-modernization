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

You are the behavioral truth-teller and the time machine: you don't transform
code, you prove the POST-change build behaves like the baseline on real I/O, and
you freeze each run so its decisions re-derive later. Invoked by the **validation**
stage (CU state `VALIDATED`) when test coverage is too thin to prove safety or a
ChangeUnit requests record-replay; also contributes the run's audit snapshot.

Authoritative contracts: `architecture/modernization-ir.md` (`equivalence.level ==
golden`, `equivalence.replay`, the `tests`/replay plan you fulfill) ·
`architecture/semantic-graph-schema.md` §7 (portable NDJSON you snapshot) ·
`architecture/deployment-architecture.md` §2/§6/§7 (pinned toolchain —
JDK/recipe-catalog/LLM model+template; sandbox/no-egress; provenance/audit
manifest your snapshot feeds) · `architecture/orchestration-state-machine.md` §3
(the `VALIDATED` "equivalence proven" guard) · `architecture/system-design.md`
§2/§4/§6 (behavior-preserving-by-default; idempotent graph-backed skills;
per-run determinism — recipe + prompt/template versions pinned).

## When to use
- **Thin tests** — `equivalence.tests` give little coverage of targeted behavior; characterization via replay is the only proof.
- **Golden requested** — CU has `equivalence.level == "golden"` or `equivalence.replay == true`; validation MUST call replay, not just build + unit tests.
- **High-risk seam cutover** — a `seam-extraction` CU moves behavior across a network/process/DB boundary (interface-facade, http-proxy, event-bridge); record the seam's I/O on `v1`, replay against `v2` before any flag flip.
- **Audit / repro snapshot** — snapshot portable graph + pins so any decision/diff is reproducible months later.

## Inputs / Outputs
**In:** baseline build (PRE-change artifact — repo `HEAD` or seam `v1`, sandbox-runnable) · transformed build (POST-change — compiler's diff applied in the CU worktree, seam `v2`) · traffic source (recorded prod-like sampled+scrubbed, OR integration suite, OR generated contract-derived/fuzzed load over affected graph endpoints) · ChangeUnit (`targets`, `businessRefs` = behavior that MUST survive, `equivalence.{level,replay}`, `seams[]` = boundaries to record).

**Out:**
- **Recorded trace** — VCR-style cassettes / WireMock mappings + DB query log + captured seam values → `artifacts/<CU>/replay/cassettes/`, content-hashed (recording itself reproducible).
- **Replay-diff report** — `artifacts/<CU>/replay/diff.json` (+ summary): per-interaction inputs, baseline-vs-transformed outputs, normalized diff, each divergence classified benign/behavioral.
- **Verdict** — `{ verdict: "equivalent"|"divergent", level: "golden", behavioralDivergences: N, benignDivergences: M, equivalenceConfidence: 0..1 }`; consumed by validation (VALIDATED guard) and risk-assessment (scoring factor).
- **Run snapshot** — `artifacts/<runId>/snapshot/`: `nodes.ndjson`, `edges.ndjson`, `pins.json` (JDK, recipe catalog version, LLM model+templateVersion, traffic-source hash, cassette hashes).

## Procedure A — Behavioral record-replay (golden check)
```
1. IDENTIFY SEAMS: from CU targets + seams[], + graph query (cypher: endpoints/tables
   reachable from these GIDs; outbound CALLS/READS/WRITES crossing a Component boundary),
   enumerate every nondeterministic/external seam to control:
     network — outbound HTTP/gRPC clients, message producers/consumers
     db      — JDBC/JPA queries + result sets
     clock   — currentTimeMillis, Instant.now, LocalDate.now
     random/UUID — Random, SecureRandom, UUID.randomUUID
     env     — env vars, system props, config, locale/tz
   An uncontrolled seam = a false divergence.

2. RECORD BASELINE in sandbox against traffic, capture enabled:
     network — WireMock/cassette RECORD mode: each outbound HTTP req→resp as replayable mapping
     db      — Testcontainers Postgres/MySQL (or read-through proxy); log query + bind params + result set
     messaging — record consumed + produced messages with keys/headers
     determinism seams — fixed clock (Clock.fixed), seeded Random/SecureRandom, seeded/counter UUID gen,
                         frozen env/locale/tz; record seam values consumed so replay re-injects identical sequence
     golden set — capture baseline HTTP responses/status, emitted messages, final DB writes
   Persist all as the recorded trace, content-hashed.

3. SELECT/GENERATE TRAFFIC: prefer recorded prod-like (sampled, PII-scrubbed) for affected endpoints;
   fall back to integration suite; fill gaps with generated load from graph endpoint/contract shapes
   (boundary + fuzzed). Exercise the businessRefs the CU must preserve. Record coverage → drives equivalenceConfidence.

4. REPLAY against transformed build, sandbox, NO live external calls (deploy-arch §2 no-egress):
     WireMock/cassettes PLAYBACK serve recorded responses; unexpected/unstubbed outbound call = hard failure.
     DB seeded from captured fixtures; recorded clock/seed/UUID/env sequences re-injected.
     Drive identical inputs in identical order; capture transformed outputs.

5. NORMALIZE outputs before diffing — allowlist-explicit, recorded in report, may only canonicalize
   non-determinism, NEVER mask a value change: timestamps/date-formatted fields, generated IDs/UUIDs,
   collection/JSON-key ordering, whitespace, ephemeral ports/hostnames, latency/duration, trace/correlation IDs.

6. DIFF & CLASSIFY interaction-by-interaction (normalized baseline vs transformed):
     benign     — fully explained by a declared normalization rule (new UUID, reordered map). Reported, not failing.
     behavioral — real difference in value/status/message/emitted query effect/branch taken.
                  Any behavioral divergence FAILS unless the CU plan explicitly authorizes that change.

7. EMIT verdict + diff.json (equivalent/divergent, counts, equivalenceConfidence). Return to validation.
   The diff is the evidence; part of the provenance chain.
```

## Procedure B — Run snapshot & reproducibility
```
1. EXPORT portable graph (semantic-graph-schema §7): nodes.ndjson + edges.ndjson, envelopes intact,
   provenance back-links included.
2. CAPTURE pins → pins.json: pinned JDK, recipe catalog version (recipe-bom@x),
   LLM model+templateVersion (deploy-arch §7), traffic-source hash, cassette/fixture hashes.
3. STORE keyed by runId → artifacts/<runId>/snapshot/. Immutable, content-addressed; referenced by the
   audit manifest (deploy-arch §6) so source span → graph fact → IR → diff → validation → risk → PR is traceable.
4. EXPOSE reproduce(runId[, decisionId]): rehydrate portable graph (in-memory/Neo4j), re-apply pins,
   re-derive the requested decision/diff. Idempotent + graph-backed (system-design §4) ⇒ identical
   pins+inputs MUST reproduce original output bit-for-bit (after normalization); a mismatch is itself a
   finding — a determinism seam leaked.
```

## Determinism seams — how non-determinism is controlled
| Seam | Baseline (record) | Replay (re-inject) |
|------|-------------------|--------------------|
| clock | `Clock.fixed(instant, zone)`; record value | same fixed clock |
| random | seeded `Random`/`SecureRandom`; record draws | identical seed/sequence |
| UUID | seeded/counter generator; record issued IDs | identical generator |
| env / props / locale / tz | frozen snapshot; record reads | identical snapshot |
| network | WireMock/cassette **record** | **playback**, no egress |
| db | Testcontainers + query/result capture | seeded fixtures, no live DB |
| ordering | record emission order | replay same order; normalize unordered sets |

Determinism makes the diff *meaningful*: every difference left after seam-pinning + normalization is
attributable to the code change, not the environment (same per-run pinning system-design requires).

## How the verdict is consumed
- **Validation (golden).** For `equivalence.level == "golden"` or `equivalence.replay == true`, the replay `verdict` is the equivalence proof: `equivalent` satisfies the VALIDATED "equivalence proven" clause; `divergent` fails the CU to `FAILED`, attaching `diff.json`, which feeds the planner for re-strategy.
- **Risk-assessment (`equivalenceConfidence`).** Read as a RiskScore input (function of targeted-behavior coverage, traffic representativeness, benign/behavioral counts); thin/low-confidence replay raises risk and can push a CU above `autoApplyCeiling` even when the verdict is nominally `equivalent`.

## Hard invariants
- **Captures reproducible** — traces + snapshots content-hashed and immutable; same baseline + traffic + pins ⇒ same recording.
- **No live external calls during replay** — runs under no-egress; any unstubbed outbound call (HTTP, DB, broker) is a hard failure, never a silent passthrough (a passthrough makes the diff a lie).
- **Normalization never masks real changes** — only declared, allowlisted non-determinism canonicalized; every rule recorded and applied identically to both sides.
- **Both sides, same inputs, same order** — byte-identical inputs and seam sequences; divergence in *inputs* is a harness bug, not a verdict.
- **Determinism leaks are findings** — a non-reproducible `reproduce(runId)`, or a divergence with no seam explanation, is surfaced, not silenced.
- **Verdict is evidence-backed** — every `equivalent`/`divergent` ships with the diff and the coverage/confidence it derived from.

## Definition of done
**Record-replay:** baseline trace captured + hashed; representative traffic exercised the CU's `businessRefs`; transformed build replayed with all seams controlled and no live egress; outputs normalized via a recorded allowlist; every divergence classified benign/behavioral; verdict + `equivalenceConfidence` + diff emitted for validation and risk-assessment.
**Snapshot:** portable graph + pins stored immutably under `runId`, and `reproduce(runId)` rehydrates the graph and re-derives the decision identically — run auditable, every diff reproducible.
