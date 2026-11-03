# haWAL contract rules

Use these rules when selecting an operation and reviewing its returned transaction. Check current balances, fees, pauses and claim eligibility.

## Operation checks

| API operation | Behavior and conditions |
|---|---|
| `stake` | Spend WAL, mint haWAL. At least 1 WAL; active-validator list must be nonempty; minted output must be nonzero; staking must not be paused. |
| `withdraw` | Redeem haWAL, create an `UnstakeTicket`,. Gross WAL equivalent must be at least 1 WAL and no more than total protocol WAL; unstaking must not be paused. Current time must fit the configured Walrus epoch time range. |
| `withdraw_instant` | Redeem haWAL, receive WAL after service fee. Same gross amount and pause checks; enough WAL must be available to complete the withdrawal. |
| `claim` | Consume an owned `UnstakeTicket` and receive its recorded WAL amount. Claim must not be paused and both maturity conditions below must pass; accessible WAL must cover the claim. |

Amounts use the protocol exchange rate; do not assume haWAL and WAL are 1:1. WAL and haWAL response amounts use 9 decimals. Preserve raw-integer precision, keep API `amount` in human units, and reserve SUI for gas.

Instant service fee in raw units is
`floor(gross_wal * current_service_fee / 10_000_000)`.
Read the current config, show gross/fee/net estimates and check simulation effects. Total protocol WAL is not a guarantee of immediate capacity; Walrus withdrawal conditions affect accessible funds.

Delayed withdrawal uses **Walrus** epochs: claim is scheduled for n+1, or n+2 after the midpoint of the configured epoch. Do not substitute Sui epochs or assume a fixed epoch duration; read the current Walrus configuration. Use the ticket's recorded values. Claim requires BOTH:

- Current **Walrus `System.epoch`** >= `claim_epoch`.
- **Clock timestamp** >= `claim_timestamp_ms`.

The ticket is consumed on successful claim. Presence in `get_unstake_tickets_list` does not prove maturity.
