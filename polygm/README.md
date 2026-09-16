# PolyGM prototype

Working reference implementation for the plan in `../PLAN.md`.
Read-only. No private keys, no orders, no money moves.

## Run

```bash
PORT=8080 python3 server.py
```

No dependencies — stdlib only (Python 3.10+).

## What it does

Pulls live data from four public Polymarket endpoints (no auth needed):

| Endpoint | Used for |
|---|---|
| `gamma-api.polymarket.com/events` | event + market metadata, volume, liquidity, open interest |
| `clob.polymarket.com/book` | order book depth per outcome token |
| `data-api.polymarket.com/trades` | the live tape |
| `lb-api.polymarket.com/volume` | all-time trader leaderboard |

A background thread rebuilds a shared cache every `REFRESH_SECS` (default 20)
so N users cost 1 request, not N. Polymarket's Cloudflare limits are per-IP,
which makes server-side caching mandatory rather than an optimisation.

## Endpoints

| Path | Returns |
|---|---|
| `GET /` | the Mini App UI |
| `GET /api/state` | everything + `_meta` (cache age, upstream errors) |
| `GET /api/events` \| `/api/tape` \| `/api/leaderboard` | single collections |

## UI tabs

- **Markets** — sortable by 24h volume / liquidity / open interest / ending soon. Tap → order book + paper order.
- **Live tape** — every fill, newest first, ≥$1k flagged `WHALE`.
- **Whales** — all-time volume leaderboard straight from Polymarket.
- **Trade** — portfolio stub + the documented production wallet/order flow.

Paper orders persist in `localStorage` and never touch the chain.

## What the real build changes

1. Polling → CLOB WebSocket `wss://ws-subscriptions-clob.polymarket.com/ws/market`
2. In-memory cache → Redis + ClickHouse (this is what becomes the paid analytics/API tier)
3. Add a **separate** trading service: managed wallets → `py-clob-client-v2` → signed
   CLOB V2 order with `builderCode` → `POST /order`. Never co-locate keys with the HTTP server.
4. Risk service (per-user daily cap, kill switch) *in front of* the order path.

## Gotchas already handled / worth knowing

- CLOB **V1 is dead** (mandatory V2 since 28 Apr 2026); collateral is **pUSD**, not USDC.e.
- `minimum_order_size` is **5 shares**; tick size varies per market — off-tick orders are rejected.
- `DELETE /cancel-all` is limited to 250 per 10s; `GET /balance-allowance` only 200. Cache positions.
- Check `accepting_orders`, `seconds_delay`, `neg_risk` per market before submitting.
