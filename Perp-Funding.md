# Perp Funding

> **Status: design.** Forward-looking architecture for Avodex perpetual futures. See [Perpetual Futures](Perpetuals.md) for the overview.

**Funding** is the mechanism that keeps a perpetual's price tethered to its spot index. Because a perp never expires, periodic funding payments nudge the contract back toward the index: when the perp trades at a premium, longs pay shorts; at a discount, shorts pay longs.

## Funding is a peer transfer — net zero by construction

Avodex perps are **peer-to-peer**: there is no external market maker on the other side. Every contract is created by matching one long against one short, so total long size always equals total short size. Funding moves value **between traders**, and across the whole market it sums to **exactly zero** — it neither mints nor burns USDT. Any sub-cent rounding remainder is swept to the [insurance fund](Perp-Liquidation.md#insurance-fund) so the books stay exact to the smallest unit.

## How the rate is computed

Between settlements, the engine samples the **premium** of the mark price over the index at a fixed cadence and time-averages it. At each interval the funding rate is derived in the standard way:

```
rate = avg_premium + clamp(interest − avg_premium, ±band)
rate = clamp(rate, ±cap)
```

- The **interest** term is a small constant (often zero for a USDT-margined linear contract).
- The **band** keeps the interest term from dominating a quiet market.
- The **cap** is a hard per-interval limit on how large funding can be.

The per-position payment is then `position_size × mark × rate`, charged to longs and credited to shorts when the rate is positive (and vice-versa).

## Settlement

At each funding time a live scheduler computes the rate and emits a single **journaled settlement event** carrying the rate and mark price it decided on. The engine applies it deterministically: it walks the market's open positions in a fixed order, books each payment, and records the interval. Because the decisive values travel inside the event, replaying the log reproduces identical funding — the engine never re-samples or re-reads a live price during replay. Each interval is idempotent: a duplicate or replayed settlement for an interval already booked is a harmless no-op.

Funding adjusts a position's **margin**, which means it directly changes the position's equity.

## Interaction with liquidation

Because funding moves margin, a payment can push a position closer to — or past — its liquidation threshold. Two rules follow:

1. **Liquidation is re-checked immediately after every funding settlement**, so a position weakened by funding is caught at once.
2. The funding **cap is kept well below the maintenance-margin rate.** This guarantees a single funding payment can never jump a healthy position straight through to insolvency, bypassing the liquidation buffer. It is a hard safety constraint, enforced when a market is configured.

When settling, funding is applied **first**, then liquidation is evaluated against the updated equity — so the right positions are caught, and only those.

## When pricing is unavailable

Funding depends on the index. If the oracle layer cannot price the market, funding **freezes** (no settlement is emitted) and the market goes reduce-only, matching the liquidation freeze. When pricing recovers, funding resumes from the next whole interval; the frozen interval is **not** back-computed from stale data.

## Worked example

A market funds hourly; one interval's averaged premium works out to a rate of `+0.03%` (the perp is trading rich, so longs pay shorts). Two opposing positions, each 10 units at a mark of 100:

- The **long** pays `10 × 100 × 0.0003 = 0.30 USDT` (its margin drops by 0.30).
- The **short** receives `0.30 USDT` (its margin rises by 0.30).

The two cancel exactly — nothing is created or destroyed — and liquidation is re-checked for both straight afterward.

> Numbers here are illustrative, chosen to show the mechanism.

See also: [Perpetual Futures](Perpetuals.md) · [Perp Mark Price](Perp-Mark-Price.md) · [Perp Liquidation & Insurance Fund](Perp-Liquidation.md).
