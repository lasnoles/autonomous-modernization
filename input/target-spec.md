# Target Spec — Modernization Intent Input

> **This is the human-authored input that drives a run.** Copy this file to
> `input/<run-name>.md`, fill it in, and point the `orchestrator` at it. It is
> the **source of truth for INTENT** (what you *want*) — see
> `SOURCES_OF_TRUTH.md`. The `planner` reads it as the IR `scope.objective`,
> `scope.constraints`, and target end-state; everything downstream serves it.
>
> Two parts: a **machine-readable block** (parsed directly into the run config
> + Modernization IR) and **free-form design sections** (your intent in prose,
> which the planner/architect prompts consume). Fill both. Delete the guidance
> comments (`> …`) once filled, or leave them — the parser ignores prose.

---

## 1. Machine-readable run config

> Parsed verbatim into `workflows/modernization.yaml` inputs + IR `scope`/`gates`.
> Keep keys; replace values. Anything you omit falls back to workflow defaults.

```yaml
run:
  name: billing-jakarta-payments-2026q3        # unique run id stem
  owner: min_than_zaw@msi-global.com.sg
  environment: staged                          # dry-run | shadow | staged | prod-rollout

scope:
  repos:                                       # repos in scope + their role
    - { name: billing-svc, role: primary, vcsUrl: "git@…/billing-svc.git", defaultBranch: main }
    - { name: web-app,     role: consumer, vcsUrl: "git@…/web-app.git",    defaultBranch: main }
  objective: >
    Migrate billing-svc from Java 8 + Spring Boot 1.x to Java 21 + Spring Boot
    3.x, and extract the Payments capability behind a strangler-fig seam.
  constraints:                                 # HARD rules — the planner must obey
    - "No breaking change to the public REST API (/api/v1/**)."
    - "Max one service/seam extracted per wave."
    - "No data migration in this run."
  outOfScope:
    - "Frontend redesign."
    - "Database engine change."

target:
  language: java                               # SAME language → modernization (AST recipes).
                                               # DIFFERENT language → a port (e.g. `go`); see below.
  runtime:                                     # the platform end-state
    java: "21"
    frameworks: [ "spring-boot:3.x", "jakarta-ee:10" ]
    buildSystem: maven
  # --- Cross-language port example (uncomment to port a component to Go) ---
  # language: go                               # triggers `language-port` ChangeUnits (generate strategy)
  # portScope: [ "Payments" ]                  # which capability/component to port (not the whole repo)
  # runtime: { goVersion: "1.22", frameworks: [ "chi", "pgx" ], moduleLayout: "standard" }
  # equivalence: golden                        # ports are proven black-box via replay; never syntactic
  # See architecture/multi-language-and-porting.md
  removeDebtCategories: [ deprecated-api, outdated-dep, security-fix ]
  adoptPatterns: [ constructor-injection, jakarta-namespaces, records-for-dtos ]
  retirePatterns: [ field-injection, javax-namespaces, xml-config ]

externalSystems:                               # dependent systems & their interfaces
  # Provide what you know. Anything omitted/unknown is DISCOVERED from code; if
  # still unknown, the autopilot builds a typed PLACEHOLDER + tracked Gap and
  # mocks it — it never skips the integration. See open-question-resolution.md.
  - name: postgres
    sysKind: db
    known: true
    interface: { type: jdbc, schema: "fixtures/billing.sql" }
  - name: rabbitmq
    sysKind: queue
    known: true
    interface: { type: amqp, exchanges: [ "invoices", "payments" ] }
  - name: payment-provider
    sysKind: thirdparty
    known: false                               # interface unknown → placeholder + mock
    interface: { spec: "TODO: provide OpenAPI, or one captured prod req/resp" }

autonomy:                                      # how far it may go without you
  autoApplyCeiling: 0.4                         # risk ≤ this → auto-apply
  blockAbove: 0.8                               # risk ≥ this → block (needs re-plan)
  requireHumanFor: [ seam-extraction, security-fix ]
  questionPolicy: autonomous                    # how much it asks vs assumes (open-question-resolution §5.1):
                                                #   autonomous  = ask only on blocking+unsafe unknowns (default)
                                                #   interactive = also ask on unknown interfaces & ambiguous intent
                                                #   manual      = ask on every gap/assumption
  onOpenQuestion: pause-scope                   # pause-scope (only the dependent repo/CU waits) | halt-run

concurrency:
  maxRepos: 4
  maxCUsPerRepo: 6

acceptance:                                    # run-level definition of done
  - "Build green on Java 21 / Spring Boot 3."
  - "All existing + characterization tests pass; no behavioral regression."
  - "Payments reachable via seam with v1 fallback and O(1) rollback."
  - "Zero remaining deprecated-api / high-severity vuln debt items in scope."
```

---

## 2. Target architecture & design intent  *(free-form)*

> Describe the end-state design. The `architect` and `planner` prompts read this
> as narrative intent; be concrete about boundaries and shapes you want.

**Target component/service shape**
> e.g. "Payments becomes an independently deployable component behind a
> `PaymentGateway` interface; Invoicing stays in billing-svc. No new shared DB
> tables between them."

**Seams to introduce** (where old & new coexist)
> e.g. "Seam `payments`: interface-facade + feature-flag `payments.v2`,
> incremental cutover 1→10→50→100%, default v1, rollback = flip flag."

**Layering / dependency rules to enforce**
> e.g. "web → service → domain → persistence; no domain→web; integration
> isolated behind ports."

**Reference architecture / standards to conform to**
> e.g. link or describe your org's target architecture, naming, API standards.

---

## 2b. External systems & interfaces  *(optional — unknowns become placeholders)*

> List dependent systems and what you know about their interfaces. The more you
> give, the higher-fidelity the mocks and E2E tests. For anything you leave out
> or mark `known: false`, the autopilot discovers what it can from the code and,
> if still unknown, builds a **typed placeholder + mock** and registers a **Gap**
> — the integration is always built and tested, never skipped.

| System | Kind | Interface known? | Where's the spec / how to call it? |
|--------|------|------------------|------------------------------------|
| postgres | db | yes | JDBC; schema in `fixtures/billing.sql` |
| rabbitmq | queue | yes | AMQP; exchanges `invoices`, `payments` |
| payment-provider | third-party | **no** | unknown → placeholder `POST /charge`; provide OpenAPI to upgrade |

> Each unknown above becomes a `Gap` you'll see in the run report, with
> `whatWouldResolveIt`. Supplying the real spec later supersedes the placeholder
> and re-validates everything built on it.

---

## 3. Behavior to preserve  *(critical — feeds PRESERVES / equivalence)*

> List behavior that MUST NOT change. The planner sets these as `PRESERVES`
> targets and the validation stage proves equivalence against them. Be specific;
> "everything" is not actionable — name the high-stakes rules/capabilities.

- Capability: **Invoicing** — `criticality: high`
  - Rule: "An invoice total must equal the sum of its line items."
  - Rule: "Invoices cannot be issued to inactive customers."
- Capability: **Payments** — `criticality: high`
  - Rule: "A payment cannot exceed the outstanding invoice balance."
- Public contracts: `POST /api/v1/invoices`, `POST /api/v1/payments` — request/
  response shapes and status codes are frozen.

> Anything not listed here is still protected by tests/characterization, but
> items listed here get explicit equivalence checks and raise risk if touched.

---

## 4. Allowed behavior changes  *(optional)*

> If you *do* want behavior to change somewhere, say so explicitly — otherwise
> the autopilot treats every change as a regression. Each entry should name the
> old and new behavior and why.

- (none) — this run is behavior-preserving.

---

## 5. Context, known risks, and priorities  *(optional but valuable)*

**Priority order** (when trade-offs arise)
> e.g. 1) safety/reversibility, 2) remove security debt, 3) framework upgrade,
> 4) seam extraction.

**Known landmines**
> e.g. "InvoiceService uses reflection-based serialization; PaymentProcessor
> has thin test coverage — expect characterization tests / replay."

**Deadlines / change windows**
> e.g. "Apply only during staged window; prod rollout gated on SLOs."

---

### How this file is used

| Section | Consumed by | Becomes |
|---------|-------------|---------|
| §1 run config | orchestrator / workflows | run config + IR `scope`/`gates` |
| §2 design intent | architect, planner | target end-state, seam placement |
| §3 preserve list | planner, validation, risk | `PRESERVES` edges, equivalence checks |
| §4 allowed changes | planner, validation | authorized behavior deltas |
| §5 context | planner, risk | sequencing, risk weighting |

Run `environment: dry-run` first — the autopilot will analyze, plan, compile,
and validate against this spec **without opening any PRs**, so you can review
the plan it derives from your intent before anything is applied.
