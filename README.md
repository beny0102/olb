# Open League Baseball

**Open-source scorekeeping, league management and game-day visualization for leagues that don't have a Hawk-Eye budget.**

> **Status: D4 + tier 1 + score merge.** The D4 deploy slice runs the API, viewer and scorer with seeded league data, and tier 1 adds admin setup, catalog and roster management, the scorer's game lifecycle and `deploy/up.sh`. The score merge moves the offline scorer into the web app at `/score` and retires the separate process; `up.sh` runs the API and web as plain processes, with optional HTTPS behind Tailscale Serve. There is still no release and no pilot league. The verification tables and limits are in the [D4 case study](docs/case-study.md) and the [score-merge case study](docs/case-study-score-merge.md) — see the [roadmap](#roadmap) for what remains.

---

## The gap

Professional baseball measures everything. Every pitch gets a release point, a spin rate, a plate location, a 3D trajectory rendered live in the browser within a second. That stack — camera arrays, radar, proprietary calibration, per-game licensing — is priced for organisations with broadcast revenue.

Below that tier, the state of the art is often a paper scorebook and a spreadsheet typed up the following week. The data exists, but it dies in a notebook.

Two things are missing between those worlds:

1. **Software that treats a human scorekeeper as a first-class data source**, not as a fallback.
2. **A live visualization layer that degrades gracefully** — running on a laptop with no cameras, improving as hardware arrives, never requiring a rewrite.

Open League Baseball is an attempt at both.

## The approach

Three decoupled layers, joined by a versioned event contract rather than by assumptions about each other.

```mermaid
flowchart TB
    subgraph capture["1 · Capture"]
        A["Scorekeeper PWA<br/>human events"]
        B["Simulator<br/>Godot, synthetic ground truth"]
        C["Vision worker<br/>IP cameras, optional"]
    end

    contract["Canonical event + telemetry contract<br/>versioned · provenance-tagged · append-only"]

    subgraph core["2 · Core"]
        D["League ERP<br/>seasons · rosters · rules"]
        E["Game state machine<br/>play-by-play · derived stats"]
        F["Realtime gateway<br/>snapshot + delta"]
    end

    subgraph client["3 · Fan experience"]
        G["3D field viewer<br/>Replay and live"]
        H["Box score and stats"]
    end

    A --> contract
    B --> contract
    C --> contract
    contract --> D
    D --> E
    E --> F
    F --> G
    F --> H

    classDef hub fill:#0b3d2e,stroke:#1e8f6a,color:#e6fff5
    class contract hub
```

The contract in the middle is the project. Layers can be replaced — a different scorer UI, a better tracker, a different renderer — as long as they speak it. That is why the [data contract](docs/vision.md#5-the-canonical-data-contract) is the first thing specified and the only thing treated as a compatibility surface.

**Scorekeeping first.** The product is useful with zero cameras and zero machine learning. Vision is an accelerator bolted onto a working system, never a precondition for one.

## Repositories

The project is a set of independent repositories joined by a published contract, not a monorepo ([ADR 0002](docs/adr/0002-repository-topology.md)). Each one can be built, versioned and deployed on its own.

| Repository | Responsibility | Stack |
| --- | --- | --- |
| [`olb`](https://github.com/beny0102/olb) | Whitepaper, ADRs, compose profiles, acceptance suite | Docs, Docker |
| [`olb-contract`](https://github.com/beny0102/olb-contract) | Canonical schema, codegen, conformance fixtures | JSON Schema → TS + Python |
| [`olb-api`](https://github.com/beny0102/olb-api) | League ERP, game state machine, SSE gateway | NestJS, Postgres; optional Redis for multi-replica (ADR 0011) |
| [`olb-web`](https://github.com/beny0102/olb-web) | Public viewer, league pages, admin console and scorer at `/score` | Next.js, React Three Fiber |
| `olb-scorer` | Retired: legacy draft export only; scoring lives in `olb-web` `/score` | Next.js, IndexedDB |
| `olb-vision` | Calibration CLI and pitch tracking | Python, OpenCV, YOLO |
| `olb-sim` | Synthetic games with ground truth | Godot 4 headless |

`olb`, `olb-contract` (0.2.0), `olb-api` (pre-alpha), `olb-web` (pre-alpha) and
`olb-scorer` (retired, tag `v0.1.0-legacy`) exist today. `olb-sim` and
`olb-vision` are Phase 0 scaffolds; links go live as each repository is
published.

## What works without cameras

| Capability | `basic` | `demo` | `vision` |
| --- | :---: | :---: | :---: |
| Seasons, teams, rosters, eligibility | ✅ | ✅ | ✅ |
| Live play-by-play from a phone | ✅ | ✅ | ✅ |
| Line score, standings, season stats | ✅ | ✅ | ✅ |
| Per-player box score | — | planned | planned |
| 3D pitch view (synthetic trajectory) | — | ✅ | measured |
| 3D field with animated plays | — | planned | planned |
| Pitch location, speed, break | — | seeded | measured |
| Multi-camera 3D trajectory | — | — | planned |

`demo` is the portfolio path: a seeded league with a reproducible nine-inning
game and a synthetic pitch view, so anyone can evaluate the project with one
command and no data entry. It does not animate full fielding plays yet.

## What this is not

Scope discipline is part of the design. These are *non-goals*, not backlog items:

- **No player tracking in v1.** Vision is scoped to the pitch: release point, plate location, speed, break. Full 18-player tracking and batted-ball 3D are research ambitions, described as such.
- **No native mobile apps.** The scorer is a PWA; the viewer is responsive web.
- **No betting, odds, or wagering integrations.** Ever.
- **No facial recognition**, and no identity inference from video.
- **No redistribution of MLB or other licensed data.** The play-by-play model is inspired by [Retrosheet](https://www.retrosheet.org/) conventions, which are freely published.
- **No video editing suite.** Clips and highlights are out of scope until the core is stable.
- **No "AI-powered" catch-all features.** Every model must name the decision it makes and the error it can produce.

## Deployment profiles

One compose file, four profiles. The point of the DIY story is that a league starts on profile `basic` and never has to migrate away from it.

```bash
# from this repository, with the four repositories side by side in ~/work/olb
cp docker/workspace.dockerignore ../.dockerignore
docker compose --profile basic up -d    # API, Postgres, web.
docker compose --profile demo  up -d    # basic + a seeded nine-inning game.
```

```
docker compose --profile vision up -d   # reserved: GPU worker + camera ingest
docker compose --profile sim   up -d    # reserved: Godot headless simulator
```

`basic` boots from `docker-compose.yml`. `demo` adds a one-shot `seed` service that waits for the
API to be healthy and appends a reproducible nine-inning game, so the box score and standings are
already populated on first boot; a second run detects the seeded game and appends nothing.
`vision` and `sim` remain reserved names documented in
[ADR 0005](docs/adr/0005-deployment-profiles.md), not commands that work yet.
The build context is the parent directory (the API consumes `olb-contract` as a `file:`
dependency), which is why the workspace ignore file is copied one level up.

## Roadmap

| Phase | Deliverable | Exit criterion |
| --- | --- | --- |
| **0 · Foundations** | Data contract v1, repository topology, compose profiles, CI, ADRs | Contract compiles to TypeScript and Python from one source; CI green |
| **1 · Scorekeeping** | League ERP, scorer PWA (offline-first), game state machine, live gateway, 2.5D viewer | A seeded 9-inning game is scored end to end and followed live on a phone |
| **2 · Simulation and 3D** | Godot headless simulator with ground truth, R3F 3D viewer, replay | A synthetic game renders with true 3D trajectories, reproducible from a seed |
| **3 · Pitch vision** | Calibration CLI, single/dual camera pitch tracking | Plate location within a stated tolerance on a physical field |
| **4 · Multi-camera** | Time sync, triangulation, batted-ball flight, player tracking | Research phase; no date committed |

Each phase must leave the previous one working. There is no "big rewrite" slot in this plan.

**Current position:** phase 1 is implemented through tier 1, the score merge and
the privacy baseline. A seeded game can be scored offline in `/score`, synced to
the API and followed in the viewer, but no pilot league has exercised it in
production. Realtime is SSE plus polling: one API replica by default, or Redis
Pub/Sub fan-out and shared login counters behind a Caddy gateway for a
multi-replica deployment (ADR 0010, ADR 0011). The viewer has a 3D pitch view,
not a full 3D field with animated plays. The supervised pilot checklist is
[docs/release-phase-1.md](docs/release-phase-1.md).

> **Deviation on record:** the compatibility surface (contract, API, viewer, portal) was built
> before the league ERP. The order actually followed, and why, is [ADR 0003](docs/adr/0003-build-order.md).

## Reference hardware

| Profile | Spec | Approx. cost |
| --- | --- | --- |
| Annotator only | Any laptop, 2 vCPU / 2 GB RAM | €0 |
| Self-hosted core | Mini-PC, 4 vCPU / 8 GB RAM / 256 GB SSD | €250–400 |
| Vision build | Mini-PC + NVIDIA RTX 3060 12 GB, 2× PoE cameras (global shutter, 1080p60), switch, UPS | €1,400–1,800 |
| Development (this project) | Ryzen 3 3300X, 46 GB RAM, Radeon RX 5700 XT 8 GB on ROCm | — |

Notes that belong in a README rather than a footnote:

- **NVIDIA is the supported path** for the vision worker. AMD works — the machine above runs PyTorch on ROCm — but RDNA1 (`gfx1010`) is outside ROCm's officially supported list, so versions are pinned and breakage is expected on upgrades.
- 8 GB of VRAM handles **one** 1080p stream comfortably, not four. The hardware page in the docs states the actual limits instead of implying headroom.
- Video is the dominant storage cost: roughly 40–60 GB per game for two 1080p60 streams. This drives the retention defaults in the privacy section.

## Privacy

The target audience includes youth leagues, which means the default configuration must be safe for children's data.

- Public viewer defaults to **jersey number and position**, not name or likeness.
- Name publication is an explicit, revocable opt-in per player.
- Video retention defaults to **7 days**, configurable, with a documented deletion path.
- No facial recognition, no biometric templates, no identity inference.
- Consent records are part of the data model, not a policy document.

The Phase 1 baseline implements hidden-by-default names, a per-player guardian
consent record, admin grant/revoke and public fallback to jersey number or
position. Video retention, self-service consent and the erasure workflow are
still specified but not implemented.

See [docs/vision.md §10](docs/vision.md#10-privacy-and-minors).

## Contributing

Design-only phase, so the most useful contribution right now is **review of the data contract**: attack the schema, not the code.

- License: [Apache-2.0](LICENSE), no open-core, no CLA.
- Sign-off required ([DCO](https://developercertificate.org/)), not a contributor agreement.
- Conventional Commits; ADRs for decisions that are hard to reverse.
- Documentation and code in English; the product UI ships in Spanish first.
  String extraction and a second locale are not implemented yet; the hardcoded
  Spanish copy is a known Phase 1 limit, not a shipped i18n system.

## Acknowledgements

The event model borrows heavily from [Retrosheet](https://www.retrosheet.org/)'s public event file conventions — a decades-long act of volunteer archival work that this project would rather build on than reinvent. MLB Gameday is the reference for what a fan-facing live view should feel like; it is a comparison, not an affiliation.
