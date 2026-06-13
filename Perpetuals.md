# Perpetual Futures

> **Status: design.** This page describes how perpetual futures are designed to work on Avodex's in-house matching engine. It is forward-looking architecture, written at a conceptual level.

A **perpetual** ("perp") is a leveraged contract that tracks a spot index with no expiry. Avodex's perps are **linear, USDT-margined**: one contract equals one unit of the base asset, profit and loss settle in USDT, and the collateral you post is USDT.

## The key idea: same matching core, different settlement

The matching engine does not care whether the thing it matches is a spot asset or a contract. A buy and a sell cross by price–time priority and produce a fill — identically for spot and perp. The difference appears **only after** the fill, in how it is booked:

| | Spot fill | Perp fill |
|---|---|---|
| Buyer receives | the base coin | a **long position** (no coin changes hands) |
| Settlement | two assets swap hands | a position opens/closes; **margin** locks; realized PnL settles in USDT |
| You hold | the asset | a position backed by collateral |

So perps reuse the entire trading core — the order book, price–time priority, [self-trade prevention](Self-Trade-Prevention.md), time-in-force, [client order ids](Client-Order-IDs.md), deterministic sequencing and replay, snapshots, the [oracle](#mark-price) layer, and conditional (trigger) orders. What is new lives in a **settlement branch** plus three perp-native subsystems: [mark price](#mark-price), [funding](Perp-Funding.md), and [liquidation](Perp-Liquidation.md).

## Positions and margin

Each account holds at most one **net position** per market — a signed quantity, an average entry price, and the collateral (**margin**) backing it. Avodex perps use **isolated margin** in the first iteration: each position locks a fixed amount of USDT, and a position's losses can never exceed the margin posted to it.

For a position of size `Q` (signed: positive long, negative short) opened at price `E`, with margin `M`, at mark price `P`:

```
Notional        = |Q| · P
Unrealized PnL   = Q · (P − E)           (one formula covers long and short)
Position equity  = M + Unrealized PnL
Initial margin   = Notional_at_entry / leverage
Maintenance margin = Notional · maintenance-margin-rate
```

Opening or increasing a position locks margin; reducing or closing it releases margin proportionally and realizes PnL into USDT. All of this happens **atomically** with the fill — if the collateral isn't there, the order is rejected, exactly as a spot order is rejected for insufficient balance.

## Mark price

Trades match at order-book prices, but **funding and liquidation are driven by the mark price, not the last trade.** Mark price is anchored to an **oracle index** (an aggregate of external reference prices) rather than the most recent print, so a single thin-book trade cannot push someone into liquidation. A live evaluator periodically publishes the mark price into the engine as a journaled event, so replay reproduces the exact same values without re-reading any live source.

Because the index is essential — you cannot run a perp without it — perp markets are **oracle-required**. If no reference price is available, the market degrades safely: funding and liquidation freeze, and the market accepts **reduce-only** orders until pricing recovers.

## What stays the same

Everything that makes the spot engine correct carries over unchanged:

- **Deterministic, replayable settlement** — perp-specific events (mark updates, funding, liquidation) flow through the same single sequencer and write-ahead log as ordinary orders, carrying their decisive values in the event itself. Replaying the log reproduces identical state, including positions and margin.
- **Atomic, conserved ledger** — a perp fill's position update, margin lock, fee, and realized PnL are one all-or-nothing settlement.
- **Conditional orders** — the engine's existing mark-driven trigger orders are exactly stop-loss / take-profit for a position.

## Roadmap shape

1. A single market (isolated, linear), positions and margin, manual liquidation — prove the position ledger and settlement branch.
2. [Funding](Perp-Funding.md) settlement.
3. Automated [liquidation](Perp-Liquidation.md) and the insurance fund.
4. Cross-margin, auto-deleveraging, leverage/risk tiers, more markets.

See also: [Perp Mark Price](Perp-Mark-Price.md) · [Perp Funding](Perp-Funding.md) · [Perp Liquidation & Insurance Fund](Perp-Liquidation.md) · [Perp Risk Engine](Perp-Risk-Engine.md) · [Perp Bots](Perp-Bots.md).
