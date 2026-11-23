# Multi-replica scaling plan

- **Status:** Accepted and implemented; see ADR 0011 and `deploy/docker-compose.scaled.yml`
- **Date:** 2026-11-23
- **Scope:** `olb-api`, `olb-web`, `olb-whitepaper` deployment docs
- **Related:** ADR 0010 (SSE plus polling, one replica), ADR 0004 (contract)

## Goal

Run two or more `olb-api` processes behind one gateway without:

- losing SSE updates when the scorer writes to a different replica;
- allowing the login throttle to be bypassed by alternating replicas;
- exposing Redis or the API replicas directly;
- changing the event contract or the public API shape.

The single-replica `basic` profile must keep working with no Redis.

## Why this is needed

`olb-api` currently keeps two pieces of state in process memory:

1. `GameStreamService` stores a `ReplaySubject` per game and publishes folded
   states directly to local SSE subscribers.
2. `LoginRateLimiter` stores failed-login buckets in `Map` objects.

With two replicas:

- an event appended on `api-1` is invisible to an SSE client connected to
  `api-2`; only the three-second polling fallback recovers it;
- five failures on each replica produce ten attempts instead of five;
- a restart clears every login bucket in that process.

There is a third, less obvious issue: `appendEvent` reads and folds the log
after the write, so two replicas can publish states out of order. The web client
currently applies any state it receives, including an older `lastSeq`. The
client must reject stale states before multi-replica traffic is enabled.

## Recommended architecture

Keep SSE plus polling, add Redis for shared ephemeral state, and put a gateway
in front of the API replicas.

```text
browser / phone
      |
      v
  gateway (Caddy)
      |
      +------ api-1 --------+
      |                     |
      +------ api-2 --------+---- Postgres (source of truth)
                            |
                          Redis
                    (Pub/Sub + counters)
```

- **Postgres** remains the source of truth for events, sessions and league data.
- **Redis** carries only fan-out messages and short-lived login counters. It is
  not durable game storage.
- **SSE** remains the live transport. WebSocket and Redis Streams are a later
  decision, not part of this plan.
- **The contract does not change.** The SSE payload remains the existing
  `GameStreamMessage` with a folded `GameState`.

### Why Redis and not Postgres LISTEN/NOTIFY

Postgres `LISTEN/NOTIFY` would avoid a new service, but its payload limit is
8 KB and every replica would need to re-read and re-fold the event log after
each notification. The whitepaper already names Redis for this role. Redis
Pub/Sub is the smallest change that gives cross-replica fan-out; Redis Streams
can replace Pub/Sub later if durable replay or consumer groups are needed.

Sticky sessions alone are rejected: they reduce the SSE split but do not fix
cross-replica event fan-out, login limits or restart behavior.

## Work breakdown

### P0 — Decision and ADR

1. Write ADR 0011, `supersedes ADR 0010`, accepting Redis Pub/Sub plus shared
   login counters for multi-replica deployments.
2. Keep the one-replica `basic` profile and ADR 0010 behavior as the default
   volunteer deployment.
3. Define the scaled deployment contract: gateway, Redis, two API replicas,
   no published API or Redis ports.
4. Document that Redis loss degrades SSE to polling and makes login fail closed
   when `REQUIRE_REDIS=1`.

**Acceptance:** ADR approved; no code changes before approval.

### P1 — Shared game stream

1. Extract a `GameStreamBus` interface with two implementations:
   - `LocalGameStreamBus` for tests and the Redis-free profile;
   - `RedisGameStreamBus` using one publisher/subscriber connection pair.
2. Refactor `GameStreamService` to publish through the bus and keep the local
   `ReplaySubject` only as the per-process subscriber buffer.
3. On publish:
   - deliver locally first so the scoring replica remains instant;
   - publish `{origin, gameId, state}` to `olb:game:<id>` through Redis;
   - skip the Redis echo when `origin` equals this process id.
4. On startup, subscribe to `olb:game:*` with one pattern subscriber and route
   messages to the local game subjects.
5. Use `state.lastSeq` as the ordering key. Remove the in-process `revision`
   counter from the public behavior; duplicate states are harmless. A state
   with a lower `lastSeq` must never replace a newer one.
6. In `streamGame`, read the snapshot, then watch the local subject and filter
   updates whose `lastSeq` is less than or equal to the snapshot. The replay
   buffer still closes the read/subscribe gap.
7. In `olb-web`, ignore an SSE or polling state when
   `next.lastSeq < lastSeq.current`. Keep connection status handling separate.
8. If Redis publish or subscribe fails:
   - never fail the already-committed event append;
   - log the failure with the instance id;
   - keep local SSE working;
   - let other replicas recover through polling.
9. Add an instance id (`HOSTNAME` or a random UUID) to logs and messages.

**Files:** `olb-api/src/games/game-stream.service.ts`,
`olb-api/src/games/games.service.ts`, `olb-api/src/games/games.controller.ts`,
new `olb-api/src/realtime/*`, `olb-web/src/components/live-game.tsx`, tests.

**Acceptance:**

- two Nest application instances sharing one Redis: an event published by A
  reaches an SSE subscriber on B;
- a delayed state with a lower `lastSeq` does not overwrite a newer state;
- one instance without Redis behaves exactly as today;
- existing SSE and polling tests still pass.

### P2 — Shared login throttle

1. Make `LoginRateLimiter` an interface with async methods.
2. Keep `InMemoryLoginRateLimiter` for tests and the Redis-free profile.
3. Add `RedisLoginRateLimiter`:
   - keys contain SHA-256 hashes of IP and normalized email, never raw values;
   - 15-minute fixed window, five failures per IP+email, thirty per IP;
   - `INCR` plus `PEXPIRE` in one Lua script per key;
   - successful login deletes both keys;
   - a configured but unreachable Redis returns `503 login temporarily
     unavailable` instead of silently bypassing the limit.
4. Add `TRUST_PROXY` handling so `@Ip()` reads the client address from the
   gateway, not the gateway address. Only trust the known proxy hop.
5. Keep the existing 429 response shape and `retryAfterSeconds`.
6. Add startup validation: `REQUIRE_REDIS=1` without `REDIS_URL` refuses to
   start.

**Files:** `olb-api/src/auth/login-rate-limit.ts`,
`olb-api/src/auth/auth.controller.ts`, `olb-api/src/main.ts`,
`olb-api/src/app.module.ts`, `olb-api/test/login-rate-limit.test.ts`.

**Acceptance:**

- six failures alternating between two replicas produce a 429 on the sixth;
- a successful login clears both counters;
- two different emails from one IP do not share the per-identity bucket;
- an IP over thirty failures is blocked across replicas;
- Redis keys expire and contain no email or IP text;
- with `REQUIRE_REDIS=1` and no Redis, startup fails clearly.

### P3 — Deployment and lifecycle

1. Add a scaled compose file rather than changing the `basic` stack:
   `olb-api/deploy/docker-compose.scaled.yml` (or an equivalent documented
   override). It adds:
   - `redis:7-alpine`, internal only, healthcheck, no host port;
   - `gateway` (Caddy) publishing one API port and proxying `api:3001`;
   - two API replicas with no published host port;
   - web pointed at the gateway URL.
2. Configure the gateway for SSE:
   - disable response buffering (`flush_interval -1` in Caddy);
   - long read timeout;
   - forward `X-Forwarded-For` and `X-Forwarded-Proto`.
3. Keep the direct `up.sh` path as the one-replica option. Add a separate
   `up-scaled.sh` only if the compose path is not enough for the host.
4. Make migrations safe for concurrent startup. Add a Postgres advisory lock
   around `runMigrations`, or run migrations once before starting replicas.
   Concurrent `CREATE TABLE`/`INSERT schema_migrations` must not race.
5. Add a realtime readiness signal to `/health`: storage, realtime mode
   (`memory` or `redis`), and Redis ping. Do not mark the API unready merely
   because Redis is down if games can still be read and polling works.
6. Add graceful drain on `SIGTERM`: stop accepting new SSE connections, close
   existing streams after a short grace period, and let clients reconnect to
   another replica.
7. Document backup, restore and rollback for the scaled stack. Redis has no
   backup requirement because it holds no durable game data.

**Environment for the scaled stack:**

```text
REDIS_URL=redis://redis:6379
REQUIRE_REDIS=1
TRUST_PROXY=1
API_REPLICAS=2
```

**Acceptance:**

- `docker compose ... up --scale api=2` starts two healthy API replicas;
- the gateway serves web, API and SSE;
- killing one API replica does not stop the game; clients reconnect and the
  state catches up from Postgres;
- Redis restart does not lose game data; SSE degrades to polling and login is
  unavailable while `REQUIRE_REDIS=1`;
- a fresh database starts with two replicas without a migration race.

### P4 — Verification and release

1. Unit tests for local and Redis stream buses.
2. Redis integration tests, skipped when `TEST_REDIS_URL` is absent, plus a CI
   Redis service.
3. HTTP test with two app instances sharing Redis:
   - POST an event to replica A;
   - hold an SSE connection on replica B;
   - assert the state arrives with the correct `lastSeq`.
4. Login HTTP test alternating replicas.
5. Scaled smoke on disposable ports and database:
   - `/health` on both replicas;
   - `/`, `/score`, `/stats`, game page;
   - live second screen across replicas;
   - six failed logins -> 429;
   - backup/restore of Postgres only.
6. Update `olb-whitepaper/README.md`, `docs/vision.md`, ADR index, deployment
   docs and `STATUS.md`.
7. Tag a new release candidate only after the scaled smoke passes.

## Test matrix

| Case | Expected |
| --- | --- |
| One API, no Redis | Existing behavior; no new service required |
| Two APIs, Redis up | Event on A reaches SSE on B; no duplicate local echo |
| Two APIs, Redis down | Local SSE works; remote clients use polling; login 503 if required |
| Delayed stale state | Client keeps the higher `lastSeq` |
| Six failed logins across A/B | 429 with `retryAfterSeconds` |
| Successful login after failures | Both counters cleared |
| Replica killed during game | SSE reconnects; snapshot matches Postgres |
| Two replicas start on empty DB | One migration applies; no duplicate or failed migration |
| Backup/restore | Postgres dump restores; Redis empty is acceptable |

## Rollout and rollback

1. Implement behind `REDIS_URL`; merge with the one-replica tests green.
2. Deploy the scaled profile against a disposable database and run the smoke.
3. Run the fictional pilot on the scaled stack only after the smoke passes.
4. Roll back by scaling to one API replica and removing `REDIS_URL` and
   `REQUIRE_REDIS`. No schema or contract rollback is needed because Redis
   holds no durable data.
5. If the client stale-state guard causes a problem, it can be reverted
   independently; it is compatible with the one-replica path.

## Risks and mitigations

- **Redis becomes a new operational dependency.** Keep it out of the basic
  profile; document memory and restart behavior; do not publish its port.
- **Pub/Sub drops messages during a disconnect.** Polling remains the recovery
  path; do not claim durable delivery.
- **Distributed fixed-window limits are approximate.** Keep the same limits;
  move to a token bucket or sliding window only if abuse data requires it.
- **Gateway buffers SSE.** Explicitly disable buffering and test through the
  gateway, not only against the API port.
- **Spoofed forwarded IPs.** Trust only the gateway hop and document the proxy
  boundary.
- **SSE connections consume memory and file descriptors.** Set a connection
  budget and a maximum stream duration before a large public launch.
- **Migration races.** Advisory lock or one-shot migration is mandatory before
  two replicas start.

## Open decisions before implementation

1. **Redis vs Postgres LISTEN/NOTIFY.** Recommendation: Redis, matching the
   whitepaper and keeping notifications small.
2. **Gateway.** Recommendation: Caddy for simple SSE configuration; nginx is
   acceptable if the deployment already uses it.
3. **Profile name.** Recommendation: a separate `scaled` deployment file, not a
   change to `basic`.
4. **Redis failure policy.** Recommendation: fail closed for login, degrade
   realtime to polling, keep reads and writes available.
5. **When to require Redis.** Recommendation: only when
   `REQUIRE_REDIS=1`; the default one-replica profile stays Redis-free.

## Definition of done

- [x] ADR 0011 accepted and linked from the ADR index.
- [x] Shared stream bus and Redis login limiter implemented.
- [x] Client rejects stale states.
- [x] Scaled compose stack runs two API replicas behind the gateway.
- [x] Migration startup is safe with concurrent replicas.
- [x] Cross-replica SSE and login-limit tests pass.
- [x] One-replica stack still passes all existing tests without Redis.
- [x] Scaled smoke verified; rollback path documented (the scaled database
  volume is separate from the basic package).
- [x] `STATUS.md` and release notes reflect the result.

## Explicit non-goals

- Replacing SSE with WebSocket.
- Durable SSE replay or Redis Streams consumer groups.
- BullMQ or background workers.
- Horizontal Postgres scaling.
- Multi-region deployment.
- Changing the event or realtime contract.
