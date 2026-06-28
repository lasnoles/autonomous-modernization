# Roslyn Analyzers & Code Fixes (.NET / C#)

Deterministic AST transformations for C#/.NET repos in a mixed estate (some
modernization programs span JVM and .NET services that share contracts). Used
by the `transformation-compiler` as the `codemod` strategy for .NET repos.

## Catalog contract

```yaml
id: upgrade-nullable-reference-types
appliesTo: [language-level, pattern-refactor]
engine: roslyn
version: Microsoft.CodeAnalysis@4.x
analyzer: Acme.Modernization.NullableAnalyzer        # DiagnosticAnalyzer id
codeFix: Acme.Modernization.NullableCodeFix          # CodeFixProvider
params: {}
preconditions: ["repo.lang == csharp"]
```

## Shape

Transformations are an `DiagnosticAnalyzer` (detects the pattern) plus a
`CodeFixProvider` (rewrites the syntax tree). The compiler applies the fix
across the solution in the worktree and captures the diff:

- Load with `MSBuildWorkspace.OpenSolutionAsync`
- Run analyzer → collect diagnostics → apply code fix → format → diff

## Typical fixes

| ChangeUnit kind | Fix |
|-----------------|-----|
| framework-upgrade | .NET Framework → .NET 8 API swaps |
| language-level | enable nullable refs, pattern matching, records |
| api-migration | Newtonsoft.Json → System.Text.Json |
| security-fix | insecure crypto / deserialization fixes |

## Cross-repo coupling

Same rule as the other engines: a contract-changing CU is paired and
contract-locked with its consumers regardless of their language.
