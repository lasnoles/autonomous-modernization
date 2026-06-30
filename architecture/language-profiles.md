# Language Profiles — declare the language, auto-route the pipeline

The autopilot is language-plugin-based. You **declare `source.language`** in the
Target Spec (or let INTAKE detect it); the orchestrator loads that language's
**profile** and every stage consults it to pick the right concrete tools —
parser, dependency resolver, entry-point/ORM detectors, transformation engine,
scanners, and build/test/coverage runners. Nothing downstream hard-codes Java.

> One knob in, the whole path follows: `source.language: python` ⇒ tree-sitter-python
> + Jedi/Pyright resolver, FastAPI/Django/Flask route detectors, SQLAlchemy/ORM
> data-access, LibCST/ruff recipes, pip-audit/bandit scanners, pytest+coverage.py.

This is what makes the system multi-language without per-language forks of the
skills: the skills are written against the **profile interface**, and a profile
is data + a thin set of detectors.

## 1. How a profile is resolved (INTAKE)

```
at INTAKE, per repo:
  1. language = target-spec `source.language`  (else: detect from build files —
     pom/gradle→java, pyproject/requirements→python, package.json→ts, *.csproj→csharp, go.mod→go)
  2. load profiles[language]  (this file)            → the tool/detector bindings
  3. preflight verifies ONLY that profile's tools    (tooling-and-provisioning §3; the
     PREFLIGHT gate filters tooling/manifest.yaml by the active profile's `usedTools`)
  4. record language + profile version in provenance (toolchainHash covers it)
```

Every later stage reads `profile.<area>` instead of assuming Java. A repo whose
language has no profile → blocking Question (never guess a toolchain).

## 2. Profile interface (what each profile must bind)

| Profile field | Consumed by | Java | Python |
|---|---|---|---|
| `indexer.grammar` | semantic-indexer | tree-sitter-java | tree-sitter-python |
| `indexer.resolver` | semantic-indexer | JavaParser+SymbolSolver / JDT + ASM bytecode | Jedi / Pyright (pyright-langserver) |
| `indexer.typing` | semantic-indexer (confidence) | static (resolved ⇒ 1.0) | **dynamic ⇒ most call/dataflow edges < 1.0** |
| `buildSystems` | INTAKE, validation, devops | maven, gradle | poetry, pip, uv, setuptools |
| `depManifests` | semantic-indexer, technical-debt | `pom.xml`, `build.gradle`, lockfiles | `pyproject.toml`, `requirements*.txt`, `poetry.lock`, `Pipfile.lock` |
| `entryPointDetectors` | semantic-indexer §7 | Spring `@RestController`/JAX-RS/`@KafkaListener`/`@Scheduled`/`main` | FastAPI/Flask routes, Django urls+views, Celery `@task`, APScheduler, click/argparse, ASGI/WSGI app |
| `dataAccessDetectors` | semantic-indexer §8 | JPA `@Entity`/Hibernate, JDBC, MyBatis | SQLAlchemy models/Core, Django ORM models, raw DB-API (psycopg/sqlite3), Alembic |
| `recipeEngine` | transformation-compiler | **OpenRewrite** (recipe) | **LibCST** (codemod) + ruff/pyupgrade/django-upgrade |
| `recipeCatalog` | planner, transformation-compiler | `recipes/openrewrite/` | `recipes/python/` |
| `scanners.vuln` | technical-debt §1 | OWASP dependency-check, OSV | `pip-audit`, OSV (PyPI), `safety` |
| `scanners.security` | technical-debt §5 | SpotBugs + find-sec-bugs | **bandit** |
| `scanners.smell` | technical-debt §5 | PMD, SpotBugs, Sonar rules | **ruff**, pylint |
| `scanners.deadcode` | technical-debt §5 | (graph "dead code candidates") | **vulture** + graph |
| `scanners.dup` | code-review (duplication) | **PMD-CPD** | **jscpd** (or pylint similarities) — `go`: dupl; `ts`: jscpd |
| `scanners.complexity` | technical-debt §3 | cyclomatic from L0 | cyclomatic from L0 (or **radon**) |
| `deprecationDetector` | technical-debt §2 | OpenRewrite `FindDeprecated*` | `pyupgrade`/ruff rules, `DeprecationWarning` mining |
| `test.runner` | validation | JUnit via maven/gradle | **pytest** (tox/nox) |
| `test.coverage` | validation | JaCoCo | **coverage.py** |
| `test.mutation` (optional) | validation | PIT | **mutmut** / cosmic-ray |
| `formatter` | transformation-compiler §6 | google-java-format / spotless | **black** + **ruff format** + isort |
| `targetIdiomProfile` (ports) | transformation-compiler `generate` | `recipes/port/<lang>/` | `recipes/port/<lang>/` |

A profile is "complete enough" to run the analysis stages (INDEX→PLAN) once
`indexer.*`, `buildSystems`, `depManifests`, detectors, and `scanners.*` are
bound; EXECUTE additionally needs `recipeEngine`/`recipeCatalog` and `test.*`.

## 3. The two reference profiles

### 3.1 `java` (the mature profile)
Deterministic recipe coverage is strongest here (OpenRewrite). Static typing +
bytecode means most L0 edges resolve at `confidence = 1.0`. This is the profile
the rest of the design uses as its running example.

### 3.2 `python`
Same rails, two honest differences from Java:

1. **Dynamic typing ⇒ lower-confidence structure.** Call targets, attribute
   types, and data-flow are often inferred, so the indexer emits those edges at
   `confidence < 1.0` with the resolver named (`indexer.resolver`). The existing
   confidence model already routes low-confidence changes to replay/
   characterization instead of auto-apply — so this is handled, not a blocker.
2. **No OpenRewrite-class recipe engine.** Python modernization leans on
   **codemods** (LibCST) + targeted fixers (`pyupgrade`, `ruff --fix`,
   `django-upgrade`) for the deterministic rung, and drops to `llm-patch` (under
   the same validation gate) more often than Java. The fallback ladder
   (recipe → codemod → llm-patch → manual) is unchanged; Python simply starts at
   the `codemod` rung for most kinds. See `recipes/python/`.

Everything else — black-box equivalence (replay/validation/tester), Podman
environments, planner waves/seams, risk gating, OKF wiki, resume/batching —
is reused unchanged, because it operates on behavior at the interface, not on
Python syntax.

### 3.3 `go` (primarily a PORT TARGET)
Go is the reference **target** for cross-language ports (Java→Go, Python→Go). As
a target it binds:

| Field | Binding |
|---|---|
| `recipeEngine` | **none for same-language** (Go modernization is rare here); ports use the **`generate`** strategy, not AST recipes |
| `targetIdiomProfile` | **`recipes/port/go/profile.yaml`** — module layout, error/nullability/DI/concurrency idioms, source-lib→Go mappings, scaffold files |
| `buildSystems` | go modules (`go.mod`) |
| `build` / `test.runner` | `go build ./...` / `go test ./... -race -cover` (test + coverage are built into the toolchain) |
| `test.coverage` | `go test -coverprofile` + `go tool cover` |
| `scanners.vuln` | **govulncheck** |
| `scanners.security`/`smell` | **staticcheck** |
| `formatter` | **gofmt + goimports** |
| `indexer.*` (only if Go is also a *source*) | tree-sitter-go + go/packages+go/types; chi/net-http & gRPC entry points; pgx/sqlc/GORM data access |

So `target.language: go` routes PLAN to emit `language-port` CUs, the compiler's
`generate` rung to read `recipes/port/go/`, and validation/devops to build, test,
scan, and stand up the generated Go service under Podman — proven black-box
against the frozen interface (`multi-language-and-porting.md`).

## 4. Adding a new language profile

1. Add the language's row to the table in §2 (bind every field).
2. Add its tools to `tooling/manifest.yaml` (grammar, resolver, recipe engine,
   scanners, test runner) with pinned versions + `verify`, tagged `usedByLang`.
3. Add a `recipes/<lang-engine>/` catalog (or reuse an existing engine's).
4. Add the entry-point + data-access detector list the indexer will run (§2).
5. No skill code changes: the skills already read `profile.<area>`. They were
   written against this interface precisely so a new language is data + detectors,
   not a fork.

## 5. Same-language modernize vs. cross-language port

- `source.language == target.language` → **modernization**: use the profile's
  `recipeEngine` (AST recipes/codemods). Default path.
- `source.language != target.language` → **port** (`language-port` CU,
  `strategy: generate`): index the source with the *source* profile, generate the
  target with the *target* profile's `targetIdiomProfile`, prove equivalence
  black-box (`multi-language-and-porting.md`). E.g. **`source: java, target: go`**
  or **`source: python, target: go`** → both read `recipes/port/go/profile.yaml`,
  build/test with the `go` toolchain, and cut over behind an `http-proxy` seam.
