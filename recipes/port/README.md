# Cross-Language Port Profiles (the `generate` strategy)

Unlike `openrewrite/`, `ts-morph/`, `roslyn/` — which are **deterministic
same-language** AST recipes — this catalog holds **target-language profiles** for
**cross-language ports** (`ChangeUnit.kind == language-port`, e.g. Java→Go). A
port is not a syntactic rewrite; it is a behavior-preserving **regeneration**
proven black-box via replay. See `architecture/multi-language-and-porting.md`.

The `transformation-compiler`'s `generate` rung reads a profile here to synthesize
idiomatic target-language source from the language-neutral spec (InterfaceContracts
+ BusinessRules + Flows + characterization tests + replay cassettes).

## Profile contract (`recipes/port/<lang>/profile.yaml`)

```yaml
language: go
version: "1.22"
moduleLayout: standard          # cmd/ internal/ pkg/ ...
idioms:
  errors: "explicit error returns, no exceptions"
  nullability: "zero values + ok-pattern, no null"
  di: "constructor functions, no annotations"
  concurrency: "goroutines + channels for async seams"
libMappings:                    # source-lib → target-lib intent (guidance, not 1:1)
  "spring-web": "chi or net/http"
  "spring-data-jpa": "pgx + sqlc"
  "jackson": "encoding/json"
  "log4j/slf4j": "log/slog"
scaffold:                       # files the generator always emits
  - "go.mod"
  - "cmd/<svc>/main.go"
  - "internal/<component>/*.go"
  - "internal/<component>/*_test.go"   # ports of characterization tests
equivalence: golden             # ports are validated black-box, never syntactic
```

## Rules

- Profiles are **guidance for generation**, not deterministic transforms — output
  is always a *candidate* that must pass black-box equivalence (replay + E2E).
- The frozen `InterfaceContract` is authoritative: the generated service must
  satisfy the same external contract byte-for-byte at the boundary.
- Unknown target-side libraries/interfaces become placeholders + Gaps
  (`open-question-resolution.md`), raising `gapRisk` — never silently guessed.
- `go/` is the starter profile; add `<lang>/` folders (e.g. `kotlin/`, `rust/`)
  the same way.
