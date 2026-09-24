# TRIBUTE — A-Tier Ops Hardening Report

Date: 2026-09-24. Scope: close the "S-tier needs full stack + real use proof
+ ops discipline" gap identified after finding production was running a full
day behind the repo's security/feature fixes.

## What was broken (starting point)

`api-gw.tributex402.com` (live `tribute-proxy.service`, code at
`/root/tribute/proxy/`) was a **separate, unsynced copy** of
`github.com/tributelab/tribute`'s `gateway/` dir — no CI, no CD, no deploy
script, just manual `cp`. The prior session's P0 security fix (rate-limit
XFF-spoofing) and the x402-session-discovery fix sat in git, unpushed to
production, for about a day before anyone checked. That's the core gap this
pass closes.

## What was already in place (verified, not newly built)

- **Hourly state backup** (`tribute-backup.sh`, cron `0 * * * *`) — confirmed
  live: 9 fresh `.tar.gz` archives in `/root/backups/tribute/`, latest
  06:00 UTC, `chmod 600`, integrity-checked against `data/settlements.json`.
- **15-min health watchdog** (`tribute-watchdog.sh`, cron `*/15 * * * *`) —
  confirmed live: `.watchdog_state` = `0` (healthy streak), 3-strike rule
  before alerting, Telegram delivery wired.

## What was built and verified this session

### 1. CI — GitHub Actions (`.github/workflows/ci.yml`)
Two jobs on every push/PR to `main`:
- `gateway`: `npm ci && npm test` (the 25-check smoke suite)
- `kit`: `npm ci && npm run build` (TypeScript compile)

**Verified live, not simulated**: pushed commit `e2b291e`, polled the GitHub
Actions API, confirmed run `35965369722` → `status: completed, conclusion:
success`. A broken build or test regression will now fail CI red instead of
silently merging.

### 2. RPC failover (`facilitator.js`, `server.js`)
Production logs showed recurring `JsonRpcProvider failed to detect network`
against the single configured public RPC. `TRIBUTE_RPC_UPSTREAM` now accepts
a comma-separated list; with >1 URL, `facilitator.js` builds an
`ethers.FallbackProvider` (quorum 1 — any one healthy responder answers,
no majority-vote stall). Single-URL config is unchanged (plain
`JsonRpcProvider`, zero added overhead). Also fixed `currentBlock()`, which
used a raw `.send('eth_blockNumber')` that `FallbackProvider` doesn't
expose — switched to the portable `getBlockNumber()`.

**Verified**:
- Single-URL: `currentBlock()` → real block number, unchanged behavior.
- Multi-URL (both alive): succeeds.
- **Failover test**: first URL pointed at a non-existent domain, second at
  the real RPC → `currentBlock()` still returned a real block number,
  proving the dead endpoint gets routed around automatically.
- Full smoke suite re-run after the change: still 25/25.
- **Deployed to production**: `.env` now carries both Robinhood RPC
  endpoints (`publicnode.com` + `rpc.mainnet.chain.robinhood.com`); prod
  health check confirms `rpc: { ok: true }` after restart.

### 3. Safe deploy script (`gateway/scripts/deploy-to-live.sh`)
This is the actual fix for the root cause. Before running any file copy it:
1. Runs the repo's own test suite — aborts on any failure.
2. `node --check`s every file about to ship — aborts on syntax error.
3. Backs up live `data/` + `.env` to a timestamped dir — aborts if the
   backup doesn't actually contain `data/`.
4. Copies only the *code* files (never `data/`, never `.env` — those are
   runtime state/secrets, not repo-sourced).
5. Restarts the systemd service, health-checks `/x402/health`.
6. **Auto-rolls back** to the pre-deploy backup and restarts again if the
   service fails to come up OR the health check doesn't report `ok`.

**Verified live, not dry-run**: executed for real against
`tribute-proxy.service`. Output: test suite 25/25 → syntax OK on 14 files →
backup 4.5M → files copied → service restarted → health check green.
Confirmed post-deploy: `diff` of every `.js` file between
`/root/tribute/proxy/` and `/root/tribute/repo/gateway/` → **zero
differences**. Production and the OSS repo are now byte-identical.

## Verification summary

| Check | Result |
|---|---|
| CI run on real push to `main` | ✅ completed / success (run 35965369722) |
| Gateway test suite | ✅ 25/25 (local, in deploy script, and in CI) |
| Kit build | ✅ clean |
| RPC failover — single URL unaffected | ✅ |
| RPC failover — dead 1st + alive 2nd URL | ✅ routes around dead endpoint |
| Deploy script — real run against prod | ✅ backup → deploy → health-check all passed |
| Production ≡ repo (file diff) | ✅ 0 differences across all gateway `.js` files |
| Hourly backup cron | ✅ confirmed live, latest run < 1h old |
| 15-min watchdog cron | ✅ confirmed live, healthy state |
| Production RPC failover active | ✅ `.env` has 2 endpoints, health check confirms `rpc.ok=true` |

## What A-tier does NOT yet cover (deferred to S-tier)

- **No CD** — `deploy-to-live.sh` must still be run by hand after a push;
  CI verifies the build is good but doesn't ship it automatically.
- **`marketplace.js` still flat-JSON** — lower risk than the balance/routes
  ledgers already migrated (no direct USDG balance), but still the last
  full-file-rewrite-per-mutation module in the gateway.
- **No written runbook** — deploy/rollback/incident steps exist as this
  report + script comments, not a dedicated ops doc newcomers can follow
  without archaeology.
