# SHARED CONTEXT
### Paste this into EVERY session before the task prompt.
*Last verified: 16 Sep 2026. Re-verify anything marked ⚠️ before shipping.*

---

## 1. What we are building

A **non-custodial trading terminal for Polymarket**, modelled on gmgn.ai's product shape (analytics + one-tap execution + Telegram bot), branded in Polymarket's visual language.

- We are **not** an exchange. No custody, no listings, no fiat on-ramp, no order book of our own.
- Users trade into **Polymarket's** order book. Funds live in a wallet the user controls and can export.
- Revenue: **Polymarket Builder Program fees** (primary) + Pro subscription + data/API licensing.

---

## 2. Polymarket APIs — verified live on 16 Sep 2026

| API | Base | Auth | Use |
|---|---|---|---|
| **Gamma** | `https://gamma-api.polymarket.com` | none | events, markets, tags, search, metadata |
| **CLOB** | `https://clob.polymarket.com` | none for data; L2 for orders | books, prices, history, order placement |
| **Data** | `https://data-api.polymarket.com` | none | trades, positions, holders, activity |
| **Leaderboard** | `https://lb-api.polymarket.com` | none | `/volume?window=all&limit=N` |
| **WebSocket** | `wss://ws-subscriptions-clob.polymarket.com/ws/market` | none for market channel | `book`, `price_change`, `last_trade_price`, `tick_size_change` |

### Gamma linking flow (the one everyone gets wrong)
```
GET /events?slug=<slug>  →  markets[].clobTokenIds  (JSON *string*, parse it)
                         →  markets[].conditionId
                         →  use tokenId with CLOB /price, /book
```
A market is tradable only when `enableOrderBook == true`.

### CLOB endpoints
`GET /price`, `GET /prices`, `GET /book`, `POST /books`, `GET /midpoint`, `GET /spread`, `GET /prices-history`, `GET /markets`, `GET /markets/{conditionId}`, `GET /last-trade-price`
Authed (L2): `POST /order`, `POST /orders` (batch ≤15), `DELETE /order/{id}`, `DELETE /orders`, `DELETE /cancel-all`, `GET /data/balance-allowance`, `GET /data/trades`, `GET /orders`

### Per-market fields you MUST check before submitting an order
`accepting_orders` · `seconds_delay` · `minimum_order_size` · `minimum_tick_size` · `neg_risk` · `enable_order_book`

Verified live example (Fed market, 16 Sep 2026): `minimum_order_size: 5`, `minimum_tick_size: 0.001`, `neg_risk: true`, `accepting_orders: true`, `seconds_delay: 0`.

### Rate limits (documented) — **these shape the architecture**
| Endpoint | Burst (10s) | Sustained (10 min) |
|---|---|---|
| CLOB general | 9,000 | — |
| `POST /order` | 5,000 | 120,000 |
| `POST /orders` (batch ≤15) | 2,000 | 21,000 |
| `DELETE /order` | 5,000 | 120,000 |
| `DELETE /orders` | 2,000 | 15,000 |
| `DELETE /cancel-all` | **250** | 6,000 |
| `DELETE /cancel-market-orders` | 1,500 | 21,000 |
| `GET /balance-allowance` | **200** | — |
| Gamma general | 4,000 | — |
| Gamma `/markets` | **300** | — |
| Gamma `/events` | **500** | — |
| Data general | 1,000 | — |
| Data `/trades` | 200 | — |
| Data `/positions` | 150 | — |

Limits are **per-IP** (Cloudflare) *and* **per-signer** (token buckets on orders/cancels). ⇒ Server-side caching is mandatory, not an optimisation.

---

## 3. CLOB V2 — non-negotiable facts

⚠️ **V1 is dead. V2 has been mandatory since 28 April 2026.** Every tutorial pre-dating that is wrong.

| | V1 (dead) | V2 (current) |
|---|---|---|
| SDK | `py-clob-client` / `@polymarket/clob-client` | `py-clob-client-v2` / `@polymarket/clob-client-v2` |
| Constructor | positional args | options object; `chainId` → `chain` |
| Order fields | `nonce`, `feeRateBps`, `taker` | `timestamp` (ms), `metadata`, `builder` |
| Fees | embedded in signed order | **set by protocol at match time** |
| Collateral | USDC.e | **pUSD** (ERC-20, backed by USDC) |
| Builder attribution | `POLY_BUILDER_*` HMAC headers | one `builderCode` field on the order |
| EIP-712 Exchange domain | version `"1"` | version `"2"` (API auth unchanged) |
| Base URL | `https://clob.polymarket.com` | unchanged |

V2 signed order struct: `salt, maker, signer, tokenId, makerAmount, takerAmount, side, signatureType, timestamp, metadata, builder`

Signature types: `Eoa`, `PolyProxy`, `PolyGnosisSafe`, `POLY_1271` (type 3, ERC-7739-wrapped, for new deposit-wallet accounts — both maker and signer must be the deposit wallet address).

Two credential layers: **L1** = wallet private key (signs); **L2** = api key + secret + passphrase (CLOB auth). Never hardcode either.

**Because fees are set at match time, you cannot compute a user's exact fee client-side.** Show an estimate; for market buys use an all-in spending limit so the order amount is adjusted for fees before signing.

---

## 4. Fees

### Platform fees (taker-only) — Fee Structure V2, effective 30 Mar 2026
```
platform_fee = C × feeRate × p × (1 − p)
```
| Category | feeRate | Max per 100 shares @ 50¢ | Maker rebate |
|---|---|---|---|
| Politics / Finance / Tech / Mentions | 0.04 | $1.00 | 25% |
| Sports | 0.05 | $1.25 | 15% |
| Economics / Culture / Weather / Other | 0.05 | $1.25 | 25% |
| **Crypto** | **0.07** | **$1.75** | 20% |
| Geopolitics | **0** | $0 | — |

**Makers pay zero** in every category and earn daily rebates in pUSD. No fee at settlement (formula → 0 at p=0 or p=1).

### Builder fees (ours)
- Register at `polymarket.com/settings?tab=builder` → `bytes32` builder code. Approval 1–2 weeks.
- **Caps: taker 100 bps (1%), maker 50 bps (0.5%).** Default 0. Granularity 1 bp.
- `builder_fee = notional × rate_bps / 10000`. **Additive** to platform fees — user pays both.
- Rate changes: one per 7 days, 3 days advance notice, one pending change at a time.
- Only accrues on **matched** orders.
- Maker and taker sides can carry different builder codes and rates.
- Polymarket **can revoke your fee privilege at its sole discretion** and disable your code — orders with a disabled code are rejected by the CLOB. Revocation grounds include "self-referred or non-genuine trading activity."
- Builder profiles and rates are **publicly queryable**.

**⇒ Launch at 0 bps. Betmoar and Stand.trade both charge zero and are top-5.**

### US arm (separate)
Polymarket acquired CFTC-licensed QCEX for $112M; got CFTC approval as a Designated Contract Market (Oct 2025). The US product has its own schedule: uniform 0.05 taker, −0.0125 maker rebate, effective 3 Apr 2026. **Our tool routes to the international CLOB ⇒ geofence US users out of trading.**

---

## 5. Market size — measured live 16 Sep 2026, 15:07 UTC

| Metric | Value |
|---|---|
| 24h volume, top 500 active events | **$59.1M** |
| Liquidity (same set) | **$477.3M** |
| Open interest | **$287.7M** |
| Top-10 events' share of volume | **58.4%** |
| Median event 24h volume | **$19,910** |
| Fill rate on `/trades` feed | **~20.8 trades/sec** |
| Unique wallets per 500 trades | 342 |

Run rate ≈ $600–750M/month, $8–10B/year. Highly concentrated — long tail is dead water.

### Builder ecosystem (your revenue pool)
- Weekly attributed volume hit **$125M** in Feb 2026 (3rd consecutive week >$100M), ~114 builders.
- 30d top 10: **Betmoar $101M**, PolyCop $38.8M, WagerUpPilot $24.2M, PolyTraderPro $18.9M, Polymtrade $13.5M, Kreo $10.8M, Stand $9.6M, Polygun $9.5M, Chance $9.4M, Gate $9.0M.
- Gini **0.83** across top 50. Top 6 = **81%** of lifetime volume. Median top-50 builder: **$4.7M lifetime**.
- Q1 2026: **80% of builders did <$1M for the quarter**; a third never crossed $10k.
- Only disclosed economics: **Based — ~$1M ARR at ~$10M/month** ⇒ implied ~0.83%. PolyTrack estimates 0.5–1%.

### ⇒ Revenue reality
```
$1M ARR ÷ 1.0% = $100M/yr routed = $8.3M/month  →  top-3-builder territory
```
**Year 1 realistic: $80k–$400k.** $1M ARR is a year-2/3 target. Never let builder fees exceed ~60% of revenue — Polymarket can switch it off.

---

## 6. Competitors

**Analytics (saturated, $6–30/mo):** Hashdive (72k visits/mo, free), Polywhaler (30k, the original), Polymarket Analytics (free + $20/mo, 1 ETH lifetime), Polysights (24k, pre-1.0), PolyTrack ($9.99/wk, $19/mo), PredictFolio (free, CC BY-NC), OrcaLayer ($9.99/$19.99), PolyMonit ($5.99/$9.99), PolySharks ($19.99), Unusual Predictions ($30–80/mo), Alphascope, Polyburg, Merlin Trade.

**Telegram bots:** Betmoar (leader, anonymous, no press), PolyCop, TradePolyBot, Polyfox, Polygun, PolyBot, Polylerts, PolyTracker Bot, PolyxBot, PolyCopy, Polycopybot, Kreo, Stand.trade, Chance, Rainbow.

**The gap:** every Telegram bot here is an inline-button menu. Nobody ships a real *terminal* inside Telegram. That's the wedge.

**Benchmark:** gmgn.ai does **$46.74M fees / $38.35M revenue per 30 days** ($167.52M annualised revenue), on a flat ~1% per-trade fee with a custodial hot wallet. That's the category ceiling, not prediction markets'.

---

## 7. Brand tokens — extracted from polymarket.com's compiled CSS

⚠️ These are Polymarket's production tokens. **Do not ship Polymarket's exact logo, wordmark, or marketing copy.** Use the palette and type as inspiration and build an original identity (see P2).

### Brand (blue) scale
```
light:  50 #f4fcff   100 #cadfff  200 #a0c1ff  300 #76a2ff  400 #4e7fff
        500 #2e5cff  600 #1c3fe2  700 #0f1ac6  800 #0c00a4  900 #06006f
dark:   50 #020041  100 #060071  200 #0d00a8  300 #1020c9  400 #1e44e7
        500 #3262ff  600 #5485ff  700 #7ba5ff  800 #a1c0ff  900 #c7daff
```
**Primary action: `#2e5cff`** (this is `--color-pk-brand-500`, used as the progress-bar colour).

### Neutral scale (note: 50 = darkest in dark theme, lightest in light)
```
dark:   50 #0c0e13 (base bg)  100 #1c1f27 (elevated)  200 #2d3037 (border)
        300 #45474e  400 #62646a  500 #7d8189  600 #8b929b  700 #6b727b
        800 #454a52  900 #23272d  950 #000
light:  50 #fff      100 #f4f5f7  200 #e6e8ec  300 #d4d7dc  400 #bfc3ca
        500 #abb2bb  600 #999ea7  700 #b6bbc4  800 #d2d4d9  900 #edeff1
```
### Semantics
`--color-pk-surface: gray-50` · `--color-pk-elevated: gray-100` · `--color-pk-border: gray-200` · `--color-pk-brand-strong: brand-700` · `--color-pk-brand-subtle: brand-200` (dark) / `brand-50` (light)

### Data-viz palette (Polymarket's own chart colours)
`#87BFFF` · `#4378FF` · `#2797FF` · `#FDC503` · `#FF7F0E` · `#144E8C`

### Semantic (derived from their CSS — ⚠️ not confirmed as named tokens)
success/yes `#16a34a` · danger/no `#ef4444` · warning `#f1ce57`

### Type
```
Body:      Inter Variable (100–900)
Headlines: Instrument Sans Condensed (600, 700)   ← the distinctive one
Numbers:   Geist Mono
Display:   Open Sauce One
```
### Radius & weight
`--radius: .7rem` (11.2px) · `xs` −6px · `sm` −4px · `md` −2px · `lg` = base · `xl` +4px · `2xl` 1rem · `3xl` 1.5rem
Weights: light 300 · normal 400 · medium 500 · semibold 600 · bold 700 · extrabold 800

### Feel
Dense, data-first, financial-terminal. Light is the default brand surface; **build dark-first for the terminal** and ship light as parity. High information density, tight type, mono for every number, thin borders, minimal shadow, no gradients on chrome.

---

## 8. gmgn's information architecture (patterns to rebuild, not copy)

Extracted from gmgn's own documentation. These are the product patterns that make the product work.

**Terminal = 3 columns**
- **Left rail:** Holding / Watchlist / Following lists, then Trending / Pump (new) lists
- **Centre:** chart (multi-chart up to 8, selectable granularity, trade markers on candles, avg-price lines, limit-order lines) → below it the activity tabs: **Activity · Traders · Holders** → **Position / Limit / Auto**
- **Right rail:** project metadata/socials, the **trading module** (wallet switcher), pool/contract info

**Activity classification taxonomy** — every trade is tagged by wallet type:
`smart money · KOL/VC · whale · new wallet · sniper · large holder · developer · followed · rat warehouse`

**Per-trader metrics:** SOL bal / account age · funding source + transfer time · bought / sold · total PnL (realised + unrealised) · avg cost / avg sold · TXs (buys green, sells red)

**Wallet profile (7D/30D):** PnL % + amount · win rate · balance · TX counts · distribution of buys/sells · **phishing check**: blacklist count, "didn't buy" (transfer-in) count, sold>bought count, **buy/sell within 10s count** (bot/copy-farm detector)

**Wallet Radar** — scan up to 10 tokens at once, rank wallets four ways: **Most Bought · Highest Profit · Earliest Bought · Shared Holdings.** Then one-click Track or Copy Trade.

**Copy trading:** follow up to 10 wallets, set multiplier, per-trade cap, daily cap, category filter, TP/SL, optional dev-sell automation.

**AFK automation:** rule-based auto-buy/auto-sell, limit orders, TP/SL, condition triggers, executes unattended.

**Alerts:** FOMO alerts on new listings, sub-second exchange-listing alerts, wallet alerts with sound toggle, push via Telegram bot.

**Two surfaces, one wallet:** web terminal for scanning/analysis, Telegram bot for fast execution. Same account, same balance.

---

## 9. Telegram constraints

- Digital goods/services sold **inside** Telegram must be paid for **exclusively in Telegram Stars (XTR)**. No crypto, no third-party processor inside the Mini App. This is for App Store/Play compliance.
- Stars ≈ $0.013–0.015. App stores take up to 30%. Withdrawal via Fragment: 21-day hold, 1,000-Star minimum.
- ⇒ **Sell Pro on the website (Stripe + crypto). Offer a Stars-priced equivalent inside Telegram.**
- If you distribute your own token via a Mini App, Telegram requires **TON** and removes apps distributing Ethereum/BNB assets. **No token in year one.**
- Mini App = full web app inside Telegram, inherits user identity, no separate login.
- BotFather is free. Bot usernames are permanent-ish — pick carefully.

---

## 10. Regulatory posture

- **Non-custodial ⇒ not an exchange.** No MSB registration, no money-transmitter licences, no VASP registration for the tool itself.
- **But** you operationally hold keys that can move user money. Treat it like a custodian even though you aren't one legally. One breach ends the company.
- **Geofence US users out of trading** (international CLOB ≠ US-regulated product).
- **No "guaranteed returns" language anywhere.** Show losing wallets next to winning ones. Copy-trading must display drawdown, not just PnL.
- **India (founder is in Surat):** a non-custodial analytics/execution tool for global users is not a reporting entity under FIU-IND's VDA regime the way an exchange is. Get one paid legal opinion rather than guessing. Model 30% VDA tax + 1% TDS for any Indian users.
- **Never wash-trade to climb the builder leaderboard.** That's an explicit revocation ground.

---

## 11. Reference implementation

`/home/user/polygm/server.py` — zero-dependency Python: live Gamma + CLOB + Data + leaderboard ingestion, shared cache refreshed every 20s, 5 JSON endpoints, static serving. Verified working: 60 events, 300 tape rows, 24 books, 25 leaderboard rows, `errors: []`.

`/home/user/polygm/public/index.html` — mobile-first Mini App UI, 4 tabs, live tape with whale flags, order-book depth sheet, paper-trade flow.

Use it as the starting point for P4–P6, not as a throwaway.

---

## 12. Corrections from the P01 build session (added 2026-09-16 16:31 UTC, nothing above deleted)

Verified live from the build workspace with `tools/datasource-probe.py` (26/26 endpoint assertions).
These are fact corrections to §2 and §6, so later sessions do not re-inherit them.

| § | Stated | Correct |
|---|---|---|
| §2 | Gamma `/markets` 300, `/events` 500 per 10 min (implies large pages) | **Pages are capped at 100 rows** regardless of `limit=`; default is 20. `offset=` works. Any top-N volume figure needs pagination (300 rows → $59.2M, which reproduces §5's $59.1M). |
| §11 | "Fill rate on `/trades` feed ~20.8 trades/sec" | `data-api /trades` is **served from Cloudflare cache** (`cf-cache-status: HIT`; byte-identical 8 s apart, one pull 300 s stale). Measured tape rate **14.7–33.3 fills/sec**. It is a snapshot for backfill, **not a feed** — real-time tape requires the WebSocket. |
| §5 | Wedge "5-minute crypto Up/Down — highest frequency, highest fee rate" | Up/down markets are **≤0.14% of top-100 market volume** and **absent entirely** from volume-ordered `/events` (0 of 300; event-level `volume24hr` is null/0 there). Frequency ≠ volume; do not size a product on it. |
| §2 | Leaderboard `lb-api /volume` | Also `GET /profit` (both for `window=1d\|7d\|30d\|all`). **`/pnl` does not exist (404)** and **`/rank` is unusable** (400 naming a missing `rank` param even when it is supplied). PnL/rank must be computed by us and labelled as our estimate. |
| §2 | Data API "trades, positions, holders, activity" | Exact required params: `/positions?user=`, `/activity?user=`, `/value?user=`, `/traded?user=`, `/holders?market=` (400 with the param name in the error otherwise). **There is no `/profile` endpoint (404)** — trade rows already carry `name, pseudonym, bio, profileImage, title, eventSlug, icon, transactionHash`. |
| §3 | Fee category table lives only in this doc | Gamma exposes **`feeType` per market** (`crypto_fees_v2`, `politics_fees`, `sports_fees_v2`, `economics_fees`, `culture_fees`, `finance_prices_fees`), so fee category is readable, not guessable. `feeRate` itself was `null` on sampled markets. |
| §4 | (PnL guidance) | `REDEEM` rows in `/activity` have **`price == 0` on 366/366** and carry payout in **`usdcSize`** (nonzero on 280/366, exactly `1:1` with `size` ⇒ $1/share). Computing realised PnL from `price × size` prices every redemption at zero. |
| §6 | "Polywhaler (30k…)" etc. | Competitor onboarding flows are **not** machine-inspectable from a sandbox: 4/6 sites return empty bodies to curl, `polymarketanalytics.com` returns **429 Vercel Security Checkpoint**. Step counts must be measured by a human with a wallet. |

Open upstream gap: `⚠️` on §7 brand tokens was not re-checked this session (CSS-derived values are a
design input, not an API fact).

## 13. Corrections from the P02–P03 build session (added 2026-09-17, nothing above deleted)

These are measurements, not preferences. Where one contradicts a number earlier in this file, the row says
which artifact now owns the corrected value.

| Ref | Claim as written above | Measured |
|---|---|---|
| §12 | `feeType` exposes six values including `finance_prices_fees` | Five are in the retained deep probe (2,500 market rows): `crypto_fees_v2`, `politics_fees`, `sports_fees_v2`, `economics_fees`, `culture_fees`. The sixth came from an earlier pull whose JSON was overwritten, so it is **unretained, not refuted** — and it cannot be re-tested by tag because **Gamma ignores `tag_slug`** (see next row). Treat the taxonomy as *open*: read `feeType` per market, never hard-code an allow-list. Owned by `polygm-platform/docs/P01-product-spec.md` + `tools/p01-gate-check.py` C9. |
| §12 | (method) | **`gamma-api /markets?tag_slug=` is ignored.** `tag_slug=politics`, `tag_slug=sports` and `tag_slug=zzzznotatag` all return identical 40-row sets. Any "no markets match tag X" conclusion drawn from that parameter is void. `order=volume24hr&ascending=false` with `limit`≤100 and `offset` **is** honoured. |
| §12 | (method) | Gamma caps `limit` at **100** and silently ignores larger values (`limit=5000` returns 100 rows). `offset` works, so full sweeps need paging. |
| §11 | (brand) | Product renamed **Openout**; the wordmark is *generated*, not edited (`tools/rename-wordmark.mjs` measures the face and lays out `brand/svg/lockup-horizontal.svg`; `--check` fails if the old name is live or the mark drifted). `brand/svg/mark.svg` is never redrawn; its geometry is hash-pinned (`c33b4d5fd82b7acd`) in `brand/BRAND-KIT.md`, and the pin is re-verified against the file by a gate, not trusted. |
| §11 | raster filenames imply size | Read from PNG headers: `app-icon-512.png` is **1254×1254**, `avatar-512.png` **1254×1254**, `og-1200x630.png` **1731×909** (aspect 1.904:1 vs 1.91:1), `brandboard.png` **1536×1024**. Filenames overstate nothing but do not match; a true 1200×630 derivative is still owed before the landing page ships. No rasteriser exists in the build sandbox, so these cannot be regenerated there — recorded as open, not silently shipped. |
| §7 / brand colour | buy vs sell hues are distinguishable | **Overturned by P03 for text use.** `action.buy` vs `action.sell` differ by ΔL 0.040 (dark) / **0.008** (light); protan ΔE 9.0/6.9 — the same distance P02's own palette search rejected for chart series. `alert.critical` **is** `action.sell` (ΔE 0.0), and `outcome.no` sits ΔE 3.2–4.9 from both. No buy/sell pair reaches 4.5:1 in *both* themes. Consequence adopted in the design system: money magnitude is `text.primary`, direction is a mandatory `+/−` glyph + ▲/▼ caret in a fixed slot + filled/outline chip; severity never shares a row with outcome. Owned by `tools/component-colour-audit.py`. |
| §7 | 600ms price-change flash | Breaks the motion ceiling the kit itself cites (≤300ms per transition). Retokened to **90ms in / 200ms out (290ms)**, background-only, no number animation. Owned by `brand/tokens.json` `motion.flash_on_change`. |
| §7 | (method) | Order books are **not** usually one-sided: 22 sampled books, **0/22** fully one-sided, **8/22** with ≤5 levels on the near side; a Fed market measured **63 asks / 3 bids** (spread 996 ticks) against the "94 asks, zero bids" anecdote. Design for *near-empty*, and keep one-sided as a reachable-but-rare state. |
| general | doc counts | Any count in a spec must be generated or checked: P03's hand-written "~468 named stories" was **594** off once the inventory was enumerated (1,062). |

## 14. Contrast finding that binds the frontend phases (added 2026-09-17)

**No foreground in this product may claim WCAG "AA-large".** The largest text token in the system is 13px
(`density.font_size_px`), and the large-text exemption needs ≥18.66px bold or ≥24px. Therefore every hue used
as text owes 4.5:1, not 3:1 — measured against the background it actually sits on, which for data surfaces is
`bg.elevated`, not `bg.base`.

What that rules out, with the numbers: `outcome.no #D55E00` is 3.87:1 (light/`bg.base`), 3.55:1
(light/`bg.elevated`), 4.26:1 (dark/`bg.elevated`); `action.sell` is 4.43:1 (light/`bg.elevated`) and 4.38:1
(dark/`bg.elevated`). So a YES/NO chip word, a BUY/SELL pill label, a whale badge, or a book-ladder price may
**not** be painted in those hues. Adopted composition: `text.primary` word + hue on the outline/tint/depth
bar (3:1 non-text). Recorded as `tokens.rules["hue-never-small-text"]`.

Process lesson, for anyone extending the audits: `component-colour-audit.py` printed a grade of "AA-large"
for the 3.0–4.5 band while failing only below 3.0 — and its own failure message cited the 4.5:1 body-text
floor it was not applying. A grader can name the right bar and enforce a different one, so a permissive band
needs a canary that must fail when the band returns (here: `CONTROL@hue as small text on an elevated panel`,
verified by restoring the old grader and watching the gate report `canary BROKEN`).

Unchanged and still owed by the brand owner: deepening the money reds (light `#b91c1c` → 6.47:1/5.93:1)
would make a small coloured label legal again; the dark theme has no same-family red that clears 4.5:1 on its
panels. `brand/BRAND-KIT.md` marks these values **fixed**, so this is a decision, not a task.


## 15. Corrections from the P05 build session (added 2026-09-18, nothing above deleted)

Every row is a measurement from `polygm-platform` (fixtures under `tests/fixtures/p05/`, live runs under
`docs/verification/`), not a re-reading of the docs. Where an earlier section recorded something that turned out
wrong, the earlier text stays and this section supersedes it.

| Ref | What was believed | What the venue does, and the consequence |
|---|---|---|
| §12 (probe) | `cache_buster_works` — a query-string buster defeats the CDN cache on `/trades`. **SUPERSEDED 2026-09-18.** | The staleness is origin-side, not edge-side: the unbounded `GET data-api/polymarket.com/trades` view measured **236 s / 257 s / 277 s** behind at samples 20 s apart, with `cf-cache-status: HIT` and byte-identical bodies under `&_=<ms>` busting. `tools/p01-probe.json` is left as recorded (it is the diff baseline for `make probe-fresh`); the *conclusion* is void. Cache-busting appears nowhere in the product, and a gate check fails if it returns. |
| — | `/trades` "latest page" is a fresh view. | It is a *view* that does not refresh. The fresh path is a bounded range: `?limit=100&takerOnly=true&start=<unix_s>&end=<unix_s>` measured **0–1 s** current, 3/3 samples, `cf-cache-status: MISS`. `end` must be ≈ `now + 60` because the indexer stamps some fills up to **+4.3 s** ahead of the clock; a range ending at `now` silently drops the newest fills. Every tape request in the product and in the harness is bounded, and `p05-gate-check.py` enforces it at the call site. |
| §4 (data) | `GET data-api/trades?market=<id>` filters by token id. | It filters by **condition id**. A token id returns 200 with **0 rows**, which looks like an empty market. Separately, `clob…/trades` is authenticated (401 anonymous) — the public tape is the data-api one. |
| §5 (venue) | A WS trade frame identifies a fill. | `last_trade_price` carries **no transaction hash**. So cross-source dedupe by an exact key is impossible and the design says so instead of faking it: **REST is the record, the socket is the latency layer.** The socket feeds the alert engine and the UI's "already booked" marker and never writes the durable tape; duplicate *alerts* are prevented by the rule's dedupe key + cooldown. |
| §5 (venue) | `POST /books` batches book reads. | It returns **400 `{"error":"Invalid payload"}`** for `{"token_ids":[…]}`, `{"asset_ids":[…]}`, a bare array, the query-string form, and with `bids`/`asks` added — including a single valid token id. `GET /book?token_id=` is 200. The per-token budget and the memory arithmetic (2,000 books ≈ 16.5 MB) are sized for polling because batching does not exist. |
| §5 (venue) | Book deltas need a sequence number to be trustworthy. | There is no sequence number, but `price_change` carries **`best_bid`/`best_ask` beside each delta** — that pair is the gap detector: our derived top of book must equal the venue's declared top. The `hash` field is recorded and diffed as a diagnostic only; one capture cannot establish what it hashes, and a check that cannot say what it compares is theatre. |
| §12 | Fill prices are exact decimals. | `data-api/trades` returns some matched prices as IEEE float noise from the venue's own pipeline — **`0.1699999983` for 0.17, in 49 of 200 rows** of the recorded fixture. The strict 6-decimal parser refused them, so a quarter of the tape was missing and the failure read as a quiet market. Fills are parsed with a rounding parser (≤ 5e-7 of movement by construction); **`/book` levels and the order path keep the strict parser**, because a resting order at an off-grid price is a venue bug worth seeing. |
| §5 (venue) | Up/down crypto markets are 5-minute. | `-updown-5m-`, **`-updown-15m-`**, 1h, 4h and daily all exist, and the bucket's start timestamp is **in the slug** — so the lifecycle needs no lookup table. 8,640 up/down markets per day per asset is why prune policy is a first-class feature. |
| §12 | `feeType` is a closed enum. | Seven values across two runs of the same top-100 sample (`crypto_fees_v2`, `culture_fees`, `economics_fees`, `finance_prices_fees`, `politics_fees`, `sports_fees_v2`, `sports_fees_v3`), and the sets differed run to run. Consequence, and it is a money consequence: an unrecognised `feeType` means **charged and flagged**, never free, and no code may branch on "is this in the list I hard-coded". |
| §12 | A transient failure and a shape change are the same event. | They are not, and the checker must say which it saw. `datasource-probe.py --check-cache` prints status and payload per failing id and labels each line `TRANSIENT?` or `SHAPE`; `make probe-fresh` propagates the exit code (it used to be wrapped so that nothing could fail). |
| §13 (schema) | `markets` has a `closed` flag. | **It does not.** P04 models resolution as `tokens.is_winner IS NOT NULL` and tradeability as `accepting_orders = 0`. Writing `is_winner = 0` for an unresolved market is worse than a schema error: it makes every open market a *loss* in the `smart_money` sample. NULL-until-resolved is the only legal value. |

Four process lessons, for whoever builds P06–P16:

1. **One freshness source per transport.** A REST poller that advances a socket's clock hides an outage: the
   composite (worst-of) keeps reading `ok`, the UI keeps promising live data, and the page that exists to catch
   the outage says nothing. My own chaos harness shipped exactly this and it cost a full 300-second run before I
   saw that check A was failing for a reason inside the test.
2. **A comparison over empty data must not pass.** "No duplicate alerts" with zero alerts fired, and "no missed
   large fills" against a reference that never reached the window, were both green in a run I reported. The gate
   now requires non-vacuity (`alerts > 0`, `live fills seen > 0`, `covered == total > 0`) and refuses the artifact
   otherwise. `--fast` runs may not contain deep-only measurements.
3. **A portable subset must never be silently smaller.** `build-sqlite-migrations.py` treats `ALTER TABLE` as
   Postgres-only and dropped every `ADD COLUMN`, so dev/CI ran a schema missing columns production had while
   every migration test passed. It now refuses with the lesson in the message. Prefer a phase-owned
   `CREATE TABLE` (0007) to an `ALTER` on someone else's table.
4. **Deriving a user-visible threshold from a re-rendered value is a bug factory.** The metadata version diff
   first compared a Gamma payload against our own stored translation and wrote a "market changed" row for every
   market on every pass (`5.0` vs `"5"`), burying the one row that mattered. `market_stats.meta_json` now stores
   the venue's tracked fields verbatim, and a diff is between two observations of the same source.

## 16. Corrections from the P06 build session (added 2026-09-18, nothing above deleted)

P06 landed in the platform repo at `9c1da25`: 31/31 gate checks, 500 tests, 7/7 chaos assertions against a real
`SIGKILL`, kill-switch drill 5/5 with worst refusal 460 ms against a 1000 ms budget, 26/26 mutants killed. Four
lessons for P07–P16, all of them bought by a probe that tried to make the answer wrong:

1. **A refusal is only evidence if it is attributable.** "0 orders placed while the switch was engaged" proved
   nothing until the same path was shown placing an order one second earlier, with the switch off, and the
   refusal *reason* named the switch. Chasing that requirement found two real product bugs (copy queued an
   intent whose Activity line read "copied" during a halt; automation had no view of the switch at all) and two
   fixture bugs — an un-armed rule (a rule must be run in dry mode before `enable` accepts it) and a refusal
   whose reason was `min interval: next allowed in 59319ms`, i.e. a per-rule throttle that had pre-empted the
   global stop. A global halt now outranks a per-rule cooldown in the evaluation order, exactly as it does in
   preflight (`kill_switch → … → user_limits`).
2. **Measure the propagation window, do not assert it away.** An out-of-process child re-reads the switch every
   `KILL_POLL_MS` (250 ms) and acts once per tick (100 ms), so work queued at the instant of engagement
   legitimately goes out. The drill therefore reports both numbers: latency to the *first attributable refusal*,
   and **zero** venue POSTs after `grace = poll + tick + slack`. A harness that demanded instant silence would
   have been fixed by weakening the product, which is the wrong direction.
3. **A generated twin needs a byte-for-byte comparison, not a subset one.** `c_schema_parity_and_append_only`
   asks that every source `CHECK` appear in `db/migrations-sqlite`, so it passed while the twin lagged my own
   migration edits and carried none of the P06 append-only triggers. Re-run `build-sqlite-migrations.py --write`
   after *every* migration change; P07 should make the check compare generator output to the committed file.
4. **A survived mutant is usually an indictment of the check.** Ten of 29 anchors did not exist in my first
   mutation harness (the anchor pre-flight turns that into a hard error rather than a smaller proof set), and the
   survivors said more about the gate than the product: the fee check asserted `floor` on an example with no
   remainder so "rounds up" was prose; the breaker check exercised only the consecutive-failure trip, never the
   50 % error-rate trip; the recovery check ran `tick(reconcile=False)`, which cannot see the
   reconcile-before-requeue ordering it claimed to protect. Two other survivors were genuinely *equivalent*
   mutants (a `min(share, observed)` line made "pay the estimate" inert; the tick-level guard is defence in depth
   behind the `enable()` guard the gate owns) — retarget those, never delete them.

## 17. The P07 security plane, and what its own tools caught (added 2026-09-18, nothing above deleted)

Built in `/home/user/polygm-platform` (`origin/main` = `2ccd7ed`). Measured, not asserted: `make p07` → 32/32 in
45 s; `make drill-p07` → 10,000 wrapped keys revoked (42 ms of our own half, 20 batch statements), 0 of 800
sessions surviving the global revocation, kill switch engaging with the queue sweep in the same statement;
`make gate-p07-mutate` → 28 planted weaknesses, 28 killed, 0 survived; suite 641 tests OK (141 of them P07);
`make p06` still 31/31; contract 176/176; lint 8/8 with 8/8 canaries; `ci-log-scan --sources` 133 files, 0
findings, `--self-test` 0 failures; `dependency-scan` pass.

**Four product bugs, all found by the phase's own checks rather than by review.**

1. `_principal` built its authorisation operation from the request URL instead of the matched route template, so
   every served route with an identifier in its path answered 500 `AUTHZ_UNDECLARED` to an *entitled* caller. The
   suite missed it because its fixtures authenticate with a dev header, which takes the other branch: ~600 green
   tests were not exercising the path the bug lived on. Anything that bypasses authentication in a fixture is now
   assumed to be blind to authorisation bugs — pair it with at least one real-session test.
2. The five P07 audit tables were append-only in the generated SQLite twin and fully editable in Postgres: the
   triggers had gone into `db/migrations-sqlite/_append_only.sql` by hand and never into `0009_security.sql`. A
   twin-only invariant is a test-suite-only invariant. `gate:c11` now asserts the declaration in the shipped
   migration *and* exercises each table with a row in it (an `UPDATE` against an empty table never reaches a
   trigger and reads as "mutable" — the vacuous-check shape bit twice this phase).
3. `totp.REQUIRED_FOR` named `address_remove` and no route asked for a code, while the contract already documented
   a 403 on that path. Declared-but-unenforced is the default failure mode of a policy table: the next phase that
   adds an action to a `REQUIRED_FOR`-style list should add the gate check that asks the route.
4. `redact` scrubbed 14 shapes but not a connection URI's password, while `ci-log-scan` refused builds for
   exactly that shape. Redactor and scanner are two independent matchers over the same threat; they drift, and the
   drift is invisible until you diff their coverage on purpose.

**Generator discipline, since §16 item 3 asked for it:** `tools/build-sqlite-migrations.py --write` must be re-run
after *every* migration edit and `--check` (wired as `make sql-sqlite-check`, part of `make check`) fails on
staleness. Note what it can and cannot do: it translates the append-only trigger block into the twin and treats
every other `CREATE TRIGGER` / `CREATE OR REPLACE FUNCTION` as PG-only. So a *conditional* trigger (e.g.
`polygm_withdrawal_hold_holds`, which refuses an UPDATE that shortens a withdrawal hold) exists in production only,
and the gate asserts its declaration in the file rather than pretending the twin carries it. Say so in the
migration comment — a reviewer who cannot find the invariant in the SQLite schema should find the sentence.

**Two rules that bind later phases.** A TOTP code is single-use inside its 30 s window, so two money actions are
one window apart by design; fixtures advance the app's clock by a window (`_advance_clock`, mirrored by
`Plane.advance_clock`) instead of sleeping or bypassing the route. And the admin surface deliberately answers 503
`SIGNER_UNAVAILABLE` when no admin token is *configured* (a misconfiguration must not look like an attack in the
dashboards the on-call reads) while a *wrong* token is 403 `ADMIN_REQUIRED` — do not "fix" the 503 into a 401.

**Where P07's honesty is load-bearing, and what it does not prove:** the 10,000-key revocation number is bounded
by the provider's revoke rate, not by our batch arithmetic (at 1 call per wallet ≈ 1 call/s that is ~2.8 hours),
and P13 owns measuring it; SIWE was deliberately not built (P10, nonce-bound typed payload); the geofence ruling
is "no app-layer block", written with its reasons and a counsel sign-off item; 14 `[UNVERIFIED]` markers each carry
a numbered launch-checklist bullet, which `gate:c30` pairs mechanically. The drill ends with eight numbered things
it does not prove — the part a passing run makes people skip, and the part a reviewer should read first.

## 18. The P08 shell, and the class of bug only a served response can show (added 2026-09-19, nothing above deleted)

P08 is done: `polygm-platform/web/` is the whole page layer — 19 routes (`/`, `/markets` as the anonymous price
surface, five `(auth)` screens, seven `(app)` screens, `/tma`, and `app/api/[...path]` as the authenticated proxy),
15 checks in `tools/p08-gate-check.py` with 11 canaries, `docs/P08-frontend-shell.md`, and the two recorded
artefacts (`P08-bundle.txt`, `P08-gate.txt`). `make p08` builds the web app, re-measures the first-load budget and
runs all 15; `make p08-offline` is the 14 that need no build, `make p08-selftest` proves the checks can fail. The
headline numbers: 15/15 and 11/11, 95 web tests in 15 files, first-load 187.6 KB on `/` against a 200 KB budget
(worst route 190.3 KB), 223 dictionary keys of which 181 are used, `schema.gen.ts` 1,875 lines from a contract that
passes 177/177.

**The one that matters.** `pgm_at` was written as a bare expiry by `src/auth/refresh.ts` and read as
`"<expiry>:<token>"` by `src/auth/server.ts`. Both halves had a green unit test. Every request after a rotation
therefore found no token and refreshed again, and the second refresh presents a *spent* single-use token, which
upstream reads as theft and answers by revoking the whole family: the user is logged out on every device because
four widgets mounted at once. The fix is not only the shape — it is that **the writer and the reader of a
two-ended contract now live in one file with a test that reads both ends**, and that the property is exercised
against `next start` + uvicorn (`gate:c11`), not against a `TestClient`. A related gap in the same module:
single-flight coalesced requests that arrived *together*, so a straggler that arrived a few milliseconds after the
winner re-presented the spent token; a settled rotation is adoptable for 2 s, and the cost (two seconds of
reuse-detection granularity) is written in the module rather than traded silently.

**What the phase's checks caught besides that** (`docs/P08-frontend-shell.md` §2.8 keeps the full 14): `formatCents`
validated with `| 0` and wrapped above $21.47 M; `i18n-check` matched ~135 of ~197 lookup sites and reported a clean
dictionary; `BillingClient` labelled *tape freshness* as "entitlement could not be read", and `PLANS` carried a
price no JSX rendered; a `var(--pgm-z-sticky)` that was never declared (a dropped custom property is legal CSS, so
the build said nothing) and a doc marker naming a test class that had never existed. Every one of those is a
control reading the wrong signal, and the answer was the same each time: keep the rule, fix the matcher, add the
planted violation that proves the matcher still bites.

**The checker was the buggiest component**, which is worth recording because it is not excused by being tooling: a
regex comment-stripper deleted `" https://telegram.org"` out of a CSP string and turned a correct config into a
"missing CSP" finding; `with HTTPConnection(...)` is not a context manager, so the boot poll raised `TypeError`,
swallowed it, and timed out after 75 s looking like a product failure; a pattern written inside a nested heredoc
landed as `r"\\s"` and silently matched nothing for a whole revision; fixed ports 8099/3112 collided with a
leftover process; `ci-log-scan.py --sources web/.next` returned `unrecognized arguments`, which a caller read as
exit 1 = "clean". Two rules for later phases: **every gate check ships with a canary that plants its own
violation**, and **a mode's wiring is itself tested** (`ci-log-scan --self-test` now runs `--built`/`--file` for
real over a temp tree, because a mode that never runs cannot fail).

**Cross-phase changes P09+ inherit.** (1) `tools/ci-log-scan.py` has a third mode, `--built PATH`, which applies
`SOURCE_RULES` to generated output — the log ruleset on minified core-js produced 13 findings and taught nobody
anything; if you want a stricter rule, add it to `SOURCE_RULES`, not to the invocation. (2) `tools/dependency-scan.py`
enforces exact pins, and it *found* P08's caret ranges, so every `web/package.json` dependency is pinned exactly
and P09's additions must be too (`npm install` will happily widen them; `make p07` will refuse). (3) P03's token
generator now emits `:root`-wrapped breakpoints, and `web/styles/{tokens.css,theme-colors.json}` are generated
mirrors checked by `tools/build-web-tokens.mjs --check`; `web/**/styles/**` and `*.test.*` are excluded from the
literal ban because they *define* the values. (4) The route ledger (`web/src/api/routes.ts`, 40 entries: 19 built,
21 refused-by-design), the contract, and the doc's launch list are triangulated by `gate:c1`; `globalSearch` is the
only `whileMissing: "hides"` route and it is gated by `ROUTES.globalSearch.built` in the command palette. Adding a
screen in P09–P12 means a ledger entry, dictionary keys, and a `[owner · test]` marker in that phase's doc — the
gate refuses the three when they disagree, including when the doc renumbers and the code does not.

**Deliberate declines, restated so they are not re-litigated:** no virtualisation (a 64-row tape cap, `TAPE_ROW_CAP`,
because a windowed list of financial rows that drops the row a user is reading is worse than a long list), `en`-only
with a key contract instead of a translation pipeline, two resizable rails with the centre as the remainder, flash
at 90/200 ms (the prompt's 600 ms was rejected as a distraction), `WidgetBoundary` per widget rather than per page,
and no `X-Frame-Options` anywhere (a per-origin CSP is the only correct frame policy for a page that is framed by
Telegram and by nobody else).

**What this phase did not and cannot prove here:** there is no browser in this workspace, so `tma-real-device`,
Lighthouse, the 60 fps rail-drag trace and Storybook are `[UNVERIFIED]` with numbered launch items (3 and 4 among
them), and `measure-first-load.mjs` reports *bytes fetched per document* — a payload claim, not a rendering one.
`git` state was damaged twice by environment snapshot rewinds (`.git` replaced with an older `main`, and
`web/node_modules` + `web/.next` deleted); the recovery that worked is `git fetch` + `git reset --mixed <tip>`
after confirming the remote tip via the API, and `npm ci` — never `--hard`, never `checkout .`.

**Standing, unchanged:** phases run strictly `P01 → P16`; no real funds move until P13 and P14 are green (kit rule
7), and P08's proxies therefore refuse money mutations through the same `whileMissing` ledger rather than wiring
them "for now".

## 19. The P09 markets surfaces, and the class of bug that only a rendered screen shows (added 2026-09-19, nothing above deleted)

P09 is done in two commits on `polygm-platform` (`d54d5e7` backend + contract, `0ad8830` web + doc + gate). The
product now has the three screens a prediction-market terminal is for: `/markets` discovery, `/market/[market_id]`
with the ladder and the chart, `/event/[event_id]` with 128 outcomes and the probability-sum invariant. The
endpoints behind them are `/v1/markets/{market_id}/history` and `/holders` and `/v1/events/{event_id}`, all in
the ledger as `built: true` under owner `P09`. Numbers: backend `make test` 674 OK, `check-openapi` 200/200,
`make lint` 92 files 0 findings; web 145 tests in 20 files, `tsc` clean, dictionary 345 keys 0 missing;
`make p08` re-run with P09 in the tree is **15/15** (worst route `/markets` 198.0 KB of a 200 KB budget,
route-level splitting proven — the landing document still does not fetch the money module);
`make p09` is **7/7** with 7/7 canaries, recorded in `docs/verification/P09-gate.txt`.

**The one that matters.** `kind="price"` in the number layer takes **tick units**; the ladder was feeding it
**micro-units**. On a 0.01-tick market that is a 100× error, and on the 0.001-tick market this phase exists for it
renders `.001` as **`1.000`** — a plausible price, ten times the size, on the market whose entire difficulty is its
tick. Both spellings of the value are integers, both pass every pure-function test, and the compiler has no opinion:
the bug lives in the *contract between* a pure function and a renderer, which is why the first screen test found it
within a minute of being written. The same mismatch put the size column out by 10^6 (232,978,723 shares rendered as
`232978.7B`), and a third instance multiplied an already-cents spread by 100 and printed a 0.2¢ spread as `$20.00`.
The permanent answer is `priceUnitsOf` / `shareUnitsOf` in `web/src/lib/depth.ts`, delegating the parse to
`src/money/cents.ts` so the app still has exactly one price parser — and the rule that **a screen test earns its
place by rendering**, because a bug in the seam between two tested units is invisible to both.

**The contract was the other seam.** `SuccessBody` unioned *every* documented response, so the body type of the book
was `{bids, asks, …} | {error: …}` — formally correct, useless in practice, and `book.bids` was a type error on a 200.
Narrowing it to 2xx exposed a second trap: `Extract<keyof R, \`2${string}\`>` matches neither `200` nor `"200"`, so
the "narrowing" silently produced `unknown` while every check stayed green. Then the body typed as `unknown`
revealed the real finding: the API has always served `cumShares` on every level and six fields on the market detail,
and `contracts/openapi.yaml` documented none of them. The fix was the contract plus `npm run gen:api`, not a local
interface — a field the ladder cannot work without is not optional documentation.

**What the phase's checks caught besides that** (`docs/P09-frontend-markets.md` §2.8 keeps the five): the p08 gate's
c8 was reading a bundle measurement produced by a server whose process was still listening on the measure port from
a previous run, so it compared two builds and reported neither; `app/markets/page.tsx` rendered an error paragraph
instead of the client component when the SSR read failed, which both dead-ended the reader and quietly removed the
markets payload from that route's measured cost; a hand-typed `BookPayload` was flagged by c2, so all four P09 body
types now come from the schema; c14 caught the same widget composed at two JSX sites (one per branch) — a retry path
and a boundary count are the same question about who composes what. Two gates' scanners exist for the money path and
for freshness, and P09's c3/c4 *call* `tools/p08-gate-check.py`'s functions over their own files rather than writing
a second opinion; a change to the rule changes both gates at once.

**The checker was, again, the buggiest component.** `p09-gate-check.py`'s canaries caught four of its own defects on
first run: a ledger reader using `\{([^}]*)\}` truncating at the `}` inside `/v1/markets/{market_id}/history`, so an
entry that *was* built and owned read as neither; the `[UNVERIFIED]` pairing check testing whether a slug appears
*anywhere in the document* (it always does — it is in the marker being checked) instead of in a numbered launch item;
comment-blind scans reporting `parseFloat` from a doc block that quotes the call it bans and `dangerouslySetInnerHTML`
from a comment saying it is never used; and a probe shelling out to `npx tsx`, which the repo does not have. Each is
the same failure as the ones above: a control reading the wrong signal, which is why the standing rule is that every
check ships with a canary that plants its own violation.

**Cross-phase changes P10+ inherit.** (1) `brand/tokens.json` now carries `rail_left`, `rail_right` and
`book_max_block`, emitted by `tools/build-tokens.mjs`; the two rail widths had been used since P08 against `auto`
fallbacks and **declared nowhere**, which is a layout that looks right for as long as nobody asks which two widths it
is laying out against. (2) `SuccessBody<Path, M>` in `web/src/api/types.ts` is the only way to name a response body;
if a field is missing from the implied type, the fix is the contract plus `npm run gen:api`. (3) The flash policy's
`source: "ws" | "rest"` parameter is the design-system-level answer to "should this move?" — the book polls REST
today and therefore never flashes, by policy, and the day `/v1/live` carries the book the ladder inherits rate
limiting, rounding-window suppression and the reduced-motion fallback by passing `ws`. (4) `MAX_ROWS = 400` on the
outcome table and `SUMMARY_AT = 5` on the discovery card are the two caps this phase chose over virtualisation;
`docs/P09-frontend-markets.md` §3 states what was not built and which launch item carries it.

**What this phase did not and cannot prove here:** there is still no browser, so Lighthouse/LCP on the three P09
routes, a 10 Hz frame trace of a 200-level ladder, virtualisation's necessity, touch behaviour on real hardware,
screen-reader output, `prefers-reduced-motion` and the Storybook states are `[UNVERIFIED]` with numbered launch items
1–9 in `docs/P09-frontend-markets.md` §4. The measured build is `next build --webpack`, not Turbopack: the sandbox has
~700 MB free and Turbopack's builder is OOM-killed there, so the bundler is part of the recorded measurement. The
`.git` rewind recurred a **fourth** time (tree current, `.git` at a P05-era tip, `origin` missing entirely); the same
recovery worked — token from `/home/user/.secrets/tokens.env`, `git remote add`, `fetch`, `git reset --mixed` — and
the rule stands: check the remote tip before claiming drift, never `--hard`, never `checkout .`.

**Standing, unchanged:** phases run strictly `P01 → P16`; no real funds move until P13 and P14 are green (kit rule
7). P09 shipped no money mutation: the ladder hands a price string to P08's ticket and nothing else.

## 20. P10 complete (the terminal, D1–D9): the engines that had to exist before the screens, and the four refusals that make them safe (added 2026-09-19, nothing above deleted)

P10 is done on `polygm-platform` in nine commits — D1–D7 (`32023ab` … `861663a`, closed with the 60fps measurement
in `92d6d4f`) and the two engines behind the last two screens in `76563e3`. The product now has the eight surfaces
the phase's own list named: `/terminal`, `/trader/[anon]`, `/whales`, `/radar`, `/portfolio`, `/copy`,
`/automation`, `/alerts`. Numbers at close: backend **822 tests OK**, `check-openapi` **365 passed / 0 failed**
over a 46-path / 50-operation contract, web **339 tests in 37 files** with `tsc` clean and the dictionary at
**825 keys, 782 used**; `next build --webpack` re-measured (**`/markets` 199.6 KB of the 200 KB budget**,
route-level splitting proven), and all three gates re-recorded against that build — **P08 15/15, P09 7/7, P10
15/15 with 12/12 canaries**, in `docs/verification/`.

**D8/D9 were read wrong once, and the correction is recorded rather than absorbed.** A phase-start reading put the
automation and alert engines in P11; `prompts/P10-frontend-terminal.md` lists them as D8 and D9, and
`prompts/P11-leaderboard.md` is the leaderboard, rankings and referrals. Deferring them would have left P10 with
two screens missing and P11 with two deliverables it never asked for. The reconciliation is `docs/P10-frontend-terminal.md`
§5.1, so the next phase does not re-derive it from chat.

**The four refusals, because they are the feature.** (1) A rule is saved as a dry run and there is no field that
makes it live: `POST /v1/automations` writes `enabled = 0`, and arming is a second endpoint that refuses without a
`dry_run_completed_ms` the engine itself wrote (`DRY_RUN_REQUIRED`, with the next step named) — dry-run is a *path*
through an evaluation, not a checkbox. (2) While the daily-loss halt stands, every money-moving path refuses
(`HALTED`) and every rule reads `halted` even when its `enabled` flag is 1, because halted outranks enabled and
calling that "active" hides the one state D8 asks to be loud about. (3) A channel the plan does not cover is
refused at save time with the plan named (402 `PLAN_REQUIRED`), not at fire time. (4) The 5-minute crypto entry
template is *listed* with the fee arithmetic that withholds it (`available: false`, `blocking_reason:
no_measured_edge`) rather than dropped — a catalog that hid it would make a user guess whether the feature is
missing or the maths said no, and the second is worth reading. Pausing is always allowed: the safe direction must
never sit behind a precondition.

**The bug the gate's own fixture found.** Saving the smallest legal rule — the builder payload minus its loss
ceiling — returned `INTERNAL`. `automation_rule_policy` has `CHECK (kind = 'auto_redeem' OR max_loss_micro > 0)`,
so the constraint arrived as a 500 with nothing about the missing field, and the *client* was the stricter of the
two (the form already refused a zero ceiling). A database CHECK surfacing as a service fault is a missing
validation: the write now answers 422 naming `maxLossMicro`, and the regression test asserts both directions
(refused for an exit rule, saved for `auto_redeem`, which cannot add risk because it cannot take a position).
Two uses of one lesson: **the server is the authority, the form is a convenience, and when they disagree the
server is the one that is wrong.**

**The gate that reported a false FAIL.** Two D9 paths were compared against the contract all along, but keyed by
verb (`("POST", "/v1/alerts")`) because a GET and a POST on one path answer different status sets; the gate's
membership test only recognised the bare-key spelling and reported them unguarded. A gate that cries wolf gets
edited around, which is worse than no gate, so the test now accepts both spellings and says why in a comment.
Separately, the staleness rule that caught the D8/D9 bundle (`the measurement is older than the newest source
file`) fired again the moment render tests were added — the rule is mtime-based over `web/src`, deliberately, and
the answer is to rebuild and re-measure, not to narrow the rule.

**What a render test earns its place for, again.** The alerts list and the alerts settings *write* disagreed about
the settings shape — the list returned the raw row, the write returned the row plus `quietHours`/`digestNow`, and
the screen reads `settings.quietHours.note` on both. Every pure function on both sides was correct and green; the
seam was the fault. One `_settings_view(uid, at)` now serves both reads, with a regression test named after the
screen's read. The same stretch produced the money-path copy of P09's class of bug: a probability rendered through
the cents helpers (`formatCents(microToCents(620000))` → `0.0062`), because cents are hundredths of a dollar and
0.62 is not money. Prices now have their own integer path (`priceMicroText` / `priceMicroFromText`, `null` on
garbage), and the rule is **one parser per unit — money through the money layer, probabilities through the price
layer, never one through the other.**

**The screen is the engine's vocabulary, not a transcription of it.** The builder's trigger and action kinds,
the AND/OR joiner map and the engine's limits (`maxLeaves`, `minIntervalNs`-class constants) are read from the
API's own vocabulary read, which reads the engine; a form that offered a kind the engine refuses would be a form
that saves rules which can never fire. Concretely: `cancel_open` is not offered an "all" scope because the engine
refuses that scope, and the ninth row is refused with the engine's own limit before anything is sent.

**What P11 inherits, deliberately.** `prompts/P11-leaderboard.md` keeps its six boards, its integrity rules, its
referral dedupe and its SSR pages unchanged. Two things built here are already load-bearing for it: the gate's
`win_rate_findings` / `threshold_findings` scanners (a board that prints a win rate under its sample gate, or a
badge with no visible rule, fails the same scanners P11 will be held to), and the radar's row shape — wallet,
matched markets, bought/sold, realised PnL, win rate with its gate, classification with its rule — which is the
row a leaderboard needs and should be extended rather than re-derived.

**The `.git` rewind recurred (fifth time)**, in the *spec* repo this time: tree current, `.git` sitting at the
P05-era tip `d9688c2`, no remote configured. The same recovery worked and is unchanged: token from
`/home/user/.secrets/tokens.env`, `git remote add`, `fetch`, `git branch backup-pre-reset HEAD`, `git reset
--mixed origin/main`, confirm the tree is clean against the tip, then commit. **The rule stands: check the remote
tip before claiming drift; never `--hard`, never `checkout .`.**

**Standing, unchanged:** phases run strictly `P01 → P16`; no real funds move until P13 and P14 are green (kit rule
7). Nothing in P10 moves money: `/automation` and `/alerts` write rules, and no loop fires them without a dry run
the engine recorded. The next phase is the kit's P11 — the leaderboard, rankings and referrals.


## 21. P11's first three deliverables: the boards, the ranking API, and the screen that reads a standing (added 2026-09-19, nothing above deleted)

`prompts/P11-leaderboard.md` is being built in order. **D1** (specification, integrity rules, pure ranking engine)
and **D2** (the rankings API, plus the seeded population the boards are demonstrated on) are pushed, and **D3**
(the standing a wallet can read back — badge, percentile, the gap to the place above, the 30-day sparkline, a
comparison of up to three wallets, and follows) is built and green. D4–D7 remain: self-rank and privacy,
referrals, the public SSR pages, the anti-gaming dashboard.

**Six boards, one formula, and the default says what it is.** `risk_adjusted` is the default and is *stated*, not
implied: trimmed net realised divided by `max(largest drawdown, 2σ)`, integers only, the best market excluded only
when it is profitable, and the lucky-gambler share clamped to 10000 bps. Sorting is by the field the board is
named after — the heading is a claim about the order, and a `win_rate` board ordered by `scoreBps` was a real bug
this phase (pinned now by a canary and a rank test). Tie-breaks run score → settled → drawdown → wallet id, so
the order is total and a recompute cannot shuffle equals.

**The phase's acceptance sentence is an engine property, not a screenshot.** Rank 47 sits above nothing and below
rank 12 at the gate pair: `w_32371216f9` (rank 12) has **48 settled markets and 93,333 bps of score**,
`w_a945dde868` (rank 47) has **26 settled and 5,000 bps** — the smaller sample is *below*, and `/why` says so in
one sentence, because sample size decides eligibility and never order. `/rank` then expresses the distance in the
board's own unit (`625 bps behind …, 5626 bps would pass them`), which is why the endpoint carries
`orderField`/`orderUnits` per board rather than a fifth vocabulary of "points".

**Three refusals D3 adds to the API's contract.** (1) A comparison is **one** read of **one** board: three
separate `/rank` calls assembled by the client can disagree about window and recompute instant, so
`/v1/leaderboard/compare` answers once and takes its pairwise sentences from the engine, with `verdict` computed
from the ordered array rather than from `rows[0]`. (2) A non-pseudonym is refused **before** anything is echoed —
`anons=0x…` is a 422 naming the field, and the response never carries an address back (the gate's c16 canary
plants exactly this; an earlier build echoed it, which is how the canary earned its place). (3) A **follow is a
watch, not a copy config**: `trader_follows` is keyed by pseudonym, idempotent per `Idempotency-Key`, unfollow
reports whether a row existed, and the tests assert `/v1/copy/configs` is untouched. Following is a read; copying
is money and stays in P10's D7.

**2.9 KB of prose was on every phone's first load, and the budget caught it.** D3's four routes took `/markets`
to **200.7 KB against the P08 200 KB budget** — a fresh, post-build measurement, so the failure was the number and
not a stale artefact. The cause was not the new screen: `web/src/api/routes.ts` is imported by the API client, so
every key, path, flag *and explanatory note* in the ledger is fetched by every signed-in document, and the notes —
prose nothing renders — were 2.9 KB of that. They now live in `src/api/route-notes.ts`, which no screen imports,
and the route measures **199.2 KB**. What makes that a fix rather than a trick is the pair holding it: the P08
gate's c1 (extended here, with a new `c1_notes` canary) and `web/src/api/route-notes.test.ts` both fail if a note
outlives its route or if a `note` field reappears on a `RouteDecl`. Coverage is deliberately *not* checked — the
launch list in `docs/P08-frontend-shell.md` §4 already explains every unbuilt route. **The lesson for every later
phase: the ledger is on the wire, so anything added to it is a byte a phone pays for.**

**Numbers at this point.** Backend **911 tests OK** (62.2 s), `check-openapi` **431 passed / 0 failed** over 57
paths, `tools/p11-gate-check.py` **16/16 with 10/10 scanners canaried** (c3 is the acceptance sentence walked over
the API, c15 the standing a wallet reads back, c16 the comparison that must not echo an address), web **366 tests
in 40 files** with `tsc` clean and the dictionary at **857 keys, 814 used**; `npm run build` renders 19 routes
including `/leaderboard`, `npm run measure` passes at **199.2 KB worst route**, and `tools/p08-gate-check.py` is
**15/15** — c4's money scan included, which is why the percentile, the best-trade share and the win rate all render
through `bpsText` instead of `.toFixed`.

**The `/snapshots` duplication D2 left behind is closed.** `_lb_history` claimed to be shared by `/snapshots` and
`/rank` while `/snapshots` still folded the same rows inline; the route now calls the shared fold and keeps only
what is specific to it (the pseudonym check and its freshness stamp). Two folds of one set of rows is how a
sparkline and a badge end up disagreeing about the same wallet on the same screen.

**The `.git` rewind recurred again (sixth time), in the spec repo.** Local HEAD sat at `d9688c2` (P05 era) with no
remote configured, while GitHub's `main` was `87cdc8a` — and the working tree already held §16–§20, so nothing was
lost by the recovery that worked before: token remote, `fetch`, `git branch backup-pre-reset-1`, `git reset
--mixed origin/main`, then confirm the tree is byte-identical to the tip. **Standing rule, restated for the third
time: read the remote tip before claiming drift; never `--hard`, never `checkout .`.**

**Standing, unchanged:** phases run strictly `P01 → P16`; no real funds move until P13 and P14 are green (kit rule
7). Next: P11 D4–D7.

## 22. P11 D4: your own row on every board, and the setting that decides who may tie it to you (added 2026-09-19, nothing above deleted)

D4 is pushed (`2299c75`). The reader now sees their own standing on all nine boards (six boards, four category
boards), gets that row **pinned when it is off the page** with the gap that would move it, is told what to do when
the gate refused the wallet, and decides whether that row carries an account handle.

**The kit's sentence cannot be taken literally, and the resolution is the deliverable.** "Appear on public
leaderboards, or stay private" collides with the integrity rule §21 already fixed: a board that drops whoever asks
not to be listed is a board that reports a flattering field, and blown-up accounts are shown rather than hidden.
D4 therefore splits the sentence: **inclusion is not optional, identity is, and it defaults to off.** A private
account is still ranked on every board it qualifies for; what "private" withholds is the *link* — no dossier
resolves, no name finds the account, nothing in any public payload connects `w_…` to `u-…`. The screen says the
half a user assumes wrongly, verbatim: "private removes the link to this account, not the row". The consent write
is an append-only `audit_log` row (action `leaderboard.identity`) carrying the previous state, so "I never agreed
to that" has an answer that is not a shrug.

**The privacy scanner is worth more than the feature.** `privacy_findings` walks every public payload the phase
can produce and fails the gate if a withdrawn handle appears in any of them; it is canaried in both directions.
Writing it forced two decisions: the account's own `/me` and `/identity` **keep** the handle on opt-out ("kept,
not published") while every published row loses it — the first run of c18 failed on exactly that, which is what a
scanner is for — and opt-out **keeps the row and the rank**, which c18 asserts, because that is what the copy
promises.

**Two bugs found on the way out of D3.** (1) `newIdempotencyKey` in `web/src/api/client.ts` joined its scope and
its random half with a **colon**, which `_IDEM_RE` (`^[A-Za-z0-9_-]{8,128}$`) refuses: every mutating request
without a caller-supplied key was answered with a 422 about a header the client had just generated itself. No
screen showed it, because the exercised screens pass their own key — it surfaced only because D4 added a mutation
nobody passes a key for. The shape is now asserted against the server's own regex in `client.test.ts` and by a new
**P08 c16** (contract pattern, `_IDEM_RE`, and the client's producer must be one rule), canaried with the
colon-joined key. (2) The D3 board panel shipped class names and **no stylesheet rules** — the one way a screen can
be finished and still look broken; D4 added the rules for both the board and the strip, and the strip's z-index uses
the design system's existing sticky rung rather than a literal (P08 c5).

**Numbers at this point.** Backend **927 tests OK** (39.9 s), `check-openapi` **450/0**, `tools/p11-gate-check.py`
**18/18 with 12/12 scanners canaried** (16.2 s; c17 walks the self-rank, c18 the identity), `tools/p08-gate-check.py`
**16/16** (13/13 canaries) and `tools/p09-gate-check.py` **7/7**, web **384 tests in 42 files** with `tsc` clean,
`npm run measure` passing at **199.3 KB** worst route, and `tools/build-sqlite-migrations.py --check` clean at
**107 tables / 54 triggers**.

**Standing, unchanged:** phases run strictly `P01 → P16`; no real funds move until P13 and P14 are green (kit rule
7); the ledger in `web/src/api/routes.ts` is on the wire, so anything added to it is a byte every phone pays for.
Next: P11 D5 — referrals (reward on the referee's first matched order over a notional threshold, dedupe, clawback,
hard self-referral block).

## 23. P11 D5: referrals — what a referral is worth, and the four ways it is stopped (added 2026-09-20, nothing above deleted)

D5 is built and green (uncommitted at the time of writing; it lands as the D5 close commit). The kit asked for a
referral link and a short code, **one** reward model chosen and justified, Sybil defences, a repeal path, a
dashboard, payout terms, and an argument about a referrer leaderboard. The two decisions below are the ones a later
phase has to live with.

**The reward is a share of the builder fee we are actually paid — not a bounty, not a deposit reward, not Pro
credit.** `SHARE_BPS` (25%) of `fee_micro_observed` on the referee's own attributable fills, for a year
(`TERM_DAYS`) from the referee's qualifying order. The property that decided it is that **the model has no fixed
cost, so a farm has no equilibrium**: a flat bounty pays the moment a stranger crosses a threshold, and
manufacturing a stranger — one small matched order — can cost less than the bounty, whereas earning $X from a fee
share requires causing about $4X of real builder fees to be paid to us out of the attacker's own money. *The
attacker is the customer.* Deposits are invisible to it, which matters because the kit names deposit-size rewards
as its trap ("deposit, withdraw, and never trade … it looks like a pyramid"), and nothing is ever paid for
recruiting — no second level, so "recruit recruiters" has no payout behind it. The two honest costs are recorded
with the model: the liability has a **tail** (hence the 30-day settle hold, the $20 minimum and the $2,000
referrer-month review) and it **pays slowly at the bottom** (one small trader earns cents in month one, which is
the point — the dashboard names the amount still to go instead of hiding it behind a pending that never clears).
The two rejected models are answered **in the served artefact**, `terms.rejected_models`, not only in the phase
document: an argument that lives only in a doc stops being made the day somebody changes the code.

**The qualifying event is a matched order, and the term runs from it — not from signup.** `terms.qualifies()`
refuses an unfilled order (a share of zero is zero), refuses a self-crossing order *structurally* rather than by
threshold (a "ignore round trips under $1" rule is a rate card for wash trading), inherits the market exclusions
the integrity rules already applied, and requires `$25` notional. Signup is free and unbounded; the schema
enforces `qualify_ms >= signed_up_ms`, which is what caught the gate's own fixture stamping the qualifying order a
millisecond *before* the signup it was supposed to follow.

**Two arithmetic rules that decide whether the number is real.** *Earned is observed, never expected*: accrual
reads `fee_micro_observed`, so when the venue settled $8 against our $40 estimate the referral is paid 25% of
$8 — the gate's c19 prints exactly that, and an accrual on expected fees is money leaving the door the first
month the venue charges less than we assumed. *`toMinimumMicro` is the gap from the **payable** balance*: $10
accrued but still inside the 30-day hold means $20 to go, not $10, because money that cannot be paid this cycle
does not count toward a payout minimum. The engine was right and the first test was wrong; the published rule is
"carried forward, never forfeited".

**The Sybil rules are an order, not a bag, and refusal is not review.** `sybil.PRECEDENCE = self_referral →
duplicate_funding → shared_device_or_ip → velocity`; the first rule that fires decides, so a self-referral is
never quietly downgraded to a device collision. `duplicate_funding` is a **refusal** — multiple wallets funded
from one source are one person, and a refusal means no fee will ever accrue — while `shared_device_or_ip` and
velocity (5/hour, 25/day) are **reviews**: held for a person, nothing accrues while it clears, nothing is
forfeited. Collapsing the two would either refuse honest referees who share a laptop or let a farm keep accruing
through a weeks-long queue. Signals are stored only as salted digests (`d_…`/`i_…`/`f_…`, never an IP, a user
agent or a funding address), a salt under 16 characters is refused, and a click row carries no user at all — the
attribution is the only place a person and a link are joined. What none of this catches is an attacker who
manufactures distinct devices, distinct funding sources and real fees paid: they are the customer the model was
built for.

**A self-referral is a revenue-integrity matter, and the builder code is the ground.** The apply route refuses the
attribution (409 `SELF_REFERRAL`) **and** disables the builder code with the reason in its note — the kit's
argument is that the builder-fee share is a revenue line and an account paying itself is taking it. The scoping
was forced by a bug in the first implementation: the app revoked the code for *any* refusal, but one code is
shared by all of a referrer's referrals, so a blanket revocation on a duplicate-funding clawback would have killed
attribution for every legitimate referee that referrer ever brought. The rule is now `kind == "self_referral"` and
nothing else: a funding collision costs the accruals, never the code.

**A clawback reverses money and keeps the history.** Unpaid accruals are cancelled first, what was already paid is
reported with `requiresRepayment`, anything under $5 is written off (a *published* rule, so a reversal is never a
surprise), and the accrual rows are not deleted — they are append-only, and the reversal is a state plus a record.
The dashboard zeroes the earned cell for a clawed-back referee rather than showing money that has been reversed.

**The funnel is the money chain; clicks are not a ceiling on it.** `FUNNEL = signups → funded → trading → earned`
is asserted monotone (`funnel_findings`), because each number comes from a different table and a join that counts a
row twice shows up as an impossible funnel rather than as a plausible figure nobody re-derives. `LEADING =
clicks` is checked only for what a counter can be wrong about alone; "clicks ≥ signups" is a requirement with a
wrong answer (one person can click five times, and a token can be forwarded), so the web panel renders the leading
row **apart** from the chain rather than quietly treating an honest non-monotonicity as a bug or dropping the row.

**No public referrer leaderboard, and the reason is served.** `GET /v1/referrals/terms` (public, no session)
answers it: "a public contest over recruitment is a spam contest with a scoreboard, and the ranking it would print
is a ranking of recruiting, not of trading." The gate asserts the route's absence, because the way that argument
loses is not somebody disagreeing with it — it is somebody adding the route in a later phase and leaving the
sentence behind.

**Web.** `web/src/terminal/referrals.ts` (pure) + `ReferralsView.tsx` on `/referrals`, rendering the seven rules,
the per-state sentences and the funnel **from the API**. The panel's only local strings are labels, and they are
literal `t()` calls in `Record<EarningsKey, string>`/`Record<PayoutKey, string>` over unions declared in the pure
module — because `scripts/i18n-check.mjs` refuses interpolated keys by design, and the fix is to make a new bucket
without a label a **compile error** rather than to silence the checker. The claim re-reads `/me` instead of
trusting the POST's echo, and exactly one POST carries one well-formed idempotency key.

**Two build-time lessons worth carrying.** (1) The referral plane's `0016_referrals.sql` **drops** P04's
never-written `referrals`/`referral_events`: `share_bps` was a per-code negotiated rate, which is exactly the
thing the model's arithmetic cannot survive (an account manager can raise it), and `referral_events` could not
answer "who is owed what, and is this referral still inside its year". A second referral schema beside the real
one is the two-authorities-over-one-number failure this project keeps writing gates about. (2) **A green test line
is not evidence when the HTTP layer is missing**: `python3 -m unittest discover -s tests` prints `OK` with the API
tests *skipped* on a machine where `fastapi` is absent, and this environment reinstals dependencies per session —
the D5 sweep read `Ran 990 tests … OK` that had skips in it, on a run where the gates were simultaneously failing
with `ModuleNotFoundError`. Read the skip count, not just the word OK.

**Numbers at this point.** Backend **990 tests OK** (83.0 s, no skips: `test_referrals.py` 37, `test_referrals_api.py`
26, `test_leaderboard_api.py` 65, `test_migrations.py` 18); `check-openapi` **498/0** (64 paths, 97 schemas, 7 new
referral operations, 9 new schemas, 4 new error codes); `tools/p11-gate-check.py` **22/22 with 16/16 scanners
canaried** (17.3 s; c19–c22 walk the reward model, the second wallet, the dashboard arithmetic and the payout
reality), `tools/p08-gate-check.py` **16/16**, `tools/p09-gate-check.py` **7/7**, `tools/p10-gate-check.py`
**15/15** — all four re-recorded on the D5 tree, because P08's c2/c7/c8/c15 read the contract, the built output,
the bundle artefact and the web suite; web **397 tests in 44 files** with `tsc` clean and `i18n-check` ok (919
keys, 876 used), `npm run measure` passing at **199.3 KB** worst route, `npm run measure:tape` passing at 1.78 ms
per second of load against the 16.7 ms frame, and `tools/build-sqlite-migrations.py --check` clean at **113
tables / 54 triggers / 16 files** with 48 PG-only drops recorded in `DROPPED.json`.

**Open, carried forward.** The `REVOKE`/grant block in `0005_triggers.sql` does not yet name `referral_accruals`,
so the append-only guard on the new ledger is the trigger and not the grant; it is a two-array edit and it lands
with D6's migration rather than being smuggled into D5 after its record is written.

**Standing, unchanged:** phases run strictly `P01 → P16`; no real funds move until P13 and P14 are green (kit rule
7); every classification label has a visible rule and a disclaimer; nothing hides a loss. Next: P11 D6 — the
public SSR pages (`/trader/<handle>`, `/market/<slug>`, `/leaderboard/<board>`) with OG tags, then D7's
anti-gaming dashboard.
