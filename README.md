# polygm — the complete build kit

Everything an AI agent or a contracted dev team needs to build **a gmgn-style
non-custodial trading terminal for Polymarket, with a Telegram bot**, branded with an
original identity inspired by Polymarket's visual language.

> This repo is the product of a researched plan. The business case, verified market
> numbers, and revenue reality live in `00-SHARED-CONTEXT.md`. **Read it first.**

## What's inside

| Path | Purpose |
|---|---|
| `00-SHARED-CONTEXT.md` | Paste into every AI session. Verified API facts, rate limits, fee tables, brand tokens, competitor data |
| `prompts/` | 16 sequential build prompts (research → branding → design → backend → data → trading → security → frontend ×4 → bot → testing → sec-testing → deploy → growth), each with role, deliverables, constraints, quality gate. `ALL-PROMPTS.md` = the combined single file |
| `PLAN.md` | The original business plan with live-verified numbers |
| `polygm/` | Working reference prototype: zero-dependency Python backend pulling live Polymarket APIs + mobile Mini App UI |
| `brand/` | The brand kit: Brand Lock (higgsfield-brandkit shape), canonical SVG mark + variants, app icon, avatar, OG image, brandboard, logo candidates, tokens |
| `skills/` | Vendored MIT skill libraries that the prompts' output should be built with |
| `SKILLS.md` | How to use the vendored skills here, including the built-in-generator adapter |
| `AGENTS.md` | Read this if you are an AI agent |

## Order of work

1. `00-SHARED-CONTEXT.md` (everyone, always)
2. `prompts/P01` → `P02` → `P03` (brand + design) — use `brand/` as the approved starting point
3. `P04` → `P05` → `P08` → `P09` (read-only product live)
4. `P07` → `P06` → `P12` (trading + bot)
5. `P13` → `P14` → `P15` (harden, ship)
6. `P10` → `P11` → `P16` (differentiate, grow)

## Legal posture

- Non-custodial ⇒ not an exchange; still treat keys like custody.
- We rebuild gmgn's *information architecture* as original work. We do **not** copy
  gmgn's or Polymarket's logos, icons, copy, or assets.
- Subscriptions inside Telegram must use Telegram Stars; on the web, Stripe.
- Geofence US users out of trading on the international CLOB.

## Licences in this repo

- Our documents, prompts, prototype code, and brand assets: all rights reserved by the repo owner unless stated otherwise.
- `skills/higgsfield-ai-skills/` — MIT, © Higgsfield AI.
- `skills/emilkowalski-skills/` — MIT, © Emil Kowalski.
