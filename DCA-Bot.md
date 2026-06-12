# DCA Bot

A **DCA bot** (dollar-cost averaging, 定投) buys a fixed amount on a fixed schedule, spreading entries over time instead of timing the market.

## Modes
- **Recurring** — runs indefinitely on a calendar cadence (hourly up to quarterly) until closed. Supports an optional **max buy price**: runs above the cap are skipped and retried next period automatically — the bot is never paused by the cap.
- **Fixed count** — runs N times on a short cadence (seconds to minutes), then completes and settles automatically. Supports **buy** (quote-denominated) and **sell** (base-coin-denominated) directions.

## Scheduling
- First run is either **immediately on creation** or at a **custom daily time**. A custom time resolves to its *next occurrence*: setting 23:50 at 23:40 fires 10 minutes later; a time already past fires tomorrow.
- Subsequent runs stride by the chosen frequency; calendar-month frequencies use real calendar arithmetic, and day-level cadences stay anchored to the chosen time so execution delay never drifts the schedule.

## Funding model (just-in-time)
Unlike a grid bot, a DCA bot **locks nothing up front**. Each run transfers exactly one period's amount from the user's main account, executes a market order, and returns any unspent remainder immediately. Accumulated holdings return to the main account at close — either as the coin, or sold back to quote currency, the user's choice.

If a run finds insufficient balance it is recorded and skipped; several consecutive shortfalls auto-pause the bot rather than letting it spin.

## Reliability
Every run carries a deterministic [client order id](Client-Order-IDs.md) and a unique run sequence, so a crash-and-replay can never double-buy or double-book a period. Skipped and failed runs are recorded alongside fills — the execution history shows what *didn't* happen, too.
