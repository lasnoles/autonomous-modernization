# ts-morph Codemods (TypeScript / JavaScript)

Deterministic AST codemods for TS/JS frontends and Node services that often sit
alongside the Java backends being modernized (e.g. a `web-app` calling a
`billing-svc`). Used by the `transformation-compiler` as the `codemod` strategy
for non-JVM repos, and to keep cross-repo HTTP/contract call-sites in sync when
a Java endpoint changes.

## Catalog contract

```yaml
id: rename-endpoint-callsites
appliesTo: [api-migration, config-migration]
engine: ts-morph
version: ts-morph@22.x
entry: codemods/rename-endpoint-callsites.ts        # exports run(project, params)
params: { from: "/api/v1/invoices", to: "/api/v2/invoices" }
preconditions: ["repo.lang in [ts, js]"]
```

## Codemod shape

Each codemod is a TS module exporting a pure transform over a `ts-morph`
`Project`, producing a diff:

```ts
import { Project } from "ts-morph";
export function run(project: Project, params: Record<string, unknown>) {
  // locate nodes via project.getSourceFiles() + AST queries
  // mutate, then project.save() into the worktree
  // return the unified diff for the compiler to capture
}
```

## Typical codemods

| ChangeUnit kind | Codemod |
|-----------------|---------|
| api-migration | update fetch/axios call-sites to new endpoint shape |
| dependency-upgrade | replace deprecated lib imports/usages |
| pattern-refactor | callbacks → async/await, var → const/let |
| config-migration | migrate env/config access patterns |

## Cross-repo coupling

When a Java ChangeUnit alters a shared contract (Endpoint/event/schema), the
planner emits a paired ts-morph CU in the consuming repo, locked on the same
contract GID (see `graph/edge-types.md` §Cross-repo) so producer and consumer
change together.
