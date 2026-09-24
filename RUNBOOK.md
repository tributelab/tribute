# TRIBUTE — Ops Runbook

Operational reference for `tribute-proxy.service` (production gateway,
serves `api-gw.tributex402.com`) and its OSS source
(`github.com/tributelab/tribute`). Read this before touching either.

## The two copies — never confuse them

| | Repo (source of truth) | Live (what actually serves traffic) |
|---|---|---|
| Path | `/root/tribute/repo/gateway/` | `/root/tribute/proxy/` |
| What it is | git-tracked, `origin` = `tributelab/tribute` on GitHub | systemd service `tribute-proxy`, **not** a git checkout |
| Data | none (`data/` is gitignored) | `data/gateway.db` + WAL — real vault entries, keys, balances, settlements |
| How code gets from left to right | `gateway/scripts/deploy-to-live.sh` (manual) or the `tribute-gateway-cd` cron (automatic, every 10 min) | — |

**Never edit `/root/tribute/proxy/*.js` directly.** Any hand-edit there gets
silently overwritten by the next CD tick or manual deploy, and never makes
it back into git. Always edit in the repo, commit, push — let CI + CD (or
the manual deploy script) carry it over.

## Normal change flow

1. Edit `gateway/*.js` in the repo.
2. `npm test` locally (25-check smoke suite) — must pass before committing.
3. Commit + push to `main` as `tributelab` (see credentials below).
4. GitHub Actions CI runs `npm test` (gateway) + `npm run build` (kit)
   automatically. Watch it: `gh run list -R tributelab/tribute` or the
   Actions tab.
5. **Auto path**: within 10 minutes, `tribute-gateway-cd` cron notices the
   new green commit and deploys it (see "CD watcher" below). You'll get a
   Telegram message either way — success or failure.
6. **Manual path** (if you don't want to wait): `bash
   /root/tribute/repo/gateway/scripts/deploy-to-live.sh` — same safety
   checks, runs synchronously.

## Credentials

- Repo push: `tributelab` identity, token at `/root/.tribute-gh-token`.
  Recipe (token must never linger in `.git/config`):
  ```bash
  git remote set-url origin "https://tributelab:$(cat /root/.tribute-gh-token)@github.com/tributelab/tribute.git"
  git -c user.name=tributelab -c user.email=tributelab@users.noreply.github.com push origin main
  git remote set-url origin "https://github.com/tributelab/tribute.git"   # strip token back out
  ```
  Do NOT use the machine's default `gh`/git credential (`ignitepadnft`) for
  this repo — wrong identity, and it's a different GitHub account entirely.
- Live `.env` secrets (`TRIBUTE_SETTLE_KEY`, `TRIBUTE_VAULT_KEY`) live ONLY
  at `/root/tribute/proxy/.env` — never in the repo, never in `.env.example`.

## Automated safety nets (what's already running)

| Job | Schedule | What it does | Silent when |
|---|---|---|---|
| `tribute-gateway-backup` | hourly | tars `data/` + `.env` to `/root/backups/tribute/`, keeps newest 14 | backup succeeds |
| `tribute-gateway-watchdog` | every 15 min | probes `/x402/health`, 3-strike rule | service healthy |
| `tribute-gateway-cd` | every 10 min | checks for a new green commit on `main`, deploys it | nothing new to deploy |

All three are `no_agent` cron jobs (script stdout only, zero LLM cost) and
deliver to Telegram `6426872166` only when there's something to report.

## Deploy script internals (`deploy-to-live.sh`)

Order of operations, each step aborts (and on later steps, rolls back) on
failure:
1. `npm test` in the repo gateway — refuses to ship a build that fails its
   own tests.
2. `node --check` every file about to be copied.
3. Backup live `data/` + `.env` to `/root/tribute-proxy-backup-<timestamp>/`
   — refuses to proceed if the backup doesn't actually contain `data/`.
4. Copy code files only (`server.js`, `facilitator.js`, ... — see the
   `FILES` array in the script). **Never touches `data/` or `.env`.**
5. `systemctl restart tribute-proxy`, wait 3s, check `systemctl is-active`.
6. `curl /x402/health`, check `.ok === true`.
7. If step 5 or 6 fails: copies every file from the just-made backup back
   over the live dir and restarts again — auto-rollback, no human needed.
8. On success: records the deployed git SHA to
   `/root/backups/tribute/.deployed_sha` (this is what the CD watcher reads
   to know whether there's anything new to deploy).

## CD watcher internals (`tribute-cd-watch.sh`, `~/.hermes/scripts/`)

1. Fetch latest commit SHA on `origin/main` via GitHub API.
2. Compare to `.deployed_sha` — if same, exit silently (nothing to do).
3. If different, check that commit's CI status via the check-runs API.
   - CI still running / unreadable → exit silently, retry next tick.
   - CI failed → alert, do NOT deploy.
   - CI green → `git fetch && git reset --hard origin/main`, then run
     `deploy-to-live.sh`.
4. Report success or failure (with the deploy script's own rollback having
   already run if it failed).

## Manual rollback (if you need to bypass the scripts entirely)

```bash
ls -1t /root/tribute-proxy-backup-* | head -1     # find the latest backup
cp /root/tribute-proxy-backup-<TIMESTAMP>/*.js /root/tribute/proxy/
systemctl restart tribute-proxy
curl -s http://127.0.0.1:8792/x402/health
```
Never restore `data/` from a backup unless you're deliberately reverting
live state (lost vault entries, corrupted DB) — restoring `data/` from an
old backup will roll back real settlements/balances, not just code.

## Health check semantics

`GET /x402/health` returns `{ ok, checks: { gateway, rpc, settlement, vault,
keys, wallets, apis } }` — each a real probe, not a canned `ok:true`. `rpc`
actually calls `eth_blockNumber` against the configured upstream(s)
(failover-aware since the RPC hardening pass — see below). A `200` with
`ok:false` means something specific is broken; check which `checks.*.ok` is
false before assuming the whole service is down.

## RPC failover

`TRIBUTE_RPC_UPSTREAM` in `/root/tribute/proxy/.env` is comma-separated
(currently `publicnode.com` + `rpc.mainnet.chain.robinhood.com`).
`facilitator.js` builds an `ethers.FallbackProvider` (quorum 1) when more
than one URL is configured — a single dead/slow endpoint no longer stalls
settlement. Only the settlement path gets full failover; the raw RPC
passthrough proxy (`POST /`, whitelisted methods) uses just the first
configured URL (see comment in `server.js`).

## Known deferred items (not yet done)

- No staging environment — CI runs the test suite, but there's no
  pre-production smoke test against real infra before a CD deploy.
- `no_agent` cron jobs don't distinguish "GitHub API rate-limited" from
  "no new commit" — both exit silently. Low risk (rate limits are generous
  for authenticated requests at this polling interval) but worth knowing.
- Deploy script has no maximum-attempts / cooldown — if a bad commit
  somehow passes CI, the CD watcher will keep retrying it every 10 minutes
  until it's fixed or reverted (each attempt properly rolls back, so this
  is safe but noisy).
