# ADR 0007 — Client event ids and revision-aware fold

- **Status:** Accepted
- **Date:** 2026-11-04
- **Deciders:** project maintainer
- **Supersedes:** none
- **Source:** scorer offline capture requirement (C0)

## Context

The scorer PWA must keep score offline and synchronise when connectivity
returns. A retry after a timeout is normal, so the API cannot append a second
copy of the same play just because the first response was lost. The client is
also the only place that knows when a play actually happened, which matters
when a batch is uploaded later.

The event contract (`olb.event/1`) already carries `id`, `occurredAt` and
`revisionOf`. The API, however, generated every `id` itself, so a retry with
the same payload produced a new event. Separately, the projection applied a
correction and the event it replaced, so an undo could not be represented as
an append.

## Decision

1. The client may supply the event `id` (a UUIDv7) and `occurredAt`.
   `seq`, `recordedAt` and `contract` remain server-owned. An append whose id
   already exists in the same game returns the stored event without writing;
   an id owned by another game is a `409`. A Postgres unique index on
   `body->>'id'` arbitrates concurrent retries, and the loser re-reads and
   returns the winner's event.
2. Corrections are append-only revisions. An event with `revisionOf` set
   supersedes the event with that id. An `amendment` with
   `payload.replacement: null` voids the event at `payload.targetSeq`.
3. The fold ignores superseded and voided events. It still counts every log
   entry for `lastSeq`, so a client resuming from `fromSeq` never replays a
   skipped event. A revision chain `A <- B <- C` applies only `C`.

The contract does not change. Producers that omit `id` or `occurredAt` keep
the previous behaviour.

## Consequences

- A retried POST is idempotent and returns the original event, even if the
  retry body differs. The server's stored value wins over the retry.
- Undo and correction are additive, so the log stays the single source of
  truth and every consumer derives the same state.
- A unique index is required on `events ((body->>'id'))`. Existing databases
  with duplicate event ids must be cleaned before the index can be created.
- Conflict semantics are explicit: reusing an id across games is rejected,
  and an invalid id or timestamp still fails contract validation with `400`.
- Scorer-side capture and upload are out of scope here (C1); this ADR covers
  the API contract and fold behaviour they depend on.
