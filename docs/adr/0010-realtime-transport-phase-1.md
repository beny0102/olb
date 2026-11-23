# ADR 0010 — Phase 1 realtime transport: SSE and polling

- **Status:** Accepted
- **Date:** 2026-11-23
- **Deciders:** project maintainer
- **Supersedes:** none
- **Related:** ADR 0004 (data contract), ADR 0009 (scorer in web)

## Context

The whitepaper specifies a WebSocket gateway with Redis Streams fan-out and
BullMQ jobs, and the `basic` profile originally listed Redis. Phase 1 shipped a
single NestJS API process with an in-process SSE endpoint
(`GET /games/:id/stream`) that emits a snapshot and subsequent folded states.
The public viewer also polls `/games/:id/state` and `/games/:id/events` every
three seconds, and the scorer syncs through the same HTTP API.

That is enough for a seeded game, one API replica and a handful of viewers. It
is not enough for multi-replica fan-out, durable jobs or horizontal scaling.
The implementation was recorded as a known limit in the score-merge case study,
but no ADR accepted the deviation; the whitepaper still described WebSocket as
the transport.

## Decision

1. **Phase 1 uses SSE plus polling.** SSE carries the snapshot/state stream from
   the API process; the viewer keeps a polling fallback so a dropped stream does
   not stop the game. The wire payloads remain the contract's `GameState`.
2. **No Redis, BullMQ or WebSocket in Phase 1.** The `basic` profile runs API,
   Postgres and web only. Jobs that would need a queue run synchronously or as
   scripts.
3. **One API replica is a stated constraint, not an accident.** SSE state lives
   in process memory; the deployment package starts exactly one API process.
4. **The contract stays transport-neutral.** SSE is an adapter over the same
   snapshot/state messages; switching to WebSocket later must not change the
   event or realtime schemas.

## Consequences

- A volunteer deployment is simpler: no Redis service, no extra port, no
  gateway process.
- The viewer works through proxies that support long-lived HTTP and degrades to
  polling when the stream is unavailable.
- Fan-out is bounded by one process. A second API replica would not share SSE
  subscribers and would publish inconsistent revisions.
- Durable jobs, replay of missed deltas and backpressure are deferred.
- The whitepaper, roadmap and stack table must describe this transport until a
  later ADR replaces it.

## Revisit triggers

Revisit before any of these:

- running more than one API replica;
- a pilot league with concurrent viewers that exhausts one process;
- background work that must survive a restart (video ingest, recalibration,
  exports);
- Phase 2 replay needing server-driven deltas beyond a single game stream;
- a client that cannot use SSE or polling.

The replacement is expected to be a WebSocket gateway with Redis Streams, as
specified in the whitepaper, behind the same contract messages.
