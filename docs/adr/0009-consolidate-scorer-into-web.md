# ADR 0009 — Consolidate the scorekeeper into the web app

- **Status:** Accepted
- **Date:** 2026-11-21
- **Deciders:** project maintainer
- **Supersedes:** none
- **Source:** score merge and tier-1 deploy

## Context

`olb-scorer` started as a separate Next.js PWA on port `3002`, with its own
origin, build, deploy process and browser storage. It shared the contract with
`olb-web` but duplicated the fold, scoring rules, lineup, finish, recovery and
proposal code. The web app already served the portal, the admin console and the
first-run setup; the scorer needed the same API session and the same design
tokens.

Two origins also split offline state: a draft created on the scorer origin was
invisible to the web origin, and moving a device meant copying browser storage
by hand. The tier-1 workflow expects one operator to log in once, prepare a
game from the league catalog and score it offline.

## Decision

1. **The scorer lives in `olb-web` under `/score`.** The routes are normal App
   Router routes in the same deployment, with a route-scoped manifest and
   service worker for the offline shell.
2. **One source of truth for scorer logic.** `olb-web/src/lib/scorer/` is
   authoritative. The standalone repository is not developed further.
3. **The standalone process is retired.** `deploy/up.sh` starts the API and the
   web app only; `deploy/down.sh` still stops a stale `scorer.pid` from the old
   three-process deployment.
4. **The old repository is frozen, not deleted.** Tag `v0.1.0-legacy` keeps the
   final app and its `/export` route so drafts in the old browser origin can be
   written as an `olb.local-games/1` bundle and imported at `/score`.
5. **The contract remains the boundary.** Moving the UI does not change the
   event schema, the API or the compatibility rules in ADR 0004 and ADR 0008.

## Consequences

- One build, one deploy, one public origin and one login session for the portal,
  admin console and scorer.
- The web bundle grows and the web app now owns the offline/PWA surface; a web
  release can affect scoring.
- The old scorer repository still exists and can confuse contributors unless
  its README states that it is retired. It does.
- Migration from the old origin is manual and one-way: export, import, then stop
  the old process.
- If the scorer ever needs an independent release cadence or a different
  runtime, it would have to be extracted again; this ADR accepts that cost in
  exchange for removing the duplicate code and the second origin now.
