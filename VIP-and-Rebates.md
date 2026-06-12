# VIP Tiers & Referral Rebates

## VIP tiers
Trading fees are tiered: higher rolling volume → lower maker/taker fees and a higher referral-rebate share. Tiers are re-evaluated daily at UTC midnight from trading volume (a user's own and their team's).

## Bot trades count
Bot products (grid, DCA) execute through deterministically derived **per-user engine sub-accounts** rather than the user's main account. An attribution layer resolves those derived accounts back to the owning user, so **bot volume counts toward VIP progression and rebates exactly like manual trading** — regardless of which product placed the order. New bot products inherit this automatically.

## Referral rebates
- A referrer earns a share of each downline trade's **fee** (the share rises with the referrer's tier, subject to a daily cap).
- Settlement is **T+1**: a daily snapshot credits the previous day's rebates in one batch.
- Crediting is idempotent — at most one ledger entry per (trade, downline), so replays and retries can never double-pay.

## Fail-closed by design
A rebate is only paid when the trade carries **verifiable fee evidence**; if fee data is missing, the rebate is skipped and logged rather than estimated. The invariant: **a rebate never exceeds the fee actually collected.** Paying out on an assumed fee would turn a data gap into a money leak.
