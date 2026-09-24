# TRIBUTE — Code Review (gateway/, kit/)

Status: **all P0 + P1 items fixed and verified** (smoke suite 25/25 green,
`kit` builds clean, manual verification of every fix below). P2 items also
closed. This file is the review + fix log for the pass done ahead of the
"huge update" announcement.

Reviewed: `gateway/` (Node, SQLite-backed HTTP gateway) and `kit/` (TS x402
middleware). Every item cites the file/line and was verified against real
running code, not just read.

---

## P0 — fixed

### 1. ✅ `X-Forwarded-For` spoofing bypassed rate limits
**Was:** any client could set `X-Forwarded-For: <random>` per request and
get a fresh IP-keyed rate-limit bucket every time — defeated the 5/hour
guard on the open `POST /keys` endpoint entirely.
**Fix:** `gateway/server.js` `clientIp()` now only trusts XFF when the
immediate TCP peer is in `TRIBUTE_TRUSTED_PROXIES` (new env var, documented
in `.env.example`). Otherwise uses `req.socket.remoteAddress` directly.
**Verified live:** minted 5 keys (201×5), 6th → 429. Retried with a spoofed
`X-Forwarded-For: 1.2.3.4` header (no trusted proxy configured) → still 429.

### 2. ✅ `balance.js` and `x402-routes.js` finished the SQLite migration
**Was:** these two modules still did full read-JSON→mutate→`writeFileSync`→
`renameSync` on every credit/debit/key-mint (balance.js — the actual
escrowed-USDG ledger) and every 402/create hit (x402-routes.js).
**Fix:** added `balances`, `balance_keys`, `balance_history`, `routes`,
`route_activity` tables to `gateway/db.js`; rewrote both modules to use
indexed prepared statements, same pattern as the rest of the codebase.
Public API (function signatures/return shapes) unchanged — no caller edits
needed elsewhere.
**Verified:** roundtrip test — credit/debit/key-create/route-create, then a
**fresh process load** against the same DB file confirmed balances, keys,
routes, and activity all persisted correctly (see terminal output: `balance
survived restart`, `routes survived restart: [verify-test, signals,
market-analytics, alerts]`).

### 3. ✅ Unauthenticated route creation — DoS floor added
**Fix:** `x402-routes.js` `create()` now rejects with a clear error once the
registry hits `MAX_ROUTES` (2000), combined with fix #1 (spoofable IP
buckets no longer bypass the existing DEFAULT rate limit on `POST
/x402/apis`).

### 4. ✅ `facilitator.js` opened a fresh RPC provider/wallet per call
**Was:** `verify()`, `settle()`, `publicView()`, `refundPayment()`,
`payout()`, `finalizeSplits()`, `currentBlock()` each called `provider()` /
`settlementWallet()`, constructing a new `ethers.JsonRpcProvider`/`Wallet`
every time — up to 3-4 fresh connections per single `/facilitator/settle`
call.
**Fix:** both are now memoized module-level singletons, created once and
reused for the process lifetime.
**Verified:** smoke suite's facilitator tests (signature recovery, balance
check, tampered/expired rejection) all still pass; behavior identical,
connection reuse confirmed by code path (no `new ethers.JsonRpcProvider`
inside the hot functions anymore).

---

## P1 — fixed

### 5. ✅ x402 v2 discovery didn't advertise the session scheme
**Was:** `/openapi.json` and `/.well-known/x402` only ever showed
`scheme: 'exact'` — the `tribute-session` option (0.01 USDG → 100 calls/1h,
~100x cheaper per call, the actual headline of "instant settlement, same
gateway") was invisible to any agent/scanner discovering routes via the
documented integration path; it only appeared in the runtime 402 body for
`/x402/premium`.
**Fix:** every paid op in `/openapi.json` now lists BOTH protocols
(`x-payment-info.protocols[0]` = exact, `[1]` = tribute-session), plus new
`/session/redeem` and `/session/{id}` path entries. `.well-known/x402`
`instructions` field now explains both schemes.
**Verified live:**
```
protocols count: 2
 - exact
 - tribute-session
session paths present: True True
```

### 6. ✅ License mismatch
`gateway/package.json` `"license"` changed from `ISC` → `MIT`, matching
README, kit, and the announcement copy.

### 7. ✅ `kit/` (publishable npm package) now builds
**Was:** `node_modules` never installed; `tsc -p .` failed on every import.
Also found mid-fix: `better-sqlite3@^11.3.0` doesn't have a prebuilt binary
for this Node version and fails to compile from source here (missing-field
gcc warnings escalated to errors in this toolchain).
**Fix:** bumped `kit/package.json` to `better-sqlite3@^12.11.1` (same
version already proven working in `gateway/`), fresh `npm install`
(prebuilt binary, no compile), `npm run build` → clean.
**Verified:** `tsc -p .` exits 0, no errors.

### 8. ✅ Test runner wired up
`gateway/package.json` `"test"` now runs `node test/smoke.js` instead of
the placeholder failure. `npm test` in `gateway/` runs the real 25-check
integration suite.

### 9. ✅ Dead code in `/reputation`
The computed-but-unused `addr` variable now actually feeds the response —
`GET /reputation?address=0x...` returns a `queried` field with that
address's score, in addition to the existing `self` + `leaderboard`.

### 10. ✅ Rate-limit buckets now survive restarts
**Was:** `ratelimit.js` was pure in-memory — every restart/deploy reset
every bucket to zero, including the 5-keys/hour guard on the open `/keys`
endpoint (a deploy moment being the easiest time to mass-mint keys).
**Fix:** added `ratelimit_buckets` table; `check()` is now write-through to
SQLite, loaded back into the in-memory cache on module init.
**Verified:** filled a 3-request bucket, reloaded the module in a fresh
process against the same DB file, confirmed the 4th request was still
denied (bucket state survived).

---

## P2 — fixed

- **Vault key fallback now warns on boot.** `agent-vault.js` logs
  `console.warn(...)` once at require-time when `TRIBUTE_VAULT_KEY` is
  unset, instead of silently deriving from `TRIBUTE_RPC_UPSTREAM`.
- **README duplicate "Gateway endpoints" section removed.** Merged the
  extra rows (`/analytics/keys`, `/reputation/:address`) into the single
  earlier section; the second, differently-formatted duplicate near the
  License section is gone.
- **Stale EIP-712 domain name fixed in examples/tests.** `examples/pay.js`
  and `gateway/test/smoke.js` signed against `"Global Dollar"` (rejected by
  the current facilitator, which expects `"TRIBUTE"` per the Sept fix).
  Both now use `"TRIBUTE"`.
- **Bonus bug found while fixing the above:** `examples/pay.js` was reading
  the `x-payment-required` header (the *v2*-encoded, different-codec
  header added by the v1/v2 dual-emission change) and `JSON.parse`-ing it
  as if it were the legacy v1 body. Fixed to read
  `x-payment-required-v1` instead, which is the header that actually
  matches the JSON shape this example parses (`maxAmountRequired` etc).
  This would have made the example script throw on every run.

---

## Not touched (deliberately out of scope)

- `x402-routes.js`'s `slugify()`/route-shape behavior unchanged — only the
  storage backend moved.
- No change to on-chain settlement logic (`verify()`/`settle()` math), only
  connection lifecycle.
- Marketplace (`marketplace.js`) still on flat-JSON persistence — smaller
  blast radius than balance/routes (fee-split escrow logic is unaffected;
  listings/subs data, not live USDG balances) and out of scope for this
  pass. Flagging as a good next item if traffic grows.

## Verification summary

| Check | Result |
|---|---|
| `node --check` on all 12 touched gateway JS files | all pass |
| `npm test` (gateway, 25-check integration smoke suite) | **25/25 pass** |
| `npm run build` (kit, TypeScript) | clean, 0 errors |
| Live server: `/openapi.json` session scheme discovery | confirmed (2 protocols listed) |
| Live server: XFF spoof no longer bypasses rate limit | confirmed (429 with and without spoofed header) |
| balance.js / x402-routes.js SQLite roundtrip + restart persistence | confirmed |
| ratelimit.js bucket persistence across restart | confirmed |
| Vault key warning fires when `TRIBUTE_VAULT_KEY` unset | confirmed in smoke test output |

Not yet done: commit + push to `tributelab/tribute`, and no CI workflow was
added (flagged in the original review as a "worth doing" item — a GitHub
Action running `npm ci && npm test` in `gateway/` and `npm ci && npm run
build` in `kit/` would catch regressions like the kit build break
automatically next time; skipped here since it wasn't asked for).
