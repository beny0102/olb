# ADR 0011 — Multi-replica API with Redis and a gateway

- **Status:** Accepted
- **Date:** 2026-11-23
- **Deciders:** project maintainer
- **Supersedes:** ADR 0010 (only for deployments that run more than one API replica)
- **Related:** ADR 0004 (data contract), ADR 0010 (Phase 1 single-replica transport)

## Context

ADR 0010 accepted SSE plus polling for one API process. It also recorded the
limits that come with that choice: geometric fan-out and login counters live in
process memory, and two replicas can publish folded states out of order.

The next operational step is running two or more API replicas behind a single
entry point. Requirements:

- an event appended on one replica reaches SSE subscribers on another;
- the login throttle cannot be bypassed by alternating replicas;
- the public API and event contract do not change;
- the volunteer `basic` profile keeps working without Redis.

## Decision

1. **Redis is shared ephemeral state, not game storage.** Postgres remains the
   source of truth. Redis carries Pub/Sub fan-out messages and login counters
   only; it is not backed up and has no durability requirement.
2. **Keep SSE plus polling.** Do not migrate to WebSocket or Redis Streams in
   this step. A replica that misses a Pub/Sub message recovers through the
   existing polling fallback.
3. **Order by `state.lastSeq`.** `GameStreamService` deduplicates and rejects
   stale states using `lastSeq`, not an in-process revision counter. The web
   client also discards a state whose `lastSeq` is lower than the current one.
4. **Publish locally first, then through Redis.** The scoring replica stays
   instant; the other replicas receive the same envelope. A Redis publication
   failure is logged and does not fail an already-committed append.
5. **The login throttle is shared.** `RedisLoginRateLimiter` stores hashed IP
   and identity keys with a 15-minute fixed window, five failures per address
   plus email and thirty per address. A configured but unreachable Redis fails
   closed with `503`; `REQUIRE_REDIS=1` without `REDIS_URL` refuses startup.
6. **One Caddy gateway is the only published port.** It proxies `/api/*` to the
   API replicas with response buffering disabled for SSE, and everything else
   to the web app. API, web and Redis have no published host ports.
7. **Migrations take a Postgres advisory lock.** Concurrent replica startup
   serializes the migration ledger and DDL.
8. **Deployments are separate.** The existing `basic` compose file and
   `deploy/up.sh` stay Redis-free and single-replica. The scaled stack is
   `deploy/docker-compose.scaled.yml` with `--scale api=2`.
9. **Draining closes SSE.** On shutdown the API completes existing streams so
   clients reconnect to another replica and resume from Postgres instead of
   holding the old process open.

## Consequences

- Two API replicas can serve a game without split live state or a bypassable
  login limit.
- Operators need Redis and Caddy, and must set `REDIS_URL`, `REQUIRE_REDIS=1`,
  `TRUST_PROXY=1` and `PUBLIC_ORIGIN`.
- Redis loss degrades live updates to polling. With `REQUIRE_REDIS=1`, login is
  unavailable until Redis returns; game reads and appends already committed in
  Postgres still work.
- Redis Pub/Sub has no replay. A replica that starts after a message misses it
  and must read the snapshot.
- The `basic` profile keeps ADR 0010's one-replica behavior. This ADR supersedes
  ADR 0010 only when the scaled deployment is used.
- Durable jobs, Redis Streams consumer groups, WebSocket and multi-region
  operation remain out of scope.

## Revisit triggers

- the login limit must survive a Redis restart without fail-closed behavior;
- clients need replay of messages missed while disconnected;
- more than two API replicas or sustained multi-replica load;
- background jobs need a durable queue (BullMQ);
- a WebSocket transport is required by the client.
