# Deposits & Withdrawals

## Deposit (on-chain → trading balance)
1. **Approve** the deposit vault to move your tokens.
2. **Deposit** into the vault; it pulls your tokens and emits an on-chain deposit event.
3. Avodex detects the confirmed event and **credits your trading balance**, making the funds tradeable.

Deposits are idempotent (keyed by the on-chain transaction), so retries are safe.

## Withdrawal
Withdrawals return funds from your Avodex balance to your wallet. On-chain withdrawal support is being finalized.

## Contracts
Contract addresses are published as the protocol deploys to each network. The Monad ecosystem listing tracks them as they go live; see the protocol entry in the Monad ecosystem registry.
