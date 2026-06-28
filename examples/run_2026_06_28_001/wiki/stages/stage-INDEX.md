---
type: stage
title: "INDEX — semantic indexing of billing-svc"
description: "Built the L0 semantic graph: 1,204 nodes and a complete entry-point inventory of 37 non-synthetic surfaces."
status: COMPLETE
runId: run_2026_06_28_001
repo: billing-svc
gid: billing-svc::Stage::INDEX
tokens: 48100
tags: [stage, index, semantic-graph, entry-points]
links: [index.md, ../entry-points/index.md, ../flows/index.md, ../log.md, ../tokens.md]
---

# Stage: INDEX

Builds the structural (L0) semantic graph and enumerates **every** entry surface.
There is no assumed front door — the indexer detects all of them
(`architecture/entry-point-discovery.md`).

> File note: this stage page is `stage-INDEX.md` rather than `INDEX.md` because
> the bundle is authored on a case-insensitive filesystem where `INDEX.md` would
> collide with `index.md`. The concept identity (the INDEX stage) is unchanged.

## What it did

- Parsed `billing-svc` (Java 8 / Spring Boot 1.5) and the `web-app` consumer.
- Materialized the L0 graph: `Repo`, `Module`, `Package`, `Class`, `Method`,
  `Field`, `Endpoint`, `Table`, `Dependency` nodes.
- Ran the full entry-point taxonomy detectors and tagged test fixtures
  `synthetic` (excluded from production flows, retained for coverage).

## Key numbers (CANON)

| Metric | Value |
|--------|-------|
| Graph nodes | **1,204** |
| Non-synthetic entry points | **37** |
| Synthetic fixtures (excluded) | 14 |

Entry points by class: HTTP 18 · messaging 6 · scheduled 4 · batch 2 ·
cli/main 1 · startup 3 · webhook 2 · library 1 — **sum 37**.
Full inventory: [`../entry-points/index.md`](../entry-points/index.md).

## Tokens

**48.1k** — see [`../tokens.md`](../tokens.md).

## Links

- Entry-point inventory: [`../entry-points/index.md`](../entry-points/index.md)
- Example endpoints: [`../entry-points/http-post-invoices.md`](../entry-points/http-post-invoices.md),
  [`../entry-points/kafka-payment-events.md`](../entry-points/kafka-payment-events.md),
  [`../entry-points/scheduled-dunning-run.md`](../entry-points/scheduled-dunning-run.md)
- Flows traced from these entry points: [`../flows/index.md`](../flows/index.md)
- Next stage: [RECOVER](RECOVER.md)
