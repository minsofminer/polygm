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
