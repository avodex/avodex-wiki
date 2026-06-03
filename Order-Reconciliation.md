# Order Reconciliation

To make sure every fill is accounted for, Avodex uses **two independent tracks** (defense in depth):

1. **Real-time events** — the engine emits fill events as orders execute.
2. **Periodic reconciliation** — a background job re-checks open orders against the engine to catch anything an event missed.

## Principle: confirm, don't assume
A "not found" response when querying an order is **ambiguous** — an order can be absent because it fully filled, because it was cancelled, or because of a transient hiccup. Avodex's reconciliation therefore **confirms fills against the engine's authoritative trade history** (real executed price and quantity) and never infers a fill from an ambiguous signal. This prevents fabricated fills and incorrect PnL.
