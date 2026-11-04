# ADR 0006 — Licensing and contribution model

- **Status:** Accepted
- **Date:** 2026-11-03
- **Deciders:** project maintainer
- **Supersedes:** none
- **Source:** §15.3 of `docs/vision.md`

## Context

The project targets adoption by volunteer leagues and reuse by third-party implementations of
the contract. A licence that discourages commercial hosting or a contributor agreement that
concentrates rights would work against that goal.

## Decision

Apache-2.0, DCO sign-off, Conventional Commits, no CLA. Anyone may fork and host it
commercially; the licence is chosen for adoption, not for protection.

## Consequences

- Every repository ships the Apache-2.0 `LICENSE` text.
- Contributions carry a Developer Certificate of Origin sign-off instead of a signed CLA.
- Commit messages follow Conventional Commits so that release notes and semver can be derived.
- Commercial hosting and forks are permitted; the project relies on the contract, not on licence
  leverage, to keep implementations interoperable.
