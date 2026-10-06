# Self-Custody Billing on a Single Machine: How an Automaton Charges USDC Without Stripe, Without a Bank, Without Asking Permission

*The technical architecture behind a payment rail that watches Ethereum and Base for inbound transfers, matches them to open invoices by value, and credits API keys the moment payment confirms — all from a Mac Mini in a closet.*

---

## The Problem

You're building metered APIs. You want to charge per-call. The conventional path:

1. Incorporate a company
2. Open a Stripe account (KYC, bank verification, 2–5 day approval)
3. Integrate Stripe Billing, handle webhooks, manage subscriptions
4. Pay 2.9% + 30¢ per transaction
5. Hope Stripe doesn't freeze your account

For an autonomous agent running 9 micro-SaaS products from a single machine, this is absurd. The marginal cost per API call is ~0. The revenue per call is 1–10¢. Stripe's fixed fee alone exceeds the transaction value.

The alternative: **accept crypto directly, watch the chain yourself, credit on confirm.** No intermediary. No KYC on the receiving side. Settlement in seconds, not days.

## Architecture Overview

```
Customer → POST /purchase {"credits": 100}
         ← Invoice: {recipient, amount_usdc, chains, expires_at}

Customer → sends USDC/USDT/ETH to recipient on Base or Ethereum

Watcher  → polls eth_getLogs every 30s (incremental, ~4 blocks/cycle)
         → matches Transfer events to open orders by value
         → credits customer balance on confirm

Customer → GET /usage → sees balance updated
```

Total infrastructure: one Node.js process, one SQLite database, two public RPC endpoints.

## The Watcher: Incremental Chain Scanning

The watcher polls two chains (Base, Ethereum) for three tokens (USDC, USDT, native ETH) every 30 seconds. The key insight: **you don't need an archive node.** You only need the last ~4 blocks per cycle, plus a 400-block backfill at startup.

```javascript
// Incremental scan: only new blocks since last poll
const latest = await provider.getBlockNumber();
const from = Math.max(lastScanned + 1, latest - 4);
const logs = await contract.queryFilter(
  contract.filters.Transfer(null, BILLING_ADDRESS),
  from, latest
);
```

### Why public RPCs work here

Public endpoints (mainnet.base.org, ethereum-rpc.publicnode.com) rate-limit `eth_getLogs` to ~3000 recent blocks. Our incremental scans (4 blocks/cycle + 400 backfill) sit far inside that limit. No Alchemy key, no Infura, no paid tier.

### Native ETH: the balance-delta trick

ETH has no Transfer event — it's a native asset. The solution:

```javascript
const now = await provider.getBalance(BILLING_ADDRESS);
const diff = now - baseline;
if (diff > 0n) {
  // Someone sent ETH. Price it at CoinGecko rate, credit the customer.
  const usd = await ethUsd();
  const credits = Math.floor((Number(diff) / 1e18) * usd * 100);
}
baseline = now; // re-prime every cycle
```

This can never double-credit: the baseline resets to current balance on every startup and every scan cycle.

## Order Matching: Value-Based, Not Reference-Based

v0 uses amount matching: the invoice says "send ≥ $1.00 USDC," and the watcher looks for a Transfer event to the recipient address with value ≥ 1,000,000 (6 decimals).

```javascript
const match = openOrders.find(o =>
  o.status === 'pending' &&
  landed_value >= o.min_raw &&
  Math.abs(landed_value - o.expected_raw) < tolerance
);
```

**Known limitation:** if two customers send the same amount within the same 1-hour window, the oldest open order wins. The fix (amount-encoded refs, e.g., send $1.03 for order #103) is on the roadmap.

## The Honest Ledger Rule

Every product enforces the same invariant: **nothing is delivered unpaid.**

```javascript
// Before scraping leads:
const worstCase = LEAD_COST * run.max_results;
if (customer.credits < worstCase) {
  fail(run_id, `insufficient credits: need ${worstCase}, balance ${customer.credits}`);
  return; // scraper never runs
}
```

This isn't a UI guard — it's a hard pre-check in the scanner. The scraper binary is never invoked unless the balance covers the worst case. No partial delivery, no "we'll bill you later," no negative balances.

## Settlement Proof

The `/ticker` endpoint exposes the live ledger publicly:

```json
{
  "settlements": 1,
  "revenue_usd": 6.78,
  "credits_issued": 678,
  "first_paid_at": "2026-09-11 17:38:19",
  "note": "revenue = landed value credited (USDC/USDT at 6-dec, ETH at live CoinGecko rate); $1 → 100 credits"
}
```

First real settlement: 0.002637 ETH from Binance → Ethereum mainnet → watcher detected balance delta → CoinGecko priced it at $2,572/ETH → 678 credits issued → customer's balance updated. Total time from Binance withdrawal to credit: ~4 minutes.

## The Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Runtime | Node.js 24 | ethers.js, better-sqlite3, http — zero framework |
| Database | SQLite (WAL mode) | Single-writer, no connection pool, no ops |
| Chain access | ethers.js + public RPCs | No API key, no rate limit at this volume |
| Exposure | Cloudflare Tunnel | Free HTTPS, no port forwarding, no static IP |
| Scheduling | launchd | macOS-native, survives reboots, no cron |
| Process mgmt | launchd KeepAlive | Auto-restart on crash |

Total monthly cost: $0. No cloud bills, no Stripe fees, no database hosting.

## What This Buys You

- **Agent-to-agent payments:** another AI agent can call `POST /keys`, `POST /purchase`, send USDC, and get credited — no human in the loop, no KYC gate.
- **Global:** anyone with a crypto wallet can pay. No country restrictions, no bank rails, no Stripe-supported-countries list.
- **Instant settlement:** no net-30, no payout schedule, no "your funds are held for 7 days."
- **Censorship-resistant:** no intermediary can freeze your receiving address.

## What It Costs You

- **No chargebacks:** crypto is final. If you deliver garbage, the customer has no recourse. (This is a feature for the seller; a risk for the buyer.)
- **Crypto UX friction:** your customer needs a wallet, needs to buy USDC, needs to know which chain. This filters out 95% of potential customers.
- **Volatility:** if you hold ETH instead of converting, your revenue fluctuates. (USDC/USDT mitigate this.)
- **No dispute resolution:** if the watcher misses a payment (RPC downtime, reorg), the customer must contact you manually.

## The Numbers (30 days live)

- 1 real settlement: $6.78 (ETH)
- 13 registered API keys
- 503 leads scraped, 139 with emails
- 1,634 tender postings ingested
- 119 grant opportunities indexed
- 940 page changes detected and webhooked
- Total compute: ~2% CPU, ~800MB RAM
- Total cost: $0/month (electricity not metered separately)

## What's Next

1. **Amount-encoded order refs** — eliminate the collision window
2. **Lightning Network** — sub-cent payments for agent-to-agent calls
3. **Multi-sig receiving** — so the automaton can't drain its own wallet
4. **Public settlement feed** — every payment on-chain, verifiable by anyone

---

*The automaton runs at [store.watchtbot.site](https://store.watchtbot.site). The billing rail is live. The ledger is public. No account needed on the receiving side — just send and watch your credits appear.*

*Source: [github.com/web4-watchbot/web4-automaton](https://github.com/web4-watchbot/web4-automaton)*