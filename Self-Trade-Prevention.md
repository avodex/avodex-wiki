# Self-Trade Prevention (STP)

**STP** is a matching-engine safety feature: it prevents a single account from trading against its own resting orders (which would be wash trading and would distort fills).

Avodex's engine uses an **expire-taker** policy — if an incoming (taker) order would match a resting order from the **same account**, the taker order is expired instead of executing.

## Why it matters for automated strategies
Automated strategies that place many orders for the same account must be designed so their own orders don't cross each other, otherwise STP will expire the crossing side. Grid strategies account for this when placing and cancelling orders.

> STP is a normal, desirable protection. It is mentioned here so strategy authors understand why a same-account crossing order may not fill.
