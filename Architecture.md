# Architecture

Avodex is layered: a web app and an API layer in front of an in-house matching engine, with a separate grid-trading service and an on-chain deposit path.

```
        Web app (Next.js)
              │
        API layer  ──────────────►  Matching engine (in-house)
              │                         · order book
              │                         · trading ledger
        Grid service ───────────────────┘  · trades
              ▲
  On-chain deposit vault ── deposit watcher ── credits trading balance
```

## Layers
- **Web app** — the trading UI and the grid-bot interface.
- **API layer** — authentication (wallet sign-in), balances, order routing to the engine, and the on-chain deposit pipeline.
- **Matching engine** — the central-limit order book, the authoritative trading ledger, and trade execution. Fully in-house.
- **Grid service** — orchestrates automated grid strategies: builds the grid, places orders through the engine, and advances strategies as orders fill. See [Grid Trading](Grid-Trading.md).
- **On-chain** — an EVM deposit vault; deposits are detected and credited to the trading balance. See [Deposits & Withdrawals](Deposits-and-Withdrawals.md).

## Design principles
- **Authoritative ledgers** — the matching engine is the source of truth for trades and balances. See [Balances](Balances.md).
- **Confirm, don't assume** — fills are confirmed against authoritative trade data rather than inferred from ambiguous signals. See [Order Reconciliation](Order-Reconciliation.md).
