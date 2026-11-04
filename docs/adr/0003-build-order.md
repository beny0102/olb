# ADR 0003 — Build order: vertical slice before the full league ERP

- **Status:** Accepted
- **Date:** 2026-11-03
- **Deciders:** project maintainer
- **Supersedes:** none
- **Amends:** roadmap in `README.md` and §12 of `docs/vision.md`

## Context

Phase 0 of the roadmap asked for seven repositories, compose profiles and ADRs 0003–0007 before
Phase 1 work began. What was actually built is the compatibility surface: contract v0.1.0, the
API that folds its event log, the 3D viewer and a league portal. The reason is defensible — the
contract is the only compatibility boundary, and a viewer is the only part a reviewer can
evaluate without installing anything — but it leaves a demonstrable front end with no league
model behind it.

There are two honest ways out: backfill the missing scaffolds until the published order is true,
or record the order that was followed and keep the exit criteria. Pretending the phases happened
in sequence is the one option that costs credibility.

## Decision

1. **Phase 0 closes with what makes the product boot**, not with a checklist of empty
   repositories: contract v0.1.0, the four existing repositories, the three missing repositories
   scaffolded with scope documents, and a `basic` compose profile that starts Postgres, the API
   and the web app.
2. **Contract v1 is tagged at the end of the domain phase**, not now. Scheduled games, lineups
   and the box score are the last cheap opportunity to change the schema; freezing a v1 before
   them would force a v2 within weeks.
3. **Order from here:** (a) close Phase 0 as described above, (b) build the game domain —
   scheduled games, lineups, complete fold, box score and standings, (c) scorekeeper PWA,
   (d) public demo and deployment.
4. **Deployment comes last** on purpose: the public demo must show a system that works, not a
   facade with empty bands.
5. The roadmap tables gain a footnote pointing at this ADR. No phase exit criterion is deleted;
   the deviation is dated and explained.

## Consequences

- “Seven repositories” is satisfied by four repositories with code and three with a scope
  document. The stubs carry no CI until they carry code; an empty pipeline proves nothing.
- The `demo` compose profile waits for the domain phase, because a seeded nine-inning game needs
  a fold that moves bases, outs and innings.
- A reviewer reading the roadmap finds the real order documented instead of an unexplained gap
  between the plan and the repository history.
- Contract v0.1.0 stays in use until the domain phase ends. Consumers must already ignore unknown
  fields and event types, which the forward-compatibility fixture enforces.
