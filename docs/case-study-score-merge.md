# Case study: score merge and process deploy

**Status:** merged and verified locally · 2026-11-21
**Scope:** consolidate the standalone scorer into `olb-web`, deploy API and web
as plain processes, and retire the `:3002` scorer. No new product features.

This is a verification snapshot, not a production-readiness claim.

## 1. What changed

- The scorekeeper moved into `olb-web` under `/score`, including
  `/score/new`, `/score/[id]`, `/score/[id]/score` and `/score/[id]/verify`.
- The score routes ship their own manifest and service worker, so the shell and
  queued local games keep working offline.
- Local games move between devices through an `olb.local-games/1` bundle.
- The retired `olb-scorer` keeps an `/export` page at tag `v0.1.0-legacy` so a
  device with drafts in the old browser origin can move them to `/score`.
- `olb-api/deploy/up.sh` builds the contract, API and web, starts the API and
  waits for `/health`, then starts the web app and waits for `/`. There is no
  separate scorer process or port.
- Public URLs can be HTTPS origins behind Tailscale Serve; the processes keep
  listening on the local ports.

The decision is recorded in [ADR 0009](adr/0009-consolidate-scorer-into-web.md).

## 2. Merge

The work had accumulated on one feature branch per repository. In all three,
`main` was an ancestor of the branch, so each merge was a fast-forward with no
conflict resolution.

| Repository | Branch | Result |
| --- | --- | --- |
| `olb-contract` | `main` | pushed; tag `v0.2.0` |
| `olb-api` | `feat/score-merge-deploy` | fast-forward to `7d93905` |
| `olb-web` | `feat/score-merge` | fast-forward to `b1e1f5c` |
| `olb-scorer` | `feat/legacy-export` | fast-forward to `863eb28`; tag `v0.1.0-legacy` |

All seven workspace repositories had a clean working tree before the merge.
The contract was published first because the API and web CI check it out from
`main`; merging in the other order would build against the previous schema.

## 3. Verification

All commands were run on 2026-11-21 in the `~/work/olb` workspace, before the
fast-forward. The API suite was run after the contract build finished; a
parallel first attempt read the contract's `dist/` while it was being rebuilt.

| Surface | Command | Result |
| --- | --- | --- |
| Contract, TypeScript | `cd olb-contract && npm test` | 20 passed, 0 failed |
| Contract, Python | `.venv/bin/python -m pytest -q` | 21 passed |
| Contract, generated output | `npm run check` | clean after build |
| API | `cd olb-api && npm run typecheck && npm test && npm run build` | 235 passed, 10 skipped; typecheck and build clean |
| Web | `cd olb-web && npm run typecheck && npm test && npm run build` | 404 passed; typecheck and build clean |
| Scorer legacy | `cd olb-scorer && npm run typecheck && npm test && npm run build` | 211 passed; typecheck and build clean |
| Database backup | `BACKUP_CONTAINER=olb-postgres ./deploy/backup.sh` | dump published, 60 KB |
| GitHub Actions | workflow runs on `3b22993`, `7d93905`, `b1e1f5c` | all successful |

The 10 skipped API tests are the Postgres integration and migration suites that
require an explicit integration database. The documentation commits that follow
the merge trigger separate runs; they do not change code.

## 4. Cutover

1. Take a backup of the live database.
2. Run the retired scorer from tag `v0.1.0-legacy`, open `/export`, and save the
   bundle for each device that still holds drafts.
3. Start the new stack with `SEED=0 ./deploy/up.sh`.
4. Import the bundles at `/score` and check the recovered games.
5. `./deploy/down.sh` stops any stale `scorer.pid` left by the old three-process
   deployment.

The cutover was not executed as part of this merge; it belongs to a deploy
window. There is no pilot league and no known device with production drafts.

## 5. Known limits

- **Next.js security advisory.** The pinned Next.js 16.4.3 build prints a
  `SecurityFatalError` recommending an upgrade and only continues after a retry.
  The advisory is unresolved in this snapshot; both web and legacy scorer are
  affected.
- **No CI for the retired scorer.** API, web and contract run in GitHub Actions;
  `olb-scorer` has no workflow, so its green suite is local evidence only.
- **One API replica.** SSE remains in-process; the deploy package runs a single
  API process.
- **No WebSocket transport.** The viewer still polls, and SSE is a one-way
  replacement for the contract's snapshot + delta.
- **No real pitch tracks.** The viewer draws synthetic trajectories until
  `olb-vision` ships.
- **Legacy drafts are manual.** A device that cannot run tag `v0.1.0-legacy`
  must re-enter its drafts or finish them before the cutover.
- **No release and no pilot league.** Tags `v0.2.0` (contract) and
  `v0.1.0-legacy` (scorer) are identifiers, not product releases.

## 6. What the slice proves

The project can carry one offline scorekeeper across a deployment cutover
without a second server process: the same Next.js app serves the public portal,
the admin console and the scorer, and a local game can be exported, moved and
imported. The merge also proves that the contract-first order works: publishing
the schema before the consumers keeps both builds on the same source.

It does not prove production security, high availability or that a real league
can run a season on the stack. Those remain open.
