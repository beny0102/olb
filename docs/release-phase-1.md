# Phase 1 release candidate and pilot checklist

**Status:** release candidate for a supervised pilot. Not a general-availability
release and not a claim of production hardening.

The Phase 1 exit criterion is one seeded nine-inning game scored end to end
from a phone, synced from an offline browser, and followed on a second device. The scorer is
served by `olb-web` at `/score`; `olb-scorer` is retired at tag `v0.1.0-legacy`
for legacy draft export only.

## What ships

| Component | Artifact |
| --- | --- |
| Contract | `olb-contract` at `v0.2.0` |
| API | `olb-api` Dockerfile or `deploy/up.sh` on port 3001 |
| Web, admin, scorer | `olb-web` standalone build on port 3000 |
| Database | Postgres 16 with migrations through `003_player_privacy` |
| Deployment docs | `olb-api/README.md`, `olb-api/deploy/` |

Redis, BullMQ, WebSocket and a separate scorer process are not part of this
release. Realtime is SSE plus polling under
[ADR 0010](adr/0010-realtime-transport-phase-1.md).

## Before the pilot

1. Take a database backup with
   `BACKUP_CONTAINER=olb-postgres ./deploy/backup.sh` and restore it into a
   throwaway database with `./deploy/restore.sh <dump> --db olb_restore_test`.
   Drop the drill database afterwards.
2. Replace every seeded demo account and password. Seeded credentials are for
   local evaluation only.
3. Set `SETUP_TOKEN`, run `/setup` once, then remove or rotate the token file.
4. Put the stack behind HTTPS (Tailscale Serve or a reverse proxy) and restrict
   `CORS_ORIGIN` to the public web origin.
5. Confirm the public roster shows jersey numbers or positions, not names, for
   every player without a guardian consent.
6. Record the pilot's guardian consents in the admin console. Granting or
   revoking a name is an audit event in `player_consents`, not a policy note.
7. Confirm at least two phones can score offline and sync the same game, and
   that a second device follows the game in the viewer.

## Deploy

```bash
cd olb-api
SEED=0 PUBLIC_HOST=<host> ./deploy/up.sh
curl -fsS http://127.0.0.1:3001/health
```

Then open `/`, `/stats`, `/score`, `/setup` and one game page. Log in, score one
pitch, verify the viewer receives it, finish or suspend the game, and download a
local-game bundle to prove the offline path.

Stop with `./deploy/down.sh`. The script refuses to start when either port is
already listening; it never kills an unrelated process.

## Rollback

1. `./deploy/down.sh`.
2. Check out the previous release tag in `olb-api` and `olb-web`.
3. Restore the pre-deploy dump into the live database only with the explicit
   `--force` flag, after taking a fresh backup of the current state.
4. Bring the stack up with `SEED=0` and repeat the smoke checks.

The event log is append-only. Rolling back code does not rewrite events; rolling
back the database does, so it is a last resort and needs the league's consent.

## Known limits

- One API replica. SSE and the login throttle are in-process.
- No video, no real pitch tracks, no full-field animated plays.
- Per-player box score and self-service privacy controls are not built.
- Spanish copy is hardcoded; there is no second locale.
- Legacy drafts from a device that cannot run `v0.1.0-legacy` must be re-entered
  or finished before cutover.

## Exit evidence

The release candidate is accepted when the commands above pass on the pilot
host, the backup drill restores the expected row counts, the privacy smoke shows
`#jersey` before consent and the full name only while consent is active, and the
pilot league signs off on the rollback procedure.
