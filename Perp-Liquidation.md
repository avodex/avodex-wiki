# Perp Liquidation & Insurance Fund

> **Status: design.** Forward-looking architecture for Avodex perpetual futures. See [Perpetual Futures](Perpetuals.md) for the overview.

When a leveraged position's losses erode its collateral toward zero, the position must be closed before it goes underwater. **Liquidation** is how the engine does this safely; the **insurance fund** is the backstop that absorbs the rare cases where market liquidity isn't enough.

## When liquidation triggers

A position is healthy while its equity stays above its maintenance margin:

```
equity            = margin + unrealized PnL (at mark)
maintenance margin = notional · maintenance-margin-rate
liquidation triggers when:  equity ≤ maintenance margin
```

Crucially, this is measured against the **mark price** (the oracle index), not the last trade — so a single thin-book print can't trigger someone's liquidation, and a position being liquidated can't cascade into others purely through its own market impact.

### Liquidation price vs. bankruptcy price

For a long position, two prices matter as the market falls:

```
liquidation price ≈ entry · (1 − initial-margin-rate) / (1 − maintenance-margin-rate)
bankruptcy price  ≈ entry · (1 − initial-margin-rate)
```

The liquidation price sits **above** the bankruptcy price. The gap between them — the maintenance-margin buffer — is the window the engine has to close the position **while collateral still remains.** Closing inside that window means the system takes no loss; only if the market gaps past the bankruptcy price does the insurance fund step in.

> Example, illustrative: a 10× long entered at 100 with a 2.5% maintenance rate liquidates around 92.3 and goes bankrupt at 90.0 — a ~2.3-point cushion the engine uses to close out.

## The liquidation waterfall

Liquidation is a journaled, deterministic event — the mark price it fires at is carried inside the event, so it replays identically. Execution proceeds in layers:

1. **Close into the order book.** The engine submits a reduce-only order on the account's behalf. Real liquidity discovers a real price, and the position holder keeps whatever margin remains after the close.
2. **Insurance-fund backstop.** If the book can't fully absorb the position, the **insurance fund** takes over the remainder at the bankruptcy price, guaranteeing the position always closes — a perp can never leave an underwater position open.
3. **Auto-deleveraging (later).** A last resort if the insurance fund itself is exhausted (see below).

### Penalty, residual, and bad debt

After the close at an achieved price:

- If margin remains, a **liquidation penalty** (a small fraction of the position's value) goes to the insurance fund, and any leftover returns to the user.
- If the position closed **below** bankruptcy, the shortfall is **covered by the insurance fund** — the user loses their posted margin but never owes more.

Either way the books stay conserved: the user's loss is absorbed exactly by margin, realized PnL, and the fund — no USDT appears or disappears.

### Partial liquidation

Rather than dumping a whole position into the book, the engine can close just enough to bring the position back to health, capped per event. This limits market impact and gives the book room to recover; if the position is still unhealthy on the next mark update, another step follows. Closing the full position is the simple first iteration; partial liquidation is a refinement.

## Insurance fund

The insurance fund is a reserved engine account that:

- **Receives** liquidation penalties (and rounding remainders from [funding](Perp-Funding.md)).
- **Pays** the shortfall when a position closes past its bankruptcy price.

It is seeded with starting capital, monitored for its balance level, and alerts when it runs low. Like the platform's other reserved accounts, it is a **non-signable** identity — no external party can ever withdraw from it; only an administrative path can adjust it. In a later iteration each market gets its own sub-fund, so a single volatile market's losses can't drain the others.

If the fund is ever exhausted, the design's last resort is **auto-deleveraging** — force-closing the most-profitable, highest-leverage opposing positions at the bankruptcy price to absorb the loss. The ranking is defined up front (and surfaced to users as a risk indicator), even though automated deleveraging is deferred to a later phase; until then, a fund shortfall is handled by alerting and replenishment.

## Cascade safety

The biggest systemic risk in any perp venue is a liquidation cascade — forced selling driving price down, triggering more forced selling. Avodex's design dampens this on several fronts:

- Liquidation is judged on the **oracle mark**, not on the prices liquidations themselves print, so a sell-off doesn't immediately amplify itself.
- **Partial liquidation** spreads closes out over time.
- The **insurance fund** absorbs whatever the book can't, instead of pushing it back onto the order book.
- Markets can be set **reduce-only** under stress to stop new risk from building.

See also: [Perpetual Futures](Perpetuals.md) · [Perp Funding](Perp-Funding.md) · [Perp Risk Engine](Perp-Risk-Engine.md) · [Insurance-Fund Case Study](Perp-Insurance-Fund-Case-Study.md) · [Perp Bots](Perp-Bots.md).
