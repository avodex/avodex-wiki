# Adversarial Lessons: Recurring Failure Modes in Perp DEXs

> **Status: design.** This page surveys **public, external** perpetual-DEX incidents (primarily on Hyperliquid, 2024–2026) and the design principles Avodex's perp engine adopts in response. It builds on the single-incident [insurance-fund case study](Perp-Insurance-Fund-Case-Study.md). Forward-looking; not a description of shipped features.

The [meme-coin vault drain](Perp-Insurance-Fund-Case-Study.md) was not a one-off — it was one instance of a **recurring pattern.** Studying the wider history makes the pattern obvious: in almost every case, no code was "hacked." Instead, **an assumption baked into the original design failed under adversarial pressure.** This page catalogues those assumptions so a perp venue can pre-empt them rather than discover them in production.

## The pattern, by incident

| When | What happened (public) | The design assumption that failed |
|---|---|---|
| Mar 2025 | A large leveraged ETH long **withdrew its own margin to force liquidation**, handing the position to the backstop vault and booking ~$1.8M while the vault lost ~$4M | A liquidated position can be offloaded near the mark price |
| Mar 2025 | Meme-coin self-manipulation forced the vault to inherit a huge short, then a spot pump squeezed it (peak loss ~$12M) | Mark price can safely track a thin spot market; position limits need not scale to liquidity |
| Oct 2025 | A market-wide cascade liquidated ~$19B industry-wide; one venue processed ~$10.3B of it (and held up) | (Stress test — passed, but showed how much leverage concentrates on one venue) |
| Nov 2025 | An attacker **burned ~$3M of their own capital** (spoof buy-wall, then pulled it) to inflict ~$4.9M of bad debt on the vault | The attacker is profit-maximising *on your venue* |
| Apr 2026 | An attacker forced a thin token into a range to **trigger auto-deleveraging**, dumping a toxic position onto the vault while profiting on an **external hedge** | ADL is only a backstop, never a weapon; the adversary cares about on-venue PnL |
| May 2026 | A third-party-listed market's oracle **mishandled a 5-for-1 stock split**, reading it as an ~80% crash; the perp fell ~45% and liquidated hundreds of traders | Oracle inputs are clean; corporate actions don't happen |
| 2026 | Permissionless market creation concentrated ~90% of open interest in a single deployer | Opening listing to third parties decentralises market creation |

*(Dollar figures are from public reporting on external venues; sourced below.)*

## The recurring blind-spots

Stripped to their roots, these collapse into a handful of structural lessons:

- **A — The backstop must never be an unbounded forced counterparty.** Four separate incidents share one root: when the book can't absorb a liquidation, the vault silently inherits the whole position. Different attacker actions all funnel to the same outcome — socialised loss.
- **B — Mark price must resist both manipulation *and* clean-data assumptions.** A thin spot market is cheap to pump; a corporate action breaks naïve feeds. Anchoring to an external index is necessary but not sufficient — if the index itself jumps, anything that merely *clamps relative to the index* follows it off the cliff.
- **C — Leverage and open-interest limits must be functions of real liquidity** (market cap and book depth), not static notional ceilings — especially for new and long-tail markets.
- **D — Griefing is unpriced.** Attackers have repeatedly *lost their own capital* to inflict larger losses on a venue, profiting via external hedges or simply seeking damage. A risk model that assumes a profit-maximising on-venue adversary misses them entirely.
- **E — A discretionary emergency override is itself a risk.** If a venue can always intervene to cap its own losses, its "decentralised" guarantees are conditional — and an ad-hoc, self-favourable rescue is what destroys trust.
- **F — Permissionless listing re-introduces every prior risk at the long tail** — and, left to raw economics, tends to concentrate rather than decentralise.

## How Avodex's design responds

The [insurance-fund case study](Perp-Insurance-Fund-Case-Study.md) covers the bounded-backstop (A), liquidity-tied-limits (C), and pre-committed-neutral-rules (E) responses. The wider history adds four more commitments:

### Assume the adversary is willing to lose money
Avodex's risk model does **not** assume an attacker only acts when it's profitable on-venue. For every market it bounds the **maximum loss an attacker could force onto the insurance fund** — given that market's open-interest cap, leverage, and real liquidity — and treats that bound as a *listing gate*. Defences rely on **structural caps**, not on the attack being unprofitable, because a griefer or an externally-hedged adversary doesn't care about the on-venue loss.

### Auto-deleveraging must not be weaponizable
Because [auto-deleveraging](Perp-Risk-Engine.md) has itself been used as an attack, it is hardened against it: deleveraged positions settle at the **oracle mark price, never at a price an attacker pushed the local book to**, and the trigger that decides when to deleverage is judged on the oracle mark, not on prices the attacker printed. Combined with the per-market loss bound above, the most an attacker can gain by *inducing* deleveraging is capped.

### Mark price needs an absolute move cap, not just a relative clamp
Anchoring the [mark price](Perp-Mark-Price.md) to an external index defends against book manipulation — but not against the index itself jumping (a stock split, a broken feed, a manipulated source). So the mark also carries an **absolute limit on how far it can move in a single update.** A jump beyond that limit isn't accepted blindly; the market freezes to reduce-only and the price is reviewed, rather than letting a bad print instantly trigger liquidations. Markets driven by assets prone to corporate actions (splits, dividends — i.e. tokenised real-world assets) are simply **not listed** until that handling exists.

### Assume you are the venue where everything unwinds at once
One venue once absorbed more than half of an entire market-wide liquidation cascade. Avodex's stress testing therefore models not just a single market's cascade but a **market-wide, all-correlated** event — checking that the per-market insurance funds, partial liquidation, circuit breaker, and deleveraging hold up together when many markets go bad simultaneously.

### Listing is controlled, not permissionless
Avodex does not open perp-market creation to arbitrary third parties in its first iterations. Every market's oracle sources, leverage, and limits are reviewed against the standards above. This sidesteps the entire class of long-tail and concentration risks that permissionless listing reopened elsewhere; if permissionless listing is ever offered, it would require *enforced* minimum risk standards rather than leaving them to deployers.

## The one-line takeaway

> Every one of these failures was a design assumption — "the vault can always close a position," "the oracle is clean," "the attacker wants to profit here," "deleveraging is only a backstop" — that held until someone attacked it. The job is to treat those assumptions as attack targets *before* launch, not after.

---

*Incidents above are publicly documented; representative sources include [CoinDesk](https://www.coindesk.com/markets/2025/03/26/hyperliquid-delists-jellyjelly-after-vault-squeezed-in-usd13m-tussle), [Cointelegraph](https://cointelegraph.com/news/hyperliquid-hlp-popcat-attack-3m-wipeout), [The Block](https://www.theblock.co/post/345866/hype-drop-hlp-vault-loss-hyperliquid-whale-liquidation), and [Halborn](https://www.halborn.com/blog/post/explained-the-hyperliquid-hack-march-2025). Figures are approximate and from third-party reporting.*

See also: [Insurance-Fund Case Study](Perp-Insurance-Fund-Case-Study.md) · [Perp Liquidation & Insurance Fund](Perp-Liquidation.md) · [Perp Risk Engine](Perp-Risk-Engine.md) · [Perp Mark Price](Perp-Mark-Price.md) · [Perpetual Futures](Perpetuals.md).
