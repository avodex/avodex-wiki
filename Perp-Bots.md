# Perp Bots

> **Status: design.** Forward-looking architecture for Avodex perpetual futures. See [Perpetual Futures](Perpetuals.md) for the overview, and [Grid Trading](Grid-Trading.md) / [DCA Bot](DCA-Bot.md) for the spot bots these build on.

Avodex's [grid](Grid-Trading.md) and [DCA](DCA-Bot.md) bots trade spot today. On perpetuals the same bot engine carries over — the lifecycle, the just-in-time funding model, idempotent execution, and reconciliation are all reused — but one deep spot assumption has to be replaced: a spot bot **holds the base coin**, while a perp bot **holds a position backed by margin.**

## What changes from spot to perp

| Spot bot | Perp bot |
|---|---|
| Buys and **holds the coin** | Opens a **position**; no coin is held |
| Closing = **sell the coin** | Closing = **reduce-only** close of the position |
| Tracks two assets (quote + base) | Tracks **one collateral** (USDT) + a position |
| No funding | Pays/earns [funding](Perp-Funding.md), reflected in PnL |
| No leverage | Leverage sets margin = notional ÷ leverage |
| Worst case: **stuck holding** the coin | Worst case: **liquidation** — margin can go to zero |

That last row is the important one. A spot grid that goes against you leaves you holding the asset; a leveraged perp bot can be **liquidated**, so every perp bot carries a new safety concern that no spot bot has (see [Liquidation guard](#liquidation-guard)).

## Per-bot isolation: the prerequisite

Spot bots all trade through a single shared (omnibus) engine account; the service tracks each bot's share in its own ledger. That works for spot because balances simply add up. It **breaks for perp**: if bot A is long and bot B is short, their positions net to zero inside one shared account — there's no per-bot margin, no per-bot liquidation price, and funding can't be attributed to either bot.

The fix is to give **each perp bot its own engine sub-account.** Avodex already derives per-user sub-accounts deterministically (a keyed, one-way derivation that produces a stable, collision-resistant, **non-signable** account identity); perp bots extend the same mechanism to derive one isolated account per bot. The engine then handles each bot's **isolated margin and liquidation natively**, and:

- **Funding attributes itself for free.** Each bot's funding lands directly in its own sub-account — no need to split a shared payment by fluctuating position share, which is nearly impossible to do correctly under an omnibus model.
- **Liquidation is isolated.** One bot blowing up cannot touch another bot or another user.

Margin moves in two legs at open (from the user's main account into the bot's isolated sub-account) and back out at close; the USDT return follows the same settlement timing as spot withdrawals.

## Cross-system reconciliation

With positions living in the engine, the engine becomes the **source of truth** and the bot service keeps a **mirror** for display and accounting. A reconciler continuously checks, per bot, that the mirrored margin, position size, and entry price match the engine — and corrects the mirror to the engine if they ever drift.

Liquidation is **engine-driven and asynchronous**, so the bot service learns about it after the fact. It detects liquidation two ways — by subscribing to the engine's events **and** by periodic reconciliation as a backstop — and moves a liquidated bot to a terminal state. The double path matters: missing a liquidation would leave a bot showing exposure it no longer has.

## Bot types

### Perp grid
The most natural perp bot. A ladder of orders accumulates exposure, but each fill changes a **position** instead of holdings. Perps unlock variants spot can't offer:

- **Long grid** — accumulate a long across a range.
- **Short grid** — accumulate a short (impossible on spot).
- **Neutral grid** — market-neutral around the mark, with leverage improving capital efficiency.

Sell orders are reduce-only or open-short rather than "sell the coin held," and there's no need to pre-buy inventory — the grid opens the position itself. **Funding is shown honestly in PnL**: a long-biased grid in a market trading at a premium pays funding continuously, so the "grid profit" is reported net of it.

### Perp DCA
Recurring entry into a position. The just-in-time model carries over — each run posts margin instead of paying full notional. Because leverage and the "average in and hold" intent of DCA are in tension, the default is **unleveraged**, with leverage gated behind clear risk disclosure.

### Funding-rate arbitrage
A perp-native, market-neutral strategy with no spot equivalent: hold the asset on spot **and** an offsetting short on the perp, so price moves cancel while the position **collects funding** when it's positive. Because Avodex runs both spot and perp on the same engine, both legs execute in-house. The bot manages the two legs together and unwinds when funding turns or the basis converges.

### Position protection
The engine's mark-driven [conditional orders](Perpetuals.md) act as automatic stop-loss / take-profit. Rather than a separate product, these are offered as built-in protection on any perp bot.

## Liquidation guard

Because a perp bot can be liquidated, each one continuously watches its own liquidation distance. As the position approaches the threshold the bot acts **before** the engine has to — alerting, optionally topping up margin, reducing the position, or stopping — because an engine liquidation (with its penalty) is the worst outcome for the user. This guard is a first-class part of a perp bot's lifecycle and a precondition for offering leveraged bots to everyday users. The engine-side [liquidation](Perp-Liquidation.md) remains the backstop that guarantees correctness even when the guard is reached.

See also: [Perpetual Futures](Perpetuals.md) · [Perp Funding](Perp-Funding.md) · [Perp Liquidation & Insurance Fund](Perp-Liquidation.md) · [Grid Trading](Grid-Trading.md) · [DCA Bot](DCA-Bot.md).
