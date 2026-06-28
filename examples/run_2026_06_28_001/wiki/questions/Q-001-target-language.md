---
type: question
title: "Q-001 — Target language: Java 21 or port to Go?"
description: "Blocking question raised at PLAN because target.language was ambiguous; answered Java 21."
status: answered
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Question::Q-001
blocking: true
raisedBy: planner
askedAt: 2026-06-28T10:21:00Z
answeredAt: 2026-06-28T10:24:30Z
answeredBy: min_than@msi-global.com.sg
tags: [target-language, blocking, answered]
links: [index.md, ../stages/PLAN.md, ../objective.md, ../log.md]
---

# Q-001 — Target language for billing-svc

**Status:** ✅ answered — **"Java 21 (modernize in place)"** by min_than at
`2026-06-28T10:24:30Z`. PLAN resumed from its checkpoint and emitted
modernization ChangeUnits (not `language-port`).

## The question (as shown in STATUS.md while open)

> **Target language for `billing-svc`: keep Java 21, or port to Go?**
> The objective text was ambiguous and `target.language` was not set in the spec.
> This decides whether PLAN emits same-language modernization ChangeUnits (AST
> recipes) or `language-port` ChangeUnits (cross-language generate strategy), so
> the run **waited** here rather than guessing.

| Option | Effect |
|--------|--------|
| **Java 21 (modernize in place)** ← chosen | OpenRewrite recipes; this run's actual plan |
| Go (port Payments) | `language-port` CU for Payments, `generate` strategy, black-box equivalence |

- **Default if forced:** Java 21 (modernize in place) — the safer, reversible reading.
- **Blocked while open:** `billing-svc::stage::PLAN` (other analysis already done; PLAN held).

## How it was answered

```
modernize answer run_2026_06_28_001 Q-001 "Java 21 (modernize in place)"
```

Equivalently, setting `target.language: java` in `input/<run>.md` would have
answered it up front and the question would never have been raised.

## Provenance

`RAISES_QUESTION` from the ambiguous-intent Gap at PLAN; `BLOCKS → PLAN`;
`ANSWERED_BY → decision` recorded at 10:24:30. See the `ASK`/`ANSWER` lines in
[`../log.md`](../log.md).
