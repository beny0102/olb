# ADR 0008 — Additive event schema 0.2 and the scoring fold

- **Status:** Accepted
- **Date:** 2026-11-05
- **Deciders:** project maintainer
- **Supersedes:** none
- **Source:** D2a scoring completion

## Context

The `0.1.0` contract covers the basic play loop: pitches, balls in play,
runners, innings, lineups and substitutions. A scored league game also needs
hit-by-pitch and intentional-ball calls, batter interference, courtesy and
re-entry substitutions, suspensions and resumptions, automatic extra-inning
runners, and a projection that exposes the current lineup, batter, pitcher,
pitch counts and the rules profile pinned at first pitch.

Existing games are already stored under `olb.event/1` with none of those
fields. They must keep folding to the same state. The scorer PWA and the web
viewer also consume the published types and cannot be forced to upgrade in the
same change.

## Decision

1. **The wire tag stays `olb.event/1`.** The media type's major version changes
   only when a consumer must change to keep reading the log. Everything in
   schema `0.2` is additive, so the tag does not move. The package version
   becomes `0.2.0`; the Python distribution follows it.
2. **Additive compatibility is the rule within a major version.** A release may
   add optional fields and new event branches. It may not remove a field,
   change a field's type, or make a previously optional field required.
   Consumers must ignore unknown fields and unknown event types. The
   `fixtures/forward/` case is the executable proof of that rule.
3. **New event branches use the standard envelope.** `interference`,
   `ejection`, `game.suspend` and `game.resume` carry the same `contract`,
   `id`, `gameId`, `seq`, timestamps, source, actor and optional revision
   fields as every other event. A producer that does not know them can still
   store and skip them.
4. **New payload fields are optional.** `pitch.call` gains `H` (hit by pitch)
   and `I` (intentional ball). `substitution.reason` gains `courtesy` and
   `reentry`, with an optional `reentry` marker. `lineup.set` gains an optional
   `dh`. `runner.from` gains `bench` and `runner.kind` gains `placed`; a placed
   runner must come from `bench`, land on a base and not be out. `game.start`
   gains an optional `rules` profile.
5. **The projected state gains optional fields.** `gameState` adds `lineups`,
   `batter`, `pitcher`, `pitchCounts` and `rules`. They are optional in the
   schema and default to `null` or empty in the fold, so a snapshot from an old
   log remains valid. `lineups` is `{ home, away }`, each side carrying its
   batting order, current pitcher and DH flag.
6. **The fold contract is deterministic and additive.**
   - `pitch` increments `pitchCounts[pitcher]` on every call. `I` behaves as a
     ball. `H` is a hit by pitch: forced runners advance, a run scores when the
     bases are loaded, the count resets and the plate appearance ends.
   - `interference` awards the batter first base, advances only forced
     runners, resets the count and ends the plate appearance.
   - `lineup.set` stores the side's order, pitcher and DH, then points
     `batter` at the batting side's first slot and `pitcher` at the fielding
     side's pitcher.
   - `substitution` replaces the outgoing player in the same batting slot,
     updates the side's pitcher when the new position is `1` (pitcher), and
     updates `batter` when the outgoing player was at bat. `courtesy` and
     `reentry` use the same mechanism; no state is invented when the side has
     no batting order.
   - `runner` with `kind: "placed"` puts the runner on the target base without a
     pitch or plate appearance.
   - `game.suspend` sets `suspended`; `game.resume` sets `live`. `ejection` is
     audit only and changes no projected state.
   - Revision and void semantics are unchanged. Old logs without the new
     fields fold to the same core state; only the new optional fields are
     populated when the log carries them.

## Consequences

- Old `olb.event/1` logs replay unchanged, and an old consumer that ignores the
  new event types continues to work.
- The API accepts and validates the new events through the same
  `POST /games/:id/events` path; no new endpoint is needed.
- The projection becomes a fuller description of the game, which the scorer
  and viewer can adopt without another contract break.
- Emitting the new events from the scorer UI, per-player box scores, realtime
  transport and admin surfaces remain out of scope; they are D2b/D3 work.
- A future breaking change still requires a new major media type, a migration
  note and a deprecation window, as recorded in the whitepaper's versioning
  rules.
