# TRIBUTE — S-Tier Report

Date: 2026-09-24. Continues from `OPS-A-TIER.md` (same session). Scope:
close the remaining gaps between A-tier ("good code + basic ops") and
S-tier ("full stack + real use proof", including operational discipline).

## What S-tier added on top of A-tier

### 1. Continuous Deployment (`tribute-gateway-cd` cron, `*/10 * * * *`)
A-tier shipped a *safe manual* deploy script; the gap was still "someone has
to remember to run it." `~/.hermes/scripts/tribute-cd-watch.sh` closes that:

- Polls the GitHub API for the latest commit on `main` (no public webhook
  endpoint on this host, so polling is the honest choice over pretending to
  have a webhook).
- Compares against `/root/backups/tribute/.deployed_sha` (written by
  `deploy-to-live.sh` on every successful deploy) — exits silently if
  nothing's new.
- **Gates on CI status** via the check-runs API before touching anything:
  pending → wait for next tick; failure → alert, refuse to deploy; success →
  proceed.
- On a green new commit: `git fetch && git reset --hard origin/main`, then
  runs the existing `deploy-to-live.sh` (test suite → syntax check → backup
  → deploy → health-check → auto-rollback-on-failure, unchanged from A-tier).
- `no_agent: true` cron — zero LLM cost per tick, script stdout is the
  Telegram message, silent when there's nothing to report.

**Verified live, twice, not simulated:**
- First real run: pushed commit `12ca915` (docs-only), watcher correctly
  found no `.deployed_sha` yet, deployed it, recorded the SHA.
- Second run (idempotency check): ran again immediately with no new
  commit — exited silently, zero side effects, confirmed via `diff` that
  nothing changed.
- Third real run (the actual S-tier commit): pushed `13fdb9a`
  (marketplace SQLite migration + runbook), waited for CI to go green,
  fired the cron job for real via the scheduler (not a manual bash
  invocation) — it detected the new commit, confirmed CI success, pulled,
  deployed, and production now matches `13fdb9a` exactly (`diff -rq`
  across every gateway `.js` file: zero differences).

### 2. `marketplace.js` migrated to SQLite (last flat-file module gone)
This was the one remaining "under load, this gets slow and eventually
corrupts" module — full `marketplace.json` rewrite on every listing
hit/settle/fail/refund. Added `marketplace_listings`, `marketplace_subs`,
`marketplace_nonces` tables to `db.js`; rewrote `marketplace.js` on indexed
prepared statements, same pattern as every other module. Public API
(function signatures, return shapes) is byte-for-byte unchanged — `server.js`
needed zero edits.

**Verified with a real functional test**, not just "it compiles":
- Spun up a local HTTP server, created a real listing with an actual
  EIP-712-signed `ListingIntent` (the exact flow a seller would use),
  including the mandatory upstream-probe (`fetchJson` against the local
  server).
- Confirmed `get()`, `catalog()`, `recordHit/Settle/Fail/Refund`,
  `earningsFor()` all produce correct aggregates.
- Confirmed the nonce replay guard rejects a reused signature/nonce.
- Confirmed `revoke()` (also EIP-712-signed) delists correctly and `get()`
  returns `null` afterward.
- Confirmed subscription grant/consume/refund math (`grantSub`,
  `consumeSub`, `refundSub`, `subsView`) is correct.
- **Confirmed persistence**: closed the process, opened a fresh one against
  the same SQLite file — catalog and subscription state both survived
  intact.
- Full gateway smoke suite re-run after the change: still 25/25.
- Wrote and tested `migrate-marketplace-json-to-sqlite.js` — idempotent,
  re-runnable, no-ops cleanly when no flat file exists (verified: ran it,
  got "nothing to migrate", matching reality — no `marketplace.json` exists
  in prod or the repo, so there was no live data at risk here).
- **Deployed to production via the real CD pipeline** (see above) — the
  live marketplace endpoint (`/x402/marketplace`) responds correctly
  post-deploy (`{ok:true, feeBps:500, apis:[]}` — correct empty state,
  no listings created yet).

### 3. Ops runbook (`RUNBOOK.md`)
A-tier's operational knowledge existed only as script comments and a
session report. `RUNBOOK.md` is the durable reference: repo-vs-live
distinction (and the explicit "never hand-edit the live dir" warning that
would have prevented the original desync), the full deploy flow (manual and
automatic), credentials/identity recipe (including the
`tributelab`-vs-`ignitepadnft` account trap), all three cron jobs
tabulated, deploy/CD script internals, manual rollback steps, health-check
semantics, RPC failover config, and an honest "known deferred items" list
(no staging env, no cooldown on repeated bad-commit CD attempts, etc.) so
the next gap doesn't get discovered the same way this one was.

## Full-stack final verification (this session, end to end)

| Layer | Check | Result |
|---|---|---|
| Tests | `npm test` (gateway, 25 checks) | ✅ pass |
| Build | `npm run build` (kit, TypeScript) | ✅ clean |
| CI | GitHub Actions on latest commit `13fdb9a` | ✅ both jobs success |
| CD | Cron-fired (not manual) deploy of `13fdb9a` | ✅ deployed, SHA recorded |
| Production sync | `diff` every gateway `.js`, repo vs `/root/tribute/proxy/` | ✅ zero differences |
| Production health | `/x402/health` all 7 sub-checks | ✅ all `ok: true` |
| RPC failover | dead 1st URL + alive 2nd URL | ✅ routes around dead endpoint |
| RPC failover in prod | `.env` has 2 endpoints, health confirms | ✅ |
| marketplace.js | full functional test (sign, list, settle, refund, revoke, sub, restart-persist) | ✅ all correct |
| marketplace.js in prod | live endpoint responds correctly post-CD-deploy | ✅ |
| Backup cron | fresh archive < 1h old, integrity-checked | ✅ |
| Watchdog cron | healthy state (0 consecutive fails) | ✅ |
| CD cron | scheduled, fired for real, idempotent on repeat | ✅ |

## What's still explicitly NOT done (honest gaps, not hidden)

- No staging environment — CI is the only gate before a CD deploy hits
  production directly. Acceptable for a single-operator OSS project at this
  scale; would be the next thing to add before a team touches this repo.
- No cooldown/backoff if a commit somehow passes CI but breaks health —
  the CD watcher would retry every 10 minutes. Each retry is itself safe
  (deploy script tests + rolls back), just noisy. Not fixed this session —
  flagged in `RUNBOOK.md`'s deferred-items list rather than silently left
  out of the report.
- Rate-limit buckets, sessions, and reputation are all SQLite now, but
  none of them replicate — this is still a single-node deployment. Fine for
  current scale; would need real infra work (not a code fix) to change.

## Tier assessment

**S-tier**, with the above three items named as the honest edge of scope —
not because they're hidden risks, but because "full stack + real use proof"
should include naming what you didn't do, not just what you did.

Everything claimed above was independently verified in this session against
live systems (real GitHub API calls, real cron firings, real production
`curl`s, real signed EIP-712 intents against a real local test server) —
not asserted from reading code.
