# Python Codemods (LibCST — primary engine)

The Python counterpart to `recipes/openrewrite/`. Python has no OpenRewrite-class
recipe engine, so the **`codemod` rung** carries most deterministic Python work,
backed by **LibCST** (lossless concrete syntax tree — preserves formatting/
comments) plus targeted fixers. The planner names a recipe id in a ChangeUnit's
`strategy.recipe`; the `transformation-compiler` runs it in the CU's worktree and
captures a unified diff. Selected automatically when `source.language: python`
(see `architecture/language-profiles.md`).

## Engines in this catalog

| Engine | What it does | Determinism |
|---|---|---|
| **LibCST** | AST/CST codemods with full fidelity; the general-purpose transform engine (rename, signature change, API swap, import rewrite, pattern rewrite) | deterministic |
| **pyupgrade** | upgrade syntax to a target Python version (f-strings, `super()`, type syntax, drop `__future__`) | deterministic |
| **ruff** (`ruff check --fix` / `ruff format`) | lint autofixes + formatting (also the formatter) | deterministic |
| **django-upgrade** | Django version-specific migrations | deterministic |
| **llm-fallback** | only when no codemod covers the kind (candidate, always validated) | candidate |

## Catalog contract

Each entry declares the same fields as every recipe catalog (so the planner names
it and the compiler runs it):

```yaml
id: python:libcst.rename-symbol            # engine-namespaced id used in strategy.recipe
appliesTo: [api-migration, pattern-refactor]   # IR ChangeUnit kinds
engine: libcst                              # libcst | pyupgrade | ruff | django-upgrade
version: "libcst@1.5"                        # pinned for provenance
params: { from: "old.module.Func", to: "new.module.Func" }
preconditions: ["buildSystem in [poetry, pip, uv]"]
determinism: deterministic                   # 'candidate' only for llm-fallback
notes: "Idempotent; preserves comments/formatting."
```

## How the compiler runs them (in the sandboxed worktree)

- LibCST: `python -m libcst.tool codemod <module.Codemod> <paths>` (or a project
  `libcst.codemod` runner), pinned interpreter + `libcst` version.
- pyupgrade: `pyupgrade --py<NN>-plus <files>`.
- ruff: `ruff check --fix --select <rules> <paths>` / `ruff format <paths>`.
- django-upgrade: `django-upgrade --target-version <X.Y> <files>`.

Output captured as `git diff`; normalized with `black` + `ruff format` + `isort`
(the profile's `formatter`) so the diff is deterministic.

## Commonly-used recipes (starter set)

| ChangeUnit kind | Recipe id |
|---|---|
| language-level (syntax upgrade) | `python:pyupgrade.to-py312` |
| framework-upgrade (Django) | `python:django-upgrade.to-5_0` |
| api-migration | `python:libcst.rename-symbol`, `python:libcst.replace-call` |
| dependency-upgrade | `python:libcst.import-rewrite` (+ bump in `pyproject.toml`) |
| pattern-refactor | `python:ruff.fix` (e.g. `UP`, `B`, `SIM` rule groups) |
| test-backfill | `python:libcst.unittest-to-pytest` |
| security-fix | `python:libcst.<targeted-fix>` (guided by bandit finding) |

## Adding a recipe

1. Confirm it's idempotent and behavior-preserving (or scope its behavior change).
2. Pin the engine/tool version.
3. Map it to the IR ChangeUnit `kind`(s) it satisfies.
4. Add at least one before/after example under `examples/`.

> When no entry's `appliesTo` covers a CU `kind`, the compiler falls to
> `llm-patch` (candidate-only, validated) — exactly as in the Java path. Python
> just reaches it sooner, since deterministic coverage is thinner than OpenRewrite.
