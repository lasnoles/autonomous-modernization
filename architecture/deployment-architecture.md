# Deployment Architecture

How the autopilot runs in practice — control plane, workers, stores, and the
trust boundaries that keep autonomous changes safe.

## 1. Topology

```
┌──────────────────────────────────────────────────────────────────────┐
│                           CONTROL PLANE                                │
│  ┌────────────────┐   ┌────────────────┐   ┌────────────────────┐     │
│  │  Orchestrator  │   │   Run/Event    │   │     Dashboard      │     │
│  │ (state machine)│──▶│     Store      │◀──│   (read events)    │     │
│  └───────┬────────┘   └────────────────┘   └────────────────────┘     │
└──────────┼────────────────────────────────────────────────────────────┘
           │ dispatch stage jobs
           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          WORKER PLANE (autoscaled)                     │
│  analysis workers  │  transform workers (git worktrees)  │  validators │
│  (index/recover/   │  (compiler: OpenRewrite/ts-morph/   │ (build+test │
│   discover/debt/   │   Roslyn/LLM patch in sandbox)      │  +replay)   │
│   plan/risk)       │                                     │             │
└─────────┬──────────────────────┬───────────────────────────┬──────────┘
          ▼                       ▼                           ▼
   ┌────────────┐         ┌──────────────┐            ┌──────────────┐
   │  Graph DB  │         │ Artifact     │            │   Git host   │
   │ (Neo4j)    │         │ store (S3/FS)│            │ (PRs/branches)│
   └────────────┘         └──────────────┘            └──────────────┘
```

## 2. Runtime substrate

- **Containers / Kubernetes Jobs** per stage execution; ephemeral and
  sandboxed (no outbound network for transform/validate except package
  mirrors). One Job = one stage execution = one set of events.
- **Git worktrees** per ChangeUnit so parallel transforms never collide.
  Worktrees are disposable; only PRs persist.
- **Toolchains pinned per repo** at INTAKE (JDK version, build tool, recipe
  catalog version, LLM model+template version) and recorded in provenance.
- **Agent tools come from a pinned manifest** (`tooling/manifest.yaml`), resolved
  by an INTAKE **preflight** that verifies → prefers containers → adapts the host
  install to the environment → re-verifies. Fleet/CI uses pre-baked pinned images;
  dev machines adapt-install. Either way runs converge to the manifest versions.
  See `architecture/tooling-and-provisioning.md`.

## 3. Trust boundaries & safety

| Boundary | Control |
|----------|---------|
| Source → workers | read-only clone; writes only in worktree |
| Workers → Git host | PRs only; no push to protected branches; no force-push |
| LLM | runs only inside compiler/validation; output never applied unvalidated |
| Autonomy ceiling | risk gate enforced in control plane, not worker |
| Secrets | injected per-job, never written to graph/artifacts |

## 4. Scaling model

- Horizontal: repos and same-wave ChangeUnits fan out across workers.
- Backpressure: orchestrator caps `maxRepos`, `maxCUsPerRepo`, and a global
  validator pool (builds are the bottleneck).
- Caching: Maven/Gradle caches, recipe ASTs, and graph subgraphs are cached
  across runs keyed by source-span hash.

## 5. Environments

| Env | Purpose | Autonomy |
|-----|---------|----------|
| `dry-run` | analyze + plan + compile + validate; never opens PRs | full, no apply |
| `shadow` | apply to throwaway fork; measure equivalence | full |
| `staged` | real PRs behind seams/flags, default-off | up to ceiling |
| `prod-rollout` | progressive flag enablement + monitoring | gated by SLOs |

## 6. Observability & audit

- Every state transition emits a structured event (see
  `orchestration-state-machine.md` §4) to the run/event store.
- Full provenance chain: source span → graph fact → IR ChangeUnit →
  recipe/patch → diff → validation result → risk score → PR → rollout.
- Audit export: a single run produces a signed manifest linking all of the
  above, satisfying "prove this change is safe and reversible".

## 7. Configuration surface (per run)

```yaml
run:
  repos: [billing-svc, web-app]
  environment: staged
  autonomy:
    autoApplyCeiling: 0.4
    blockAbove: 0.8
    requireHumanFor: [seam-extraction, security-fix]
  concurrency:
    maxRepos: 4
    maxCUsPerRepo: 6
  toolchain:
    java: "21"
    recipeCatalog: "rewrite-recipe-bom@2.x"
    llm: { model: "claude-opus-4-8", templateVersion: "2026-06" }
```
