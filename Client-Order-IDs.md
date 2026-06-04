# Client Order IDs

Every order Avodex places carries a **client order id** — a label chosen by whoever creates the order. The matching engine treats the pair **(account, client order id)** as an **idempotency key**: within a time window, re-submitting the same pair returns the *original* result instead of creating a second order. This makes order placement safe to retry over an unreliable network — a retried request can never accidentally double-place.

## One id, one live order

The same pair is also a **uniqueness key**: an account may have at most **one live order** for a given client order id at a time. If a new order reuses a client order id that already belongs to a *live* resting order, the engine **rejects** it — the same protection major exchanges apply to duplicate client ids. This guarantees the engine's records and the order book never disagree about which order an id refers to.

## Re-placing always uses a fresh id

A strategy that places an order, sees it fill, and then wants to place *another* order — for example at the same price on the next round — must use a **new** client order id, never the one it just used. Reusing the previous id is ambiguous: the engine cannot tell a genuine retry from a brand-new order, so one of two things happens:

- while the original is still live, the duplicate is **rejected**; or
- treated as a retry, the re-placement is **silently de-duplicated** and never actually rests.

Either way the strategy ends up out of sync with the book.

Avodex's [grid bots](Grid-Trading.md) follow this rule by advancing a **cycle counter** that is part of every order's id. Each round of a grid level produces a distinctly identified order, so an id is never reused across rounds.

## Why it matters

This is the "[confirm, don't assume](Order-Reconciliation.md)" discipline applied to *identity*. A fill, a balance change, or a re-placement is only correct when every order it touches is unambiguously identified. Stable, unique, never-reused client order ids are what let automated strategies place, cancel, and re-place orders reliably — and let the engine stay the single source of truth for [balances](Balances.md) and trades.
