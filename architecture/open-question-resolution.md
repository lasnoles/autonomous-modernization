# Open-Question Resolution & Placeholder Policy

Autonomy means the system **resolves its own open questions** instead of
stalling on a human — but it does so *honestly*, by consulting the sources of
truth in precedence order, recording how each question was answered, and
**never silently skipping** anything it couldn't resolve. Unknown integrations
get a typed placeholder and a tracked gap, never an omission.

This policy binds every skill. It operationalizes `SOURCES_OF_TRUTH.md`.

## 1. The resolution ladder (try each in order)

When a skill hits an open question ("what does this external service return?",
"which constraint applies?", "is this behavior intentional?"), it walks the
authority hierarchy top-down and stops at the first source that answers:

1. **Human gate / prior decision** — already-recorded approvals or answers.
2. **Intent — the Target Spec** (`input/<run>.md`) — objective, constraints,
   preserve list, allowed changes, declared external systems (§ new spec
   section). If the spec answers it, that answer is authoritative.
3. **Reality — source code & the semantic graph** — derive it: read the call
   sites, the config, the schema, the tests, the git history. Most "unknowns"
   are actually *discoverable* from Reality.
4. **System contracts** (`architecture/`, `graph/`) and **documented defaults**
   (e.g. workflow defaults, ontology conventions).
5. **Reasoned inference** — if 1–4 don't answer but the answer can be inferred
   with evidence, infer it and record `confidence < 1.0` + the basis.

Whatever resolves the question is written into provenance (`resolvedBy`,
`basis`, `confidence`) so it is auditable and reproducible.

## 2. When it still can't be resolved

If the ladder yields no answer, classify the unknown and act — **do not ignore
it**:

| Unknown is… | Action |
|-------------|--------|
| An **external interface / dependency** (system, API, schema, event) | Create a **typed placeholder** (§3) + register a **Gap**. Build the integration against the placeholder. |
| **Ambiguous intent** (spec silent, two reasonable readings) | Pick the **safest, most reversible** reading, register a Gap as an Assumption, proceed; flag for review if it touches a PRESERVES rule. |
| **Undocumented behavior** of existing code | Characterize it (capture current behavior via tests/replay), treat the captured behavior as the spec. |
| **Truly blocking & unsafe to assume** (e.g. would change a high-criticality PRESERVES rule, or risk ≥ blockAbove) | Emit `NEEDS_HUMAN`, pause that scope, keep the rest of the run moving. |

Escalation to a human is the **last resort**, reserved for unknowns that are
both *blocking* and *unsafe to placeholder*.

## 3. Typed placeholders — "don't skip the integration"

When an external interface is unknown, the system still builds the integration
surface so the code, the dependent environment, the mocks, and the E2E tests
all *exist and are exercisable* — they just point at a clearly-marked dummy
until real information arrives.

A placeholder is a **first-class, typed artifact**, not a `// TODO`:

```jsonc
{
  "gid": "billing-svc::InterfaceContract::PaymentProvider@external",
  "kind": "InterfaceContract",
  "placeholder": true,
  "protocol": "http",
  "shape": {                         // a plausible, typed dummy contract
    "POST /charge": {
      "request":  { "amount": "decimal", "currency": "string", "token": "string" },
      "response": { "status": "enum[approved,declined]", "providerRef": "string" }
    }
  },
  "confidence": 0.3,
  "source": "inferred-from-callsites",
  "gap": "GAP-007"                   // back-link to the registered gap
}
```

Rules for placeholders:
- **Typed & plausible**, derived from call sites / naming / similar systems —
  not empty. The dependent system, the mock, and the E2E test are all built to
  this shape so the integration path is real and testable.
- **Loudly marked**: `placeholder: true`, low `confidence`, and a `Gap`.
- **Isolated behind a seam/port** so swapping the real contract in later is an
  O(1)-style change, not a rewrite.
- **Replaced, not forgotten**: when the real interface is provided (spec update
  or discovery), the placeholder is superseded and the Gap resolved; everything
  built on it is re-validated.

## 4. The Gaps & Assumptions register

Every unresolved unknown and every placeholder is recorded as a `Gap` node
(graph) and surfaced in an artifact `artifacts/<run>/gaps.md` and the run
report. No gap is invisible.

```jsonc
{
  "gid": "billing-svc::Gap::GAP-007",
  "kind": "Gap",
  "category": "unknown-interface",   // unknown-interface | missing-dep | ambiguous-intent | undocumented-behavior
  "subject": "PaymentProvider external API",
  "status": "placeholdered",         // open | placeholdered | assumed | resolved
  "placeholder": "billing-svc::InterfaceContract::PaymentProvider@external",
  "assumption": "Provider exposes a synchronous /charge returning approved|declined.",
  "confidence": 0.3,
  "blocking": false,
  "whatWouldResolveIt": "Provider OpenAPI spec, or one captured prod request/response pair.",
  "raisesRiskBy": 0.15               // feeds risk-assessment
}
```

A Gap with `confidence < threshold` or `blocking: true` raises the risk score of
any ChangeUnit that depends on it (risk-assessment reads `HAS_GAP`), so the
gate naturally pauses changes built on shaky assumptions.

## 5. Surfacing questions to a human and **waiting** for answers

Some open questions should not be auto-resolved or placeholdered — the agent
should **ask and wait**. When the resolution ladder can't safely answer one (or
when policy says to ask rather than assume), the system raises a **`Question`**,
shows it in the status file + OKF wiki, **blocks the dependent scope**, and does
not proceed past it until you answer.

### 5.1 What gets asked vs assumed (the `questionPolicy`)

Configurable per run in the Target Spec (`autonomy.questionPolicy`):

| Policy | The agent waits (asks) on… | It auto-handles… |
|--------|---------------------------|------------------|
| `autonomous` (default) | only **blocking & unsafe** unknowns (e.g. would change a high-criticality PRESERVES rule; ambiguous *target language*; risk ≥ blockAbove) | ambiguous intent → safest reversible assumption; unknown interface → placeholder + Gap |
| `interactive` | the above **plus** unknown external interfaces and ambiguous intent (it asks instead of placeholdering/assuming) | nothing silently assumed |
| `manual` | every Gap and every assumption | nothing |

So if you want to *decide* every unknown (e.g. "is this Java or Go?", "what does
this API return?") rather than have the agent guess, set `interactive`.

### 5.2 The Question lifecycle

```jsonc
{
  "gid": "billing-svc::Question::Q-001",
  "kind": "Question",
  "prompt": "Target language for billing-svc: keep Java 21, or port to Go?",
  "context": "target.language not set in the spec; objective text is ambiguous.",
  "options": ["Java 21 (modernize in place)", "Go (language-port of Payments)"],
  "default": "Java 21 (modernize in place)",   // what it WOULD assume if forced
  "blocking": true,
  "status": "open",                              // open | answered | withdrawn
  "raisedBy": "planner",
  "askedAt": "2026-06-28T10:21:00Z",
  "blocks": ["billing-svc::stage::PLAN"]
}
```

1. **Raise:** create the `Question`, `RAISES_QUESTION` from the Gap/stage/CU, and
   `BLOCKS` edges to the scope that must wait.
2. **Surface:** write it to `status.json` + `STATUS.md` (top, under **Open
   Questions**) and the OKF wiki `questions/Q-001.md`; append an `ASK` line to
   `log.md`; fire an alert (the run is now waiting on you).
3. **Wait:** the blocked scope pauses (state `NEEDS_HUMAN`). Independent work in
   other repos/CUs keeps moving; a run **never reaches COMPLETE with an open
   blocking Question**.
4. **Answer (you):** see §5.3.
5. **Resume:** the answer is recorded (`ANSWERED_BY`, status `answered`), the
   `Question`→`Decision`/provenance is written, the `BLOCKS` edges clear, and the
   orchestrator resumes that scope from its checkpoint with the answer as an
   authoritative input (Intent-level — it overrides any assumption).

### 5.3 Where you answer

The status file tells you exactly how; any of these work and all resolve to the
same thing:

- **CLI:** `modernize answer <runId> Q-001 "Java 21"` (primary).
- **Answers file:** add to `input/<run>.answers.md` (or the run's spec) an entry
  keyed by question id; the watcher picks it up.
- **Inline in the spec:** filling the missing field (e.g. set `target.language`)
  answers the corresponding question directly.

Answers are **Intent** (top of `SOURCES_OF_TRUTH.md`'s hierarchy): once you
answer, that decision wins over any assumption the agent would have made.

### 5.4 Pause scope

`autonomy.onOpenQuestion` controls blast radius of the wait:
`pause-scope` (default — only the dependent repo/CU waits; the rest proceeds) or
`halt-run` (stop the whole run until answered). Either way the question is shown
and waited on.

## 6. Invariants

- **Never silently skip an integration.** Missing info → placeholder + Gap, not
  omission. A repo with unknown dependencies still ships a buildable, mockable,
  testable integration surface.
- Every self-resolved question records `resolvedBy` + `basis` + `confidence`.
- Every placeholder has a Gap; every Gap has a `whatWouldResolveIt`.
- Assumptions touching a high-criticality PRESERVES rule are flagged for human
  review even if non-blocking.
- When real info supersedes a placeholder, the dependent work is re-validated.
- **Every blocking `Question` is visible in the status file and the OKF wiki the
  moment it is raised, and the dependent scope waits** — the agent does not
  proceed past it on a guess.
- A run never reaches COMPLETE while a blocking Question is open.
- An answer is authoritative (Intent) and is recorded with `answeredBy`/
  `answeredAt`; resume re-enters the blocked scope with the answer applied.
