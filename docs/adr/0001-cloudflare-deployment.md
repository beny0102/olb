# ADR 0001 — Frontend deployment target: Cloudflare Workers via OpenNext

- **Status:** Accepted
- **Date:** 2026-11-01
- **Deciders:** project maintainer
- **Supersedes:** none

## Context

Open League Baseball has two deployment audiences that pull in opposite directions:

1. **A league running its own instance.** The project's core promise (design principle P5, *boring technology*) is that a volunteer administrator can run the whole stack from a compose file on a mini-PC. That includes the API, Postgres, Redis and the WebSocket gateway.
2. **Anyone evaluating the project.** A portfolio-grade demo must load instantly, globally, with no install. Asking a reviewer to boot Postgres is asking them not to look.

The fan viewer (`apps/web`) is the public face of both. It is mostly client-side WebGL with a handful of server-rendered, cacheable pages: league, season, team, game, box score. It needs cheap global hosting and predictable cost, not a long-lived stateful server.

Three constraints discovered while evaluating Cloudflare:

- **No stateful WebSocket backend on Workers.** The realtime gateway (§5.8) holds a per-game `seq`, an advisory writer lock and a ring buffer. On Workers that means Durable Objects and a rewrite of the gateway in the DO programming model. NestJS does not run on Workers.
- **No PostgreSQL.** D1 is SQLite: no JSONB, no advisory locks, no partial-index strategy matching the contract. The core data model assumes Postgres.
- **Worker script size is 3 MiB compressed on the free plan, 10 MiB on paid.** Only the SSR bundle counts; client assets (Three.js, R3F) are served as static assets and are not subject to it.

## Decision

**Split the deployment surface. Do not move the core.**

1. `apps/web` — Next.js 16 (App Router) deployed to **Cloudflare Workers** with `@opennextjs/cloudflare` (Node runtime, not Edge).
   - Game, team, season and box-score pages are **statically generated** (SSG) so the Worker bundle stays small and pages are cacheable at the edge.
   - The 3D viewer is mounted as a **client-only island** (`dynamic(..., { ssr: false })`). Three.js and React Three Fiber never enter the SSR bundle.
2. `apps/api` and the realtime gateway — **self-hosted**, Node + Docker Compose, as specified in §11. The public frontend connects over WSS to the self-hosted API.
3. **Demo build** — a separate static export with no backend: a recorded event log replayed client-side by the same viewer renderer used for live games (§8.1). This is the portfolio artifact.
4. **Not adopted at this stage:** D1, Durable Objects, `@cloudflare/next-on-pages` (deprecated in favour of the OpenNext adapter), full Pages Router/Edge runtime deployment.

## Consequences

**Positive**

- Global static hosting at effectively zero cost; static asset requests are unmetered.
- The demo needs no server, no database, no install — a reviewer clicks a URL.
- OpenNext is a compatibility layer, not a runtime. The same Next.js app can be redeployed to a Node host or Vercel if the adapter stalls; lock-in is limited.
- Keeps the self-hosted path intact, so the project's central claim remains true.

**Negative**

- Two deploy targets to maintain (Workers for the web app, compose for the API), with CORS and WSS configuration to keep coherent.
- `@opennextjs/cloudflare` is community-maintained and lags Next.js releases. Mitigation: pin the Next.js minor version and upgrade deliberately, never on `canary`.
- The Worker bundle approaches the free 3 MiB limit for a Next 16 + React 19 SSR build; the paid plan ($5/month) is the realistic baseline.
- Image optimization requires Cloudflare Images (paid) or a custom loader. Decision: skip `next/image` optimization for now and serve sized assets from R2.
- Debugging a transformed bundle produces worse stack traces than plain Node.

**Neutral**

- ISR and `'use cache'` work, but require KV/R2 configuration. The project does not need ISR in Phase 1 because game pages are generated on publish.
- Node middleware introduced in Next.js 15.2 is not supported by the adapter. Nothing in the design depends on it.

## Alternatives considered

| Alternative | Verdict | Reason |
| --- | --- | --- |
| Next.js `output: 'export'` static site | Viable subset | Loses SSR/ISR entirely; acceptable for the demo, insufficient for authenticated admin routes in the same app |
| SvelteKit + `@sveltejs/adapter-cloudflare` | Strong runner-up | First-party adapter, smaller bundles, excellent PWA story; rejected because it would introduce a second framework for a solo maintainer and R3F is better documented than Threlte |
| Astro + React islands | Good for content, weak for app | Most of this product is application state, not content; islands add ceremony without solving realtime |
| Vercel | Rejected | Deviates from the self-hosted principle and adds a vendor with per-seat/usage cost |
| Self-host the frontend too (Node + Next standalone) | Kept as fallback | Already supported by the compose `basic` profile; Cloudflare is an addition, not a replacement |

## Revisit triggers

Reopen this decision if any of the following becomes true:

- The OpenNext adapter is more than one Next.js major behind for more than three months.
- The viewer's SSR bundle cannot be kept under 10 MiB compressed without disabling features.
- A hosted, multi-tenant offering becomes a real product goal — at which point Durable Objects for the realtime gateway deserve their own ADR.
- WebSocket egress or connection costs on the self-hosted side dominate the operating cost of a league instance.

## References

- `@opennextjs/cloudflare` overview and supported versions: <https://opennext.js.org/cloudflare>
- Durable Objects pricing and WebSocket hibernation: <https://developers.cloudflare.com/durable-objects/platform/pricing/>
- Worker size limits: 3 MiB compressed (free), 10 MiB compressed (paid)
- Related sections: §5.8 (realtime envelope), §8 (fan experience), §11 (deployment and hardware)
