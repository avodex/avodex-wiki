# Changelog

Sanitized, user-visible highlights. Conceptual details live on the linked pages.

## 2026-06
- **DCA bot launched** — recurring + fixed-count modes (including coin-denominated sell), just-in-time funding, price-cap skip, crash-safe per-run idempotency. See [DCA Bot](DCA-Bot.md).
- **Bot trades now count toward VIP & rebates** — per-user sub-account attribution resolves bot fills back to the owning user; grid and DCA volume drive tier progression and referral rebates like manual trades. See [VIP & Rebates](VIP-and-Rebates.md).
- **Rebate engine hardened fail-closed** — payouts require verifiable fee evidence; a rebate can never exceed the fee collected.
- **Real profit curves** — periodic profit snapshots power the portfolio chart and share posters; the curve endpoint always equals the bot card's headline PnL. See [Profit Snapshots](Profit-Snapshots.md).
- **Portfolio bot orders** — running + history views with share poster (real PnL, rate/amount modes, invite QR) and full lifecycle from the card: pause, **resume** (price-delta preview before re-locking), add funds, adjust range, rename, close (sell-for-me / keep-coin).
- **Range adjustment fixes** — before/after position ratios in the preview are valued at one price (adjusting a range never trades your holdings, and the numbers now say so); partially-placed bots can adjust their range to self-heal.
- **Localization parity** — bot lifecycle and trading copy completed across all five supported languages.

## 2026-05
- Public wiki established; grid trading, balances, reconciliation, and client-order-id concept pages.
