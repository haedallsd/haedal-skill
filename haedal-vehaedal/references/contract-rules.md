# veHAEDAL contract rules

Use these rules when selecting a staking, redemption or reward operation. Check the selected position, minimum stake and reward eligibility.

## Position eligibility

Confirm that the selected position locks HAEDAL and belongs to the signer. Both its current object ownership and returned `owner` field must match the signer.

Let `W = 7 * 24 * 60 * 60 * 1000` milliseconds. Effective remaining full weeks are the saved `remaining_lock_weeks_when_stopped_decay` when non-decaying; while decaying they are `floor(max(lock_end_time - now, 0) / W)`.

| Operation | Required state / effect |
|---|---|
| `add_stake` | Positive HAEDAL amount meeting the current minimum stake; integer `lockWeeks` from 1 through 52. Creates a new position. Select decay mode from user intent. |
| `add_to_existing_stake` | Positive amount, correct owner, at least 1 remaining full week; a decaying position must not be expired. Adds principal to this position. |
| `extend_existing_lock` | Correct owner; integer `additionalWeeks` from 1 through 52; effective remaining full weeks + additionalWeeks <= 52; a decaying position must not be expired. |
| `start_decay` | Correct owner and currently non-decaying. Sets start to current chain time and end to now + saved remaining weeks; starts the countdown. |
| `stop_decay` | Correct owner, currently decaying, not expired, and at least 1 remaining full week. Freezes the remaining duration as whole weeks rounded up. The entry's full-week minimum still applies. |
| `unstake_and_claim` | Correct owner and decaying mode; current chain time >= the UTC midnight starting the day containing lock_end_time. Consumes the position and returns locked HAEDAL principal. It does not claim separate distribution rewards. |

Non-decaying mode freezes the countdown; an old `lock_end_time` is not permission to redeem. Do not silently start decay to satisfy a redemption request: explain the new countdown and obtain the user's intended action.

The UTC-day redemption threshold is
`floor(lock_end_time / 86_400_000) * 86_400_000`.
It can precede the exact timestamp by less than 24 hours; display both threshold and lock-end time with timezone. This contract rule does not imply that stop-decay/add-stake may use the same early threshold.

## Displaying position values

`locked_amount` is locked principal. `initial_amount` and `current_amount` are veHAEDAL weight fields, not additional withdrawable HAEDAL. At creation, initial weight is `floor(amount * lock_weeks / 52)`.

Use current on-chain position data to obtain time-dependent weight. For non-decaying mode it returns initial weight; for decaying mode it computes a rounded value from chain time and returns zero at exact expiry. The stored `current_amount` is not continuously updated. If a live getter is unavailable, label the field as stored/unverified instead of calling it the current balance. Use token metadata for decimal display, preserving integer precision.

## Separate reward distribution

`claim_rewards_v2` claims HAEDAL rewards for the supplied distribution periods. Send `periods` as decimal strings in the range 0..4294967295. Period IDs are not Sui/Walrus epochs or lock weeks.

`claim_rewards_v2_epoch_1` claims haSUI rewards for fixed period 1; only `signerAddress` is required.

Check the signer's eligibility, unclaimed amount, reward coin and claim-pause state before preparing either operation. Use available read tools or reward information to identify claimable periods; do not invent an HTTP discovery endpoint. One invalid period can cause the whole claim to fail. Already claimed periods may be skipped, so success with an empty or fully claimed list does not establish a positive reward payment. Verify the actual received asset and amount in simulation.

## Error handling

Keep the HTTP status, error message and full MoveAbort location when diagnosing a failure. Error numbers can be reused by different on-chain modules. Apply the SKILL.md staking error table only to staking errors; a reward claim's code 9 means claiming is paused, not that a position's lock has expired.
