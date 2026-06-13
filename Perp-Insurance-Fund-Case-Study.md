# Case Study: How a Meme Coin Drained a Perp Venue's Insurance Fund

> **Status: design.** This page studies a **public, external** incident on another perpetuals venue (Hyperliquid, March 2025) and explains the design principles Avodex's [insurance fund](Perp-Liquidation.md#insurance-fund) and [risk engine](Perp-Risk-Engine.md) adopt in response. Forward-looking; not a description of a shipped feature.

The single most important property of a perp insurance fund is that it must **never become an unbounded, forced counterparty of last resort.** The clearest lesson in why comes from a real incident, so we study it directly.

## What happened

In March 2025 a trader drained a large share of a major perp venue's backstop vault using a low-liquidity meme coin (ticker "JELLY"), with a market cap around $10M, that nonetheless had a leveraged perpetual listed against it. The attack was not a directional bet — it was a **self-liquidation** play across several fresh accounts:

1. **Matched positions.** The trader opened a large short on the meme perp on one account and offsetting longs on others — roughly delta-neutral, with the short deliberately over-leveraged. Total capital was only a few million dollars.
2. **Forced self-liquidation.** They then **withdrew margin** from the short account, deliberately pushing it past its maintenance threshold so it was liquidated. The position — hundreds of millions of tokens — was far too large for the thin order book to absorb, so the venue's **liquidator vault inherited it.**
3. **Squeeze the vault.** With the vault now holding a huge passive short, the trader **bought the spot token aggressively**, pumping its price ~400%. Because the mark price tracked that thinly-traded spot market, the vault's inherited short went deeply underwater — a paper loss in the eight figures, threatening a vault worth hundreds of millions.
4. **Emergency resolution.** Validators voted within minutes to delist the perp and **force-settled every position at the attacker's original entry price** — turning the vault's large paper loss into a small realized gain.

The money was largely contained, but the *resolution* caused its own damage: settling at a self-favourable price, decided ad hoc by a small validator set, drew heavy "they changed the rules mid-game" criticism, a sharp drop in the venue's token, and large vault outflows. The same self-liquidation-via-margin-withdrawal pattern had stressed the same vault two weeks earlier with a large ETH position — it was a *repeatable* attack, not a one-off.

## Why the backstop failed

Five interlocking failures, each a design lesson:

1. **The vault was an unbounded forced counterparty.** When the book couldn't absorb a liquidation, the vault simply inherited the whole position — with no cap on how toxic or how large.
2. **The mark price rode a single thin spot market.** On a ~$10M-cap token, a few million dollars of buying moved the reference price hundreds of percent, instantly ballooning the vault's loss. (This is the same oracle-manipulation shape as the 2022 Mango Markets drain.)
3. **Position and open-interest limits weren't tied to real liquidity.** A multi-million-dollar position was allowed on a token whose spot market couldn't possibly absorb it.
4. **Auto-deleveraging triggered too late.** It was scoped to fire only once the backstop was nearly exhausted, instead of as soon as the vault took on a dangerous toxic position — so it never socialised the bad position in time.
5. **There was no guard against self-liquidation.** Nothing stopped a trader from withdrawing margin specifically to force their own liquidation and hand an un-closeable position to the vault.

## How Avodex's design responds

These five failures map directly to five design commitments:

### 1. The insurance fund is never an unbounded counterparty
The [insurance fund](Perp-Liquidation.md#insurance-fund) backstops liquidations only up to a **bounded amount per market.** Beyond that cap, the position is socialised through [auto-deleveraging](Perp-Risk-Engine.md) instead of being silently absorbed. Crucially, **deleveraging triggers early** — as soon as the backstop's exposure to a single market's toxic position passes a danger line — *not* after the fund is drained. The fund is also **segmented per market**, so one volatile market's losses can never bleed into another's.

### 2. Limits are tied to real liquidity
Maximum leverage and total open interest for a market are derived from its **actual liquidity** — market cap and order-book depth — not set by fiat. Newly-listed and low-cap markets start with very tight caps and low leverage, loosening only as real depth accumulates. A position can never grow large relative to the liquidity that would have to absorb its liquidation.

### 3. Listing requires deep, multi-source pricing
A perp's [mark price](Perp-Mark-Price.md) is anchored to an external index — but that defence is only as strong as the index. A market is therefore listed only when it has **sufficiently deep liquidity across multiple independent reference sources.** A token whose price lives on a single thin venue either isn't listed as a perp, or is offered only as a tightly-capped, low-leverage market clearly marked as high-risk. The structural mistake in the case above was giving a deep perpetual to an asset that didn't deserve one.

### 4. Self-liquidation is guarded
A withdrawal of margin or realized profit is **rejected if it would push a position past a safety buffer above its maintenance margin** — removing the core primitive of the attack. Because the matched-position version of the attack uses *separate* accounts (so per-account self-trade prevention doesn't see it), this guard works in concert with the liquidity-tied limits and early deleveraging above: no single control stops it, but together they remove the incentive and bound the damage.

### 5. Crisis rules are pre-committed and neutral
The most lasting damage in the case study was to *trust*, because the emergency response looked like discretion exercised in the venue's own favour. Avodex's commitment is that any emergency settlement or delisting follows **published, neutral rules decided in advance** — for example, settling at the last valid reference price from *before* manipulation began, never a price chosen on the spot to benefit the fund. Delisting thresholds, the fund's status, and a trader's deleveraging-queue position are all visible ahead of time, so that emergency action reads as policy, not improvisation.

## The one-line takeaway

> An insurance fund must never be an unbounded forced counterparty — and a venue must never rescue itself by changing the rules after the fact. Prevention (liquidity-tied limits, sane listings), bounded absorption (caps, per-market funds, early deleveraging), and pre-committed neutral crisis rules are what keep a single bad market from taking down the whole system — or its credibility.

---

*The incident above is publicly documented; see e.g. [CoinDesk](https://www.coindesk.com/markets/2025/03/26/hyperliquid-delists-jellyjelly-after-vault-squeezed-in-usd13m-tussle), [Halborn](https://www.halborn.com/blog/post/explained-the-hyperliquid-hack-march-2025), and [OAK Research](https://oakresearch.io/en/analyses/investigations/hyperliquid-jelly-attack-context-vulnerability-team-solution) for independent accounts.*

This incident was one of a recurring pattern; the broader survey and its design lessons are in [Adversarial Lessons](Perp-Adversarial-Lessons.md).

See also: [Adversarial Lessons](Perp-Adversarial-Lessons.md) · [Perp Liquidation & Insurance Fund](Perp-Liquidation.md) · [Perp Risk Engine](Perp-Risk-Engine.md) · [Perp Mark Price](Perp-Mark-Price.md) · [Perpetual Futures](Perpetuals.md).
