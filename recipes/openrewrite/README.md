# OpenRewrite Recipes (Java — primary engine)

Deterministic, AST-level transformations for Java/JVM. This is the **preferred**
engine for the `transformation-compiler`: recipes are reproducible, reviewable,
and version-pinned. The planner references recipe ids in a ChangeUnit's
`strategy.recipe`; the compiler runs them in the CU's worktree.

## Catalog contract

Each recipe entry (a YAML file in this folder, or a reference to a published
recipe) declares:

```yaml
id: org.openrewrite.java.migrate.jakarta.JavaxToJakarta   # the recipe name
appliesTo: [api-migration]                                # IR ChangeUnit kinds
engine: openrewrite
version: rewrite-recipe-bom@2.x                            # pinned for provenance
params: {}                                                 # recipe options
preconditions: ["module.buildSystem in [maven, gradle]"]
notes: "Idempotent; safe to re-run."
```

## How the compiler runs them

- Maven: `org.openrewrite.maven:rewrite-maven-plugin:run -Drewrite.activeRecipes=<id>`
- Gradle: `rewriteRun` with the recipe activated
- CLI: `rewrite run --recipe <id>` (for repos without a plugin)

All runs happen in the sandboxed worktree; output is captured as a unified diff.

## Commonly-used recipes (starter set)

| ChangeUnit kind | Recipe id |
|-----------------|-----------|
| framework-upgrade | `org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_x` |
| api-migration | `org.openrewrite.java.migrate.jakarta.JavaxToJakarta` |
| language-level | `org.openrewrite.java.migrate.UpgradeToJava21` |
| dependency-upgrade | `org.openrewrite.maven.UpgradeDependencyVersion` |
| pattern-refactor | `org.openrewrite.java.RemoveUnusedImports`, `…UseDiamondOperator` |
| test-backfill | `org.openrewrite.java.testing.junit5.JUnit4to5Migration` |
| security-fix | `org.openrewrite.java.security.*` |

## Adding a recipe

1. Confirm it's idempotent and behavior-preserving (or scope its behavior change).
2. Pin the catalog version.
3. Map it to the IR ChangeUnit `kind`(s) it satisfies.
4. Add at least one example before/after under `examples/`.
