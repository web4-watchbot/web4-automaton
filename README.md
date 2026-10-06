# web4-automaton

A self-hosted, self-custody micro-SaaS platform running on a single Mac Mini.
Nine products, one billing rail, zero cloud bills.

## Products

| # | Product | Price | Status |
|---|---------|-------|--------|
| 01 | Watchbot (URL change detection) | 1¢/check | live |
| 02 | Grant Radar | 5¢/match | live |
| 03 | Tender Radar | $49/30d | live |
| 04 | Radar Packs | 10¢ arm | live |
| 05 | Visa Slot Radar | $19/30d | live |
| 06 | WhaleBot (chain watches) | 5¢/alert | live |
| 07 | Weekly Digest | 25 cr/send | live |
| 08 | Lead Radar (Google Maps) | 10¢/lead | live |
| 09 | MCP Server (9 tools) | per-call | live |

## Architecture

Single Mac Mini → Node.js + SQLite → Cloudflare Tunnel → public HTTPS.
Billing: USDC/USDT/ETH on Base + Ethereum, self-custody, watcher auto-credits.

## Documentation

- [Self-Custody Billing Rail Deep-Dive](docs/billing-rail-deep-dive.md) — how the automaton charges USDC without Stripe, without a bank, on a $0/mo stack

## Live

- Storefront: [store.watchtbot.site](https://store.watchtbot.site)
- API docs: [store.watchtbot.site/docs](https://store.watchtbot.site/docs)
- Health: [store.watchtbot.site/health](https://store.watchtbot.site/health)
- Ticker: [store.watchtbot.site/ticker](https://store.watchtbot.site/ticker)
