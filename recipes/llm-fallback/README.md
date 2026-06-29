# LLM Fallback Patches

The **last resort** in the compiler's strategy ladder, used only when no
deterministic recipe (OpenRewrite / ts-morph / Roslyn) covers a ChangeUnit.
LLM output is a **candidate diff only** — it is never applied without passing
the full `validation` stage, and it carries a lower `equivalenceConfidence`
into `risk-assessment`.

## When the compiler falls back here

1. `strategy.preferred` had no matching recipe, OR
2. the preferred recipe ran but failed/produced no change, AND
3. `strategy.fallback == llm-patch`.

Bespoke, context-specific edits (untangling a one-off legacy pattern, adapting
glue code) are the typical cases. Broad, repeatable migrations should become a
real recipe instead — see "Promotion" below.

## How it works

- Prompt: `prompts/patch-generator.md`, filled with the exact source spans, the
  PRESERVES business rules as guardrails, surrounding type/signature context,
  and style conventions.
- The model returns a minimal unified diff (or `blocked` if the change would
  break a PRESERVES rule), a per-hunk rationale, explicit assumptions, and a
  `confidence`.
- The compiler applies the diff in the worktree, normalizes formatting/imports,
  and hands the candidate to validation. Assumptions become things validation
  must verify.

## Catalog contract

```yaml
id: llm-patch
appliesTo: ["*"]                       # any kind, as fallback
engine: llm
model: claude-opus-4-8                  # pinned per run for reproducibility
templateVersion: "2026-06"
prompt: prompts/patch-generator.md
guardrails: [preserve-business-rules, no-public-api-change-unless-authorized, minimal-diff]
output: candidate-diff                  # NEVER auto-applied
```

## Safety invariants

- Output is candidate-only; validation + risk gates always run.
- Model + template version are pinned and recorded in provenance (so the diff
  is reproducible by the `replay` skill).
- No edits outside the ChangeUnit `targets`.
- A `blocked` return is a valid, expected outcome — better than a wrong patch.

## Promotion to a deterministic recipe

If the same LLM patch shape recurs across CUs/repos, capture it as an
OpenRewrite/ts-morph/Roslyn recipe and move it out of fallback. Track
fallback-frequency per pattern; high-frequency patterns are recipe candidates.
