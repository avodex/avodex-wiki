# Balances

Funds move through a few distinct contexts. Keeping them straight avoids confusion about "where is my balance."

- **Trading balance** — held by the matching engine; this is what's used to place and fill orders. It is the authoritative balance for trading.
- **Account (spot) balance** — your exchange balance as shown in the app.
- **Grid pool** — funds allocated to a running grid bot (split between resting orders and held inventory).

## How funds move
- **Deposit** (on-chain) credits your trading balance. See [Deposits & Withdrawals](Deposits-and-Withdrawals.md).
- **Starting / funding a grid bot** allocates funds from your account into the grid pool.
- **Closing a bot / taking profit** returns funds from the grid pool back to your account.

## Integrity
Every balance change is recorded with an audit trail, and balances are periodically reconciled so the per-bot accounting always matches the pool total.
