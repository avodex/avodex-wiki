# Perp Risk Engine — Limits, Cross-Margin & Auto-Deleveraging

> **Status: design.** Forward-looking architecture for Avodex perpetual futures, beyond the first iteration. See [Perpetual Futures](Perpetuals.md) and [Perp Liquidation](Perp-Liquidation.md).

Beyond the first iteration (isolated margin, single-position liquidation, insurance-fund backstop), three layers harden the perp risk model. They are **orthogonal and interlocking**: one prevents, one improves capital efficiency, one backstops.

```
prevent ───────────────► happen ───────────────► backstop
risk limits / tiers      liquidation              insurance fund ──► auto-deleverage
```

## Risk limits / leverage tiers (prevention)

Maximum leverage **decreases as a position grows.** Equivalently, the maintenance-margin requirement rises with position size — small positions can use high leverage, large positions cannot.

| Position size | Max leverage | Maintenance margin |
|---|---|---|
| small | high | low |
| larger | lower | higher |
| very large | lowest (toward 1×) | highest |

*(Illustrative — the actual tiers are tuned per market to its volatility and liquidity.)*

This is the root-cause defence against systemic risk. A forced liquidation of one enormous over-leveraged position is exactly what drains insurance funds and triggers cascades. Because a large position carries a higher maintenance requirement, its liquidation triggers **earlier** — leaving the engine a thicker cushion to close it before it goes underwater. Big positions are made structurally safer at the source. The tiers are static configuration, so they add no non-determinism.

## Cross-margin (capital efficiency)

The first iteration uses **isolated margin**: each position walls off its own collateral, and its loss can never exceed that collateral. **Cross-margin** instead lets an account's whole USDT balance back all its positions — unrealized profit on one position can support another, locking less capital — at the cost that one bad position can draw down the whole account. Liquidation becomes **account-level**: it triggers when total account equity falls below the sum of its positions' maintenance margins, and reduces positions until the account is healthy again.

### Cross-margin never crosses a bot's boundary

Avodex isolates every [trading bot](Perp-Bots.md) into its own engine sub-account specifically so one bot blowing up can't touch anything else. Cross-margin must respect that: it pools collateral **only within a single account**. A user's own manual positions can share margin with each other; a bot's collateral is **never** pooled with the user's wallet or with other bots. Otherwise one bot's liquidation could drain the user's balance — destroying the very isolation the sub-account model exists to provide. In short: **cross-margin is a feature of a user's manual trading; bots stay isolated by their account boundary.**

## Auto-deleveraging (final backstop)

Auto-deleveraging (ADL) is the last resort, reached only when a position closes past bankruptcy **and** the [insurance fund](Perp-Liquidation.md#insurance-fund) is itself exhausted. The underwater position still has to close, but there is no margin and no fund left — so the engine closes **opposing, profitable positions** at the bankruptcy price to absorb the loss.

- **Ranking.** Positions are deleveraged most-profitable-and-most-leveraged first — the traders who gained most from the move, using the most leverage, give back first. The ranking is computed entirely from the position ledger, needs no external price, and so is fully deterministic.
- **The ADL indicator.** Every trader sees a meter showing where they sit in the deleveraging queue, so the risk of being force-closed is **visible in advance.** This is a product commitment: a trader can be closed even when their direction was right, and must be able to see that risk coming.
- **What it costs.** A deleveraged position is closed at the bankruptcy price rather than the market price, giving up some expected profit. This turns an otherwise socialised loss into a *targeted, ranked, and foreseeable* one — and it leaves no bad debt behind.

## How the layers interlock

- Tighter **risk limits** mean liquidations trigger earlier with more cushion → fewer positions go past bankruptcy → the insurance fund is drained more slowly → ADL is reached less often.
- **Cross-margin** raises cascade risk (an account-level liquidation can close several positions at once), which makes risk limits matter *more* under cross-margin.
- **ADL** sits downstream of the insurance fund, which is fed by liquidation penalties — so the better the prevention works, the rarer ADL becomes.
- A manual **circuit breaker** sits above all of it: operators can halt a market entirely under extreme stress, independent of the automatic, fine-grained degradations.

These aren't independent features stacked together — they form one coherent system that weakens risk at the source, absorbs what remains, and caps the worst case.

See also: [Perpetual Futures](Perpetuals.md) · [Perp Liquidation & Insurance Fund](Perp-Liquidation.md) · [Perp Bots](Perp-Bots.md).
