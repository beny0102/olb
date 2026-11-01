# ADR 0002 — Repository topology: polyrepo with a published contract

- **Status:** Accepted
- **Date:** 2026-11-01
- **Deciders:** project maintainer
- **Supersedes:** the monorepo assumption in early drafts of §12 and §15.1

## Context

The first draft of the plan assumed a pnpm + Turborepo monorepo containing every app and package. That assumption was never argued for; it was copied from the default habit of JavaScript projects.

The system, however, is not one program. It is seven artifacts with genuinely different toolchains, release cadences and deployment targets:

- TypeScript services (API, viewer, scorer) built with Node tooling.
- A Python worker running a CUDA/ROCm stack, with its own GPU-oriented CI.
- A Godot project that exports video frames and JSON, versioned against an engine release.
- A schema package that must be consumable from both TypeScript and Python.

Constraints that argue against a single repository:

1. **The vision worker's CI cannot share a runner image with the web apps.** A GPU job and a Next.js build have nothing in common, and a monorepo pipeline ends up with heavy path filters that approximate separate repos anyway.
2. **The simulator is tied to a Godot version.** A Godot upgrade should not touch the release history of the viewer.
3. **The contract must be versioned independently.** It is the compatibility surface (design principle P2). If it only exists inside a monorepo, there is no way for a third party to build a compatible scorer or tracker — which contradicts the open-source premise.
4. **Portfolio value.** A set of narrow, well-documented repositories with real release histories demonstrates more than one large repository with mixed concerns.

## Decision

**Polyrepo, organised around one published contract.**

| Repository | Responsibility | Stack |
| --- | --- | --- |
| `olb` | Whitepaper, ADRs, compose profiles, acceptance suite, org-level documentation | Docs, Docker Compose |
| `olb-contract` | Canonical event + telemetry schema, codegen, conformance fixtures | JSON Schema, TypeScript, Python |
| `olb-api` | League ERP, game state machine, realtime gateway | NestJS, PostgreSQL, Redis |
| `olb-web` | Public fan viewer and league pages | Next.js, React Three Fiber |
| `olb-scorer` | Scorekeeper PWA, offline-first | Next.js (App Router), IndexedDB |
| `olb-vision` | Camera ingest, calibration CLI, pitch tracking | Python, OpenCV, YOLO |
| `olb-sim` | Synthetic game generator with ground truth | Godot 4 headless |

### The contract as a distribution artifact

This is the part that makes the polyrepo work rather than merely exist.

- `olb-contract` is the single source of truth: JSON Schema plus a small amount of metadata, from which TypeScript (zod) and Python (Pydantic) types are generated.
- It is published to **npm** (`@olb/contract`) and **PyPI** (`olb-contract`), semver-versioned, with the major version embedded in the media type (`olb.event/1`).
- Consumers **pin a range** (`^1.4`) and record the resolved version in their runtime metadata, so a mismatch is visible in production rather than inferred from a stack trace.
- Conformance fixtures ship inside both packages. Every consumer runs the same fixture suite in its own CI, so "compatible with the contract" is a test result, not a claim.
- Breaking changes follow §15.2 of the whitepaper: new major, written migration path, no silent coercion.

### Release train

```
olb-contract  ──▶  olb-api  ──▶  olb-web / olb-scorer
      │
      ├──────────▶  olb-vision
      └──────────▶  olb-sim
```

1. The contract releases first.
2. Consumers upgrade independently, at their own pace, within the pinned range.
3. The acceptance suite in `olb` runs against the **latest** contract and the **latest** consumer images. If it fails, the release train stops — that is the integration gate, and the reason a polyrepo can still ship a coherent product.

### Shared CI

Reusable GitHub workflows live in `olb` under `.github/workflows/`, callable from other repositories. Each repo keeps a 10-line CI file, not a 150-line copy of the same YAML. GPU-specific pipelines stay in `olb-vision` and are never inherited by anyone else.

### Local development

The `olb` repository owns `bootstrap.sh`: it clones the sibling repositories next to itself and starts the requested compose profile. A contributor works with one checkout of `olb` plus the repos they are actually touching. Nothing requires all seven to be present; the compose profiles consume published images by default and local build contexts only when the sibling directory exists.

## Consequences

**Positive**

- Independent release cadence and CI per artifact; a Godot upgrade cannot break the viewer.
- The contract is externally consumable, which is a prerequisite for third-party scorers and trackers.
- Each repository's README can state its scope, its supported contract range, and its deployment target in one screen.
- Smaller, focused repositories are easier to review and to present individually.

**Negative**

- A cross-cutting change requires coordinated pull requests across repositories. There is no atomic commit.
- Version skew becomes a real failure mode. Mitigated by the conformance suite, the pinned range, and the acceptance gate — not eliminated.
- Seven CI configurations, seven release processes, seven issue trackers.
- Local development is heavier: at minimum `olb` + `olb-contract` + the repo being changed.
- Refactors that span the contract and a consumer are slower and need a deprecation window.

**Neutral**

- Dependency bots (Dependabot/Renovate) become necessary rather than optional; they are the mechanism that keeps pins current.
- The meta repository must not become a dumping ground. Its scope is documentation, composition and acceptance — not application code.

## Alternatives considered

| Alternative | Verdict | Reason |
| --- | --- | --- |
| Single monorepo (pnpm + Turborepo) | Rejected | Atomic changes and shared tooling are real benefits, but they do not outweigh the Python/Godot separation and the need for an independently published contract |
| Two repos: contract + everything else | Rejected | Recreates the monorepo problem behind a second name; the GPU and Godot pipelines still do not belong in a web build |
| One repo per layer, not per artifact | Rejected | Layer boundaries in the whitepaper are conceptual; deployment boundaries are what repositories should follow |
| Git submodules pulling the contract into each repo | Rejected | Submodules are a poor developer experience and do not provide semver ranges or registry provenance |

## Revisit triggers

- Cross-repo changes become the majority of pull requests.
- Maintaining seven CI configurations demonstrably costs more than the isolation is worth.
- A third-party consumer needs a contract feature that the release train cannot deliver without a coordinated multi-repo release.

## References

- Design principle P2 (the contract is the product), whitepaper §3
- Contract compatibility policy, whitepaper §15.2
- ADR 0001 (frontend deployment target)
