# haSUI contract rules

Use these rules when selecting an operation and reviewing its returned transaction. Check current balances, fees, pauses and claim eligibility.

## Operation checks

| API operation | Behavior and conditions |
|---|---|
| `stake` | Spend SUI, mint haSUI. At least 1 SUI; nonzero minted output; staking must not be paused. |
| `withdraw` | Redeem haSUI and create an `UnstakeTicket`. Gross SUI equivalent must be at least 1 SUI and no more than total protocol SUI; unstaking must not be paused. |
| `withdraw_instant` | Redeem haSUI, receive SUI after service fee. Same gross amount and pause checks; actual withdrawal must find enough accessible SUI. |
| `claim` | Consume an owned `UnstakeTicket` and receive its recorded SUI amount. Claim must not be paused and both maturity conditions below must pass. |

Amounts use the protocol exchange rate; do not assume haSUI and SUI are 1:1. Keep integer precision and reserve SUI for gas. For response fields, SUI and haSUI use 9 decimal places; do not rescale API request amounts.

For instant withdrawal, fee in raw units is
`floor(gross_sui * current_service_fee / 10_000_000)`.
Read the current config, show gross/fee/net estimates and verify actual effects in simulation. Initialization defaults are not current quotes; total protocol assets are not a guarantee of immediately withdrawable capacity.

Delayed withdrawal assigns `claim_epoch` to current Sui epoch +1 or +2 according to the configured withdrawal cutoff. Use the ticket's recorded values instead of promising a fixed wait. Claim requires BOTH:

- Current **Sui epoch** >= `claim_epoch`.
- Current **Sui epoch-start timestamp** >= `claim_timestamp_ms` (not merely current Clock time).

The ticket is consumed on a successful claim. Do not infer maturity from presence in `get_unstake_tickets_list`.
