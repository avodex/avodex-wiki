# Profit Snapshots

Bot profit curves (in the portfolio and on share posters) are backed by a **periodic snapshot series**, not by recomputing history on demand.

## How it works
- A background job snapshots every live bot on a fixed interval: total PnL (realized + mark-to-market unrealized), PnL %, mark price, and position size.
- The snapshot uses **the same formula as the live overview**, so the last point of the curve always equals the headline number on the bot card — the two can never disagree.
- Paused and partially-placed bots are snapshotted too: they still hold a position, and a curve that goes blank while a bot is paused would be misleading.
- One price fetch per trading pair per round, no matter how many bots share the pair.
- Snapshots age out after a fixed retention window; the curve is a rolling view, not an archive.

## Honest-data notes
- When the series was first introduced, history was reconstructed from closed trading cycles. Reconstructed points are realized-only (historical unrealized PnL is unknowable after the fact), so old curves may show a small step where reconstruction meets live data. It ages out with retention.
- If no price is available at snapshot time the bot's entry price is used rather than dropping the point — a gap in the curve misleads more than an approximation.
