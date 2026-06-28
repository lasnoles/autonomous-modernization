# Tooling & Provisioning

How the autopilot's tools (OpenRewrite, tree-sitter, Podman, scanners, …) get
onto whatever machine a run executes on — **portably across environments without
sacrificing reproducibility.** The rule that makes both possible:

> **Declare the *what* (pinned), adapt the *how* (per environment), and verify.**
> The install *mechanism* varies by host; the resulting toolchain never does.

## 1. Why not a fixed install script, and why not "let the agent install anything"

- A single `setup.sh` assumes one OS, one package manager, network access, and no
  corporate proxy — it breaks for half of all users. Not portable.
- Letting the agent ad-hoc install "latest" breaks the system's core guarantee:
  deterministic recipes + reproducible diffs. Two machines would get different
  tool versions → different output → `replay`/audit can't reproduce a run.

The resolution is to split the two concerns:

| Concern | Environment-specific? | Where it lives |
|---------|----------------------|----------------|
| **What tools + pinned versions** | No | `tooling/manifest.yaml` |
| **How to install on this host** | Yes | the preflight step (adapted by the agent) |
| **Proof it worked** | No | the per-tool `verify` command in the manifest |

## 2. The provisioning policy

1. **Pinned, never floating.** Every tool's version comes from
   `tooling/manifest.yaml`. The agent installs *that* version, not "latest".
2. **Prefer containers.** When a tool has an `image`, run it via Podman/Docker so
   host differences collapse — this is the most portable path and usually avoids
   host installs entirely (the `devops` layer already runs everything in Podman).
3. **Adapt the host install when needed.** If a tool must run on the host and is
   missing, the agent picks the install mechanism that fits the **detected
   environment** (brew / apt / dnf / winget / sdkman / npm / `go install` /
   dotnet tool), using the manifest's `install:` hints, then re-runs `verify`.
4. **Verify, always.** A tool counts as available only when its `verify` command
   passes at the pinned version. *How* it got there is irrelevant; the version is
   what matters.
5. **No ad-hoc / unpinned installs.** The agent may choose the mechanism, not the
   version, and never installs tools not in the manifest.
6. **Record it.** Resolved versions + a `toolchainHash` go into provenance
   (`semantic-graph-schema.md` §2), so a run is reproducible and auditable.
7. **Can't install or verify → raise a Question.** Surface it via the
   wait-for-answer mechanism (`open-question-resolution.md` §5) — never silently
   skip a stage or guess a substitute.

## 3. Preflight — runs before any tool is used

At **INTAKE** (and at the start of any stage that needs tools), the orchestrator
runs **preflight** against the manifest for the tools that stage `usedBy`:

```
for each required tool in tooling/manifest.yaml (filtered by stage):
  1. run `verify`  → present at pinned version? ✅ done.
  2. else if it has an `image` and a container runtime exists → use the image. ✅
  3. else install the pinned version via the mechanism that fits THIS host
     (manifest `install:` hints), then re-run `verify`. ✅
  4. else → raise a Question (blocking) and wait. ⏸
record resolved versions + toolchainHash → provenance
```

Preflight is **idempotent** (a present, correct tool is left alone) and **cached**
(verified once per environment per run).

## 4. Two run modes

The same manifest serves both ends of the spectrum:

| Mode | How tools arrive | Determinism |
|------|------------------|-------------|
| **Dev machine / Claude Code skill** (varied envs) | preflight: verify → container → adapt-install → verify | guaranteed by pin + verify |
| **Autonomous fleet / CI** (`deployment-architecture.md`) | **pre-baked pinned images**; preflight just verifies (no install) | guaranteed by the image |

So a user on Windows, macOS, or Linux gets a working toolchain (the agent adapts),
while a production fleet uses immutable images (no install at all). Both converge
to the manifest's pinned versions.

## 5. Target build toolchain vs agent tools

Two different things, both pinned:
- **Target build toolchain** (JDK, Maven/Gradle, recipe-catalog version) is
  **detected per repo at INTAKE** and pinned from the Target Spec
  (`target.runtime`) — it must match the code being modernized.
- **Agent capability tools** (tree-sitter, ts-morph, Podman, scanners…) come from
  `tooling/manifest.yaml` and are the same regardless of the repo.

Both are recorded in provenance and reproducible by `replay`.

## 6. Offline / air-gapped

Point the manifest at an **internal mirror/registry** (artifact proxy for
images + packages). Preflight then resolves the same pinned versions without
public internet — matching the sandbox model in `deployment-architecture.md`
(workers have no/limited outbound network except package mirrors).

## 7. Invariants

- A stage never runs a tool that didn't pass `verify` at its pinned version.
- The agent chooses the install *mechanism*, never the *version*, and never
  installs a tool absent from the manifest.
- Every resolved toolchain is recorded in provenance (`toolchainHash`).
- An unresolvable tool is a blocking Question, not a skipped step.
- Container-first: if an `image` exists and a runtime is available, prefer it
  over a host install.
