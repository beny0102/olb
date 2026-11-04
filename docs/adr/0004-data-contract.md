# ADR 0004 — Canonical data contract and versioning

- **Status:** Accepted
- **Date:** 2026-11-03
- **Deciders:** project maintainer
- **Supersedes:** none
- **Source:** §15.2 of `docs/vision.md`

## Context

The contract is the only compatibility surface between capture, core and the fan experience.
Producers and consumers are built and released independently, so the rules for changing the
schema have to be explicit before the first external implementation exists.

## Decision

- The contract is versioned with **semver**, and the major version is part of the media type
  (`olb.event/1`).
- Within a major: additive changes only. New optional fields, new event types that older
  consumers may ignore.
- Deprecation requires at least two minor releases of notice and a migration note.
- Removals and semantic changes require a new major and a written migration path.
- Consumers must ignore unknown fields and unknown event types. This is tested in CI with a
  "future payload" fixture.

## Consequences

- A consumer that follows the ignore rule survives additive releases without coordination.
- Breaking a major version is a deliberate, documented migration rather than an accidental
  side effect of a schema edit.
- The compatibility rules are enforced by the conformance suite in `olb-contract`, not by
  convention alone.
