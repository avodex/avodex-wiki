# Perp Mark Price

> **Status: design.** Forward-looking architecture for Avodex perpetual futures. See [Perpetual Futures](Perpetuals.md) for the overview.

A perpetual's **mark price** is the reference price used to value positions, settle [funding](Perp-Funding.md), and trigger [liquidation](Perp-Liquidation.md). It is deliberately **not** the last trade price — basing those decisions on the last print would let a single thin-book trade move someone's collateral or trigger their liquidation.

## Two price layers the engine already has

Avodex's matching engine already runs two reference-price mechanisms for spot:

- An **internal reference** — a smoothed average of recent trades, with a robust estimator (a median of the last trade, the book mid, and the smoothed average) used by conditional/trigger orders. It resists a single bad print, but every input is an **internal** book signal.
- An **external cross-check** — an aggregator of independent outside markets that asks, continuously, "is our internal price still in the same neighbourhood as the rest of the world?" If the internal price diverges from the external consensus by too much for long enough, the market is flagged for review. This is the structural defence against price manipulation: a wash trader can move one book, but not several independent external markets at once.

## Why perps need a new mark

The internal robust estimator is fine for a user's own stop-loss — but its three inputs are all internal book signals, so a determined manipulator who controls the book can move all of them together. A perp mark decides **other people's money** — who pays funding, whose position gets liquidated — so it must be anchored to **external** reality, not internal book state.

So perp mark price is built on an **oracle index**: an aggregate of independent external markets, the same manipulation-resistant source the cross-check already uses.

- **Index price** = the external oracle aggregate.
- **Mark price** = the index, optionally nudged toward a manipulation-resistant internal fair price (an *impact* mid, measured deep enough into the book that a small order can't move it) but **clamped to a tight band around the index.** The clamp is the safety: even if someone briefly controls the book, the mark can't be dragged away from external reality.

## How an external price becomes deterministic engine state

This is the subtle part. The engine is **deterministic and replayable** — replaying its event log must reproduce identical state. But an oracle aggregator is inherently *non-deterministic*: which external source answers first depends on live network timing. You cannot read it directly inside the deterministic settlement path, or two replays would diverge.

The engine already solved this shape for its external cross-check: the oracle never mutates state directly; it only ever causes a **journaled event** to be written to the same log as everything else, and replay follows the recorded events without re-consulting any live source.

Perp mark price reuses that exact contract. A live publisher reads the oracle and the book, computes the mark, and emits a **mark-update event carrying the mark and index values it decided on.** The engine consumes the journaled value; funding and liquidation read it; and replay reproduces identical results from the event payload, never re-reading a live price. It is the same pattern as conditional orders, scheduled algos, funding, and liquidation — *a live evaluator decides, a journaled event records, replay reuses the record.*

## When pricing is unavailable

Perp markets are **oracle-required**. If every external source is unavailable, the mark stops updating and the market degrades safely — [funding](Perp-Funding.md) and [liquidation](Perp-Liquidation.md) **freeze**, and the market accepts only reduce-only orders — rather than acting on a stale or manipulable price. When pricing recovers, updates resume from fresh data; the gap is not back-filled from stale values. If the internal price merely drifts from the index (rather than the oracle failing outright), the market is flagged for manual review while pricing stays live.

## Don't confuse the three prices

A perp market carries three distinct prices with separate jobs:

1. The **internal smoothed average** still gates *order prices* — it rejects fat-finger limit orders far from the market.
2. The **external cross-check** still guards against the internal price being manipulated.
3. The **perp mark** values positions and drives funding and liquidation — and does **not** gate order placement.

Order matching uses the first two; position valuation, funding, and liquidation use the third. They read the same external source but never substitute for one another.

See also: [Perpetual Futures](Perpetuals.md) · [Perp Funding](Perp-Funding.md) · [Perp Liquidation & Insurance Fund](Perp-Liquidation.md).
