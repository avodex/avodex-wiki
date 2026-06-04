# Grid Trading

A **grid bot** profits from price oscillation by placing staggered buy and sell limit orders across a price range.

## Parameters
- **Price range** (lower / upper bound).
- **Grid count** (number of levels, 2–500).
- **Investment** (amount allocated to the bot).

## How it works
1. **Initialize** — the bot buys an initial position at market (roughly half the investment becomes the base asset), then places **buy** limit orders at the grid levels below.
2. **Buy fills** — when a buy fills, the bot places a **sell** order one level higher and opens a cycle.
3. **Sell fills** — when that sell fills, the cycle closes (realized profit ≈ price difference × quantity, minus fees) and the bot re-places a **buy** at the original level.
4. Repeating this across the grid turns oscillation into accumulated **realized PnL**.

> Every order the bot places — including the buys and sells it re-places each round — carries a unique [client order id](Client-Order-IDs.md). The bot advances a cycle counter so an id is never reused across rounds; reusing one would desync the bot from the order book.

## Lifecycle
- **Create** → **Running** → optionally **Pause** (cancel orders, hold funds) / **Resume**.
- **Close** — cancel the bot's orders, sell any held inventory back to quote currency, and return funds.
- **Adjust range** / **add investment** — re-compute and re-place the grid.

## Good conditions
Grid trading performs best in **sideways / ranging** markets; strong trends can leave the bot holding inventory on the wrong side.
