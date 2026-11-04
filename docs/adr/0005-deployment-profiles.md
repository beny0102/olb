# ADR 0005 — Deployment profiles and support matrix

- **Status:** Accepted
- **Date:** 2026-11-03
- **Deciders:** project maintainer
- **Supersedes:** none
- **Source:** §11.1–11.5 of `docs/vision.md`

## Context

The target deployment is a volunteer league on commodity hardware, not a datacentre. Hardware
arrives in stages: a laptop first, cameras and a GPU later. The packaging has to make that path
incremental, and the support boundary has to be written down so that "it runs on my machine"
does not become the contract.

## Decision

### Compose profiles

| Profile | Services | Target machine |
| --- | --- | --- |
| `basic` | API, Postgres, Redis, web | Any laptop, 2 vCPU / 2 GB |
| `demo` | `basic` + seeded league + looping game | Same, used for evaluation |
| `vision` | `basic` + GPU worker + camera ingest | Mini-PC with a discrete GPU |
| `sim` | `basic` + Godot headless | Development machine |
| `monitoring` | Prometheus, Grafana, OTel collector | Operational deployments |

Profiles are additive and composable. Starting on `basic` and later adding `vision` must never
require a data migration or a reinstall — that property is a test, not an aspiration.

**Implementation status:** `basic` and `demo` are implemented today. `basic` starts Postgres, the
API and the web app; `demo` adds a one-shot seed service that appends a reproducible nine-inning
game. Redis appears in the target definition but is not part of the boot profile yet. `vision`,
`sim` and `monitoring` are reserved names: the compose file does not define their services.

### Reference builds

| Build | Specification | Approx. cost |
| --- | --- | --- |
| Annotator | Existing phone or laptop | €0 |
| Core host | Mini-PC, 4 vCPU, 8 GB RAM, 256 GB SSD | €250–400 |
| Vision | Mini-PC + NVIDIA RTX 3060 12 GB, 2× global-shutter PoE cameras (1080p60), PoE switch, UPS | €1,400–1,800 |
| Development | Ryzen 3 3300X, 46 GB RAM, Radeon RX 5700 XT 8 GB | — |

### Storage planning

| Item | Per game | Notes |
| --- | --- | --- |
| Two 1080p60 streams, 3 h | 40–60 GB | H.264, 6–8 Mbps per stream |
| Event log | < 2 MB | Tens of thousands of small JSON events |
| Telemetry | 5–20 MB | 120 Hz × 2 cameras × ~1 s per pitch |
| Derived stats | negligible | Recomputed |

Video dominates by two orders of magnitude. This is why retention is a product default, not an
afterthought, and why "record everything forever" is not on the roadmap.

### GPU support matrix

- **Supported:** NVIDIA, CUDA, compute capability ≥ 7.5. Reference: RTX 3060 12 GB.
- **Works, not supported:** AMD RDNA1 (`gfx1010`) on ROCm. The development machine in the
  reference builds is exactly this configuration and runs PyTorch successfully; it is also
  outside ROCm's officially supported list, so images pin versions and upgrades can break it.
- **Not viable:** integrated GPUs, < 6 GB VRAM for the vision worker.
- **CPU-only:** the vision profile refuses to start rather than running at 1 fps and generating
  garbage tracks. The rest of the system runs fine without it.

### Operational reality

A volunteer league administrator is not a sysadmin. Therefore: a documented single-command
install, a health endpoint that says what is misconfigured in plain language, automatic TLS via a
reverse proxy profile, and an upgrade path that states explicitly which versions can skip which
releases.

## Consequences

- `docker compose --profile basic up -d` is the supported install today.
- Adding `vision` later is a compose concern plus hardware, not a schema migration.
- The profiles that are not implemented are named as reserved instead of being implied to work.
- Support answers for AMD GPUs and CPU-only vision are decided in advance, not per incident.
