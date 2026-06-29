# Multi-Language Support & Cross-Language Porting (e.g. Java → Go)

> **Language routing is profile-driven.** Both kinds of work below select their
> tools from the declared `source.language`'s profile
> (`architecture/language-profiles.md`). Java and **Python** are the two reference
> profiles; TypeScript/C#/Go are stubs. The two paths:

The system distinguishes two kinds of work:

- **Modernization** — *same language*, newer version/framework (Java 8 → Java 21,
  Spring Boot 1 → 3; **Python 3.8 → 3.12, Flask → FastAPI, Django 3 → 5**).
  Handled by the profile's deterministic engine — AST recipes for Java
  (OpenRewrite) / TS (ts-morph) / C# (Roslyn), **codemods for Python (LibCST +
  pyupgrade/ruff/django-upgrade)**. This is the default path everywhere else in
  the design.
- **Porting** — *change of language/runtime* (Java → Go, .NET → Java, …). AST
  recipes **cannot** do this: there is no syntactic rewrite from Java to Go. A
  port is a **behavior-preserving regeneration**, validated black-box.

This document defines how porting works without throwing away the rest of the
system — most of it is reused.

## 1. What's already language-agnostic (the enablers)

- **The semantic graph is language-tagged.** Every node carries `lang`; `fqn` is
  "language-native". L0 (structure), L1 (architecture), L2 (business rules,
  capabilities, domain entities) are expressed in behavior/relationships, not
  Java syntax. A Go service indexes into the *same* schema.
- **Equivalence is proven at the interface, not the syntax.** `replay` (record
  prod traces), `validation` (behavioral equivalence), `tester` (E2E from prod
  traces), and `devops` (Podman topology) are all **black-box** — they exercise
  HTTP/queue/DB contracts, which are identical whether the impl is Java or Go.
  *This is the linchpin: cross-language equivalence is checked exactly like
  same-language equivalence, because both compare observable behavior at the
  interface.*

So a Java→Go port reuses L0–L2 facts, the business rules to preserve, the replay
corpus, the mocks, the Podman environment, and the entire validation/risk/
INTEGRATE machinery unchanged.

## 2. What porting adds

### 2.1 Multi-language indexer
`semantic-indexer` becomes language-plugin-based: a tree-sitter grammar +
resolver per language (Java today; Go, C#, … as plugins), all emitting the same
L0 schema. The **source** repo is indexed in its language; the **target**
language has an idiom/scaffold profile (how the L0/L2 concepts map to Go
packages, interfaces, error handling, modules).

### 2.2 IR: a porting ChangeUnit kind
Add `language-port` to the ChangeUnit `kind` set (`modernization-ir.md` §2.1):
a CU that re-expresses a component in the target language. Its `strategy.preferred`
is **`generate`** (a new strategy alongside recipe/codemod/llm-patch/manual) —
there is no recipe. Its `equivalence.level` MUST be `behavioral` or `golden`
(syntactic is meaningless across languages), and `equivalence.replay` is
strongly defaulted on.

### 2.3 A generative transformation path
`transformation-compiler` gains a **generator** mode for `strategy: generate`:
instead of running an AST recipe, it synthesizes target-language source from the
*specification* the graph already holds —
```
inputs to the generator (all language-neutral):
  • the component's InterfaceContracts (the frozen external shape)
  • the L2 BusinessRules + Capabilities it must preserve
  • the Flows through it (entry points → behavior → data/externals)
  • characterization tests + replay cassettes (the executable spec)
  • the target idiom/scaffold profile (Go module layout, frameworks)
output: target-language implementation (a candidate, never auto-applied)
```
This is LLM-driven (the `llm-fallback`/generation engine), so it carries lower
`equivalenceConfidence` and is gated hard — it is *only* trusted after the
black-box equivalence proof passes.

### 2.4 The port seam (strangler-fig across languages)
A `language-port` is delivered behind a **process-level seam**, not an in-code
facade: run the new Go service alongside the Java one behind an `http-proxy` or
`event-bridge` seam (`modernization-ir.md` §4), route a slice of traffic, and
**replay production journeys against both, diffing outputs**. Cutover is
incremental; rollback is O(1) (route back to Java). This is `seam-extraction` +
`replay` composed — exactly the existing INTEGRATE machinery, now comparing two
languages.

### 2.5 Intent: target language in the spec
`input/target-spec.md` gains a `target.language` / generalized `target.runtime`
(e.g. `language: go`, `goVersion`, module layout, frameworks) so the objective
can be "port the Payments component from Java to Go", and the planner can emit
`language-port` CUs.

### 2.6 Recipe catalog
`recipes/` gains a porting profile per target language (idioms, scaffolds, lib
mappings) used by the generator. These are *guidance*, not deterministic recipes;
the deterministic catalogs (openrewrite/ts-morph/roslyn) remain for same-language
modernization.

## 3. Porting procedure (per component)

1. **Freeze the interface.** Pin the component's `InterfaceContract`(s); these are
   the contract the Go impl must satisfy byte-for-byte at the boundary.
2. **Capture the executable spec.** `replay` records production traces for the
   component's flows; `business-discovery`/`tester` ensure the rules and
   characterization tests are complete (gaps → placeholders, as usual).
3. **Generate.** The compiler's generator emits the Go implementation from the
   spec (§2.3), in its own worktree.
4. **Stand up both.** `devops` brings up Java and Go impls behind the proxy seam
   in one Podman topology with monitoring.
5. **Prove equivalence black-box.** Replay the captured journeys against both;
   diff outputs and check SLOs (`validation`/`replay`/`integration-verifier`).
   Any divergence → fix-or-FAIL, feed back to planner.
6. **Cut over incrementally.** Shift traffic Java→Go (1→10→50→100%) with
   monitoring; SLO breach → route back (O(1)).
7. **Retire** the Java component once stable (a separate CU).

## 4. Risk & gating for ports

Ports are inherently higher-risk than recipes: `strategy: generate` means lower
`equivalenceConfidence`, so `risk-assessment` will rarely auto-apply a port —
expect PAUSE/human review by default, and BLOCK if equivalence can't be proven.
Unknown target-side interfaces become placeholders + Gaps (raising `gapRisk`)
just like any other unknown. The black-box equivalence proof is what makes an
otherwise-scary cross-language rewrite safe to ship.

## 5. What stays the same (so this isn't a rewrite of the system)

Graph schema, business discovery, debt, planner sequencing/waves, seams,
validation, replay, risk gating, Podman environments, monitoring, the OKF wiki,
resume/tokens — all unchanged. Porting is a new **ChangeUnit kind + generator
strategy + target-language profile**, riding the existing rails. The reason that
works: the system already separates *what behavior to preserve* (language-neutral)
from *how it's implemented* (language-specific), and proves safety at the
interface.
