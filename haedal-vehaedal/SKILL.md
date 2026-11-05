---
name: haedal-vehaedal
description: |
  Use when the user mentions staking HAEDAL, locking, extending lock, claiming rewards, decay, or operating VeHaedal on Haedal Protocol (a DeFi ecosystem on SUI).
  Trigger phrases: "stake haedal", "lock haedal", "extend lock", "claim rewards from haedal", "unlock/redeem haedal", "add to stake", "start decay", "stop decay", "vehaedal".
  This skill prepares unsigned transactions via https://skillsapi.haedal.xyz/api/v1/vehaedal/* using curl POST. It distinguishes lock/decay operations, principal redemption, and separate reward distributions. Discover existing positions with get_vehaedal_list when needed; check current ownership and operation eligibility before building.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(curl:*), Bash(jq:*)
license: MIT
metadata:
  author: haedal
  version: '0.1.2'
---

# Haedal VeHaedal

**Haedal Protocol** is a DeFi ecosystem on the [SUI](https://sui.io) blockchain. **VeHaedal** is its vote-escrowed governance token: lock HAEDAL for veHaedal to earn rewards and voting power; lock duration and optional decay affect reward rates. This skill calls the Haedal Skills API for add stake, extend lock, claim rewards, decay control, and unstake — no custom scripts required.

Call the VeHaedal HTTP APIs directly with curl.

**API request `amount` is a human-readable decimal string**: send `"amount":"20"` for 20 HAEDAL, without multiplying by decimals. Object response amounts are raw integer units. `lockWeeks` and `additionalWeeks` are integer week counts; `periods` contains reward-distribution IDs, not token amounts or Sui/Walrus epochs.

## Contract rules and transaction handoff

Read [contract rules](references/contract-rules.md) before preparing a transaction. They cover lock modes, operation eligibility, principal redemption and separate reward distributions. Use current chain state for balances, fees, pauses and eligibility.

Choose `isDecaying` from the user's intended lock behavior. `false` freezes the remaining duration: it does not unlock automatically as wall-clock time passes. `start_decay` starts a new countdown from the current chain time using the saved remaining weeks. Explain this consequence before preparing a non-decaying position; do not silently choose a mode.

HTTP 200 with `txBytes` means an unsigned transaction was prepared. Before handing it to a signing wallet, decode and check the sender, network, Move targets, position, input asset/amount, recipient and expected asset changes, then run an unsigned checked simulation. If tooling or state is unavailable, report incomplete verification and do not describe the transaction as ready to sign. Signing and broadcasting require the user's authorization.

## Querying veHaedal objects (get_vehaedal_list)

Several methods (add_to_existing_stake, extend_existing_lock, start_decay, stop_decay, unstake_and_claim) require a **`vehaedalObj`** — the object ID of an existing veHaedal position. When the user triggers one of these methods:

1. **Ask the user** whether they already have the veHaedal object ID. If yes, use it directly.
2. **If discovery is needed**: call **get_vehaedal_list** with the user's `address`. On HTTP 200 the response body is:
   ```json
   {
     "list": [
       {
         "objectId": "0x...",
         "fields": {
           "current_amount": "10000000000",
           "initial_amount": "10000000000",
           "is_decaying": true,
           "lock_end_time": "1803708378450",
           "lock_start_time": "1772258778450",
           "locked_amount": "10000000000",
           "original_lock_weeks": "52",
           "owner": "0x...",
           "remaining_lock_weeks_when_stopped_decay": "52",
           "token_type": { "fields": { "name": "...::haedal::HAEDAL" }, "type": "0x1::type_name::TypeName" }
         }
       }
     ]
   }
   ```
3. **Validate and present positions**: check the full `VeHAEDAL<HAEDAL>` type, current object ownership AND the stored `owner` against the signer. Show decimal-formatted locked principal, mode and effective remaining weeks. Label raw `current_amount` as a stored field, not a live balance; use `staking::get_current_veheadal` for the time-dependent value. Show `lock_end_time` with timezone, but do not treat it as an unlock date for a non-decaying position.
4. **Let the user choose** which veHaedal object to operate on (or take the first entry if only one exists). Use the selected `objectId` as the `vehaedalObj` parameter for the subsequent call.

Flow summary: user triggers a method that requires `vehaedalObj` → ask if they have the object ID → if not, call get_vehaedal_list(address) → present the list → user selects one → take that `objectId` as `vehaedalObj` → call the target method.

Apply the same ownership and operation-specific eligibility checks when the user supplies an object ID directly. An empty list means no existing position was found. `unstake_and_claim` consumes the position and returns its locked HAEDAL principal; distribution rewards require a separate claim.

## Base URL

`https://skillsapi.haedal.xyz/api/v1/vehaedal`

## Request body

| Method                   | Required fields                                | Notes |
|--------------------------|------------------------------------------------|-------|
| add_stake                | signerAddress, amount, lockWeeks, isDecaying   | New position; 1–52 weeks; positive amount meeting pool minimum |
| get_vehaedal_list        | address                                        | Query veHaedal objects owned by this address; returns `list` of objects |
| add_to_existing_stake    | signerAddress, **vehaedalObj**, amount          | Requires an existing veHaedal object ID |
| extend_existing_lock     | signerAddress, **vehaedalObj**, additionalWeeks | Requires an existing veHaedal object ID |
| start_decay              | signerAddress, **vehaedalObj**                  | Requires an existing veHaedal object ID |
| stop_decay               | signerAddress, **vehaedalObj**                  | Requires an existing veHaedal object ID |
| unstake_and_claim        | signerAddress, **vehaedalObj**                  | Decaying position eligible for redemption; returns principal |
| claim_rewards_v2         | signerAddress, periods                         | Separate distribution claim; select eligible periods and verify reward coin |
| claim_rewards_v2_epoch_1 | signerAddress                                  | Claim haSUI rewards for fixed period 1 |

## curl examples

**add_stake**

This example uses decay mode, so the lock counts down. Use `false` only when the user intends to freeze the countdown as described above.

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/vehaedal/add_stake" \
  -H "Content-Type: application/json" \
  -d '{"signerAddress":"0xYOUR_ADDRESS","amount":"20","lockWeeks":4,"isDecaying":true}'
```

**get_vehaedal_list** (query veHaedal objects owned by an address, used to obtain vehaedalObj)

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/vehaedal/get_vehaedal_list" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS"}'
```

**add_to_existing_stake** (requires vehaedalObj, obtainable from get_vehaedal_list → list[].objectId)

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/vehaedal/add_to_existing_stake" \
  -H "Content-Type: application/json" \
  -d '{"signerAddress":"0xYOUR_ADDRESS","vehaedalObj":"0xOBJECT_ID","amount":"10"}'
```

**extend_existing_lock** (requires vehaedalObj)

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/vehaedal/extend_existing_lock" \
  -H "Content-Type: application/json" \
  -d '{"signerAddress":"0xYOUR_ADDRESS","vehaedalObj":"0xOBJECT_ID","additionalWeeks":2}'
```

**start_decay** (requires vehaedalObj)

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/vehaedal/start_decay" \
  -H "Content-Type: application/json" \
  -d '{"signerAddress":"0xYOUR_ADDRESS","vehaedalObj":"0xOBJECT_ID"}'
```

**stop_decay** (requires vehaedalObj)

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/vehaedal/stop_decay" \
  -H "Content-Type: application/json" \
  -d '{"signerAddress":"0xYOUR_ADDRESS","vehaedalObj":"0xOBJECT_ID"}'
```

**unstake_and_claim** (requires vehaedalObj)

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/vehaedal/unstake_and_claim" \
  -H "Content-Type: application/json" \
  -d '{"signerAddress":"0xYOUR_ADDRESS","vehaedalObj":"0xOBJECT_ID"}'
```

**claim_rewards_v2**

Each period ID is a decimal string in the u32 range (0..4294967295). The period IDs below illustrate the request format only. Obtain eligible unclaimed periods for the signer, check reward coin type and pause state, and replace this array; do not assume periods 1 and 2 are claimable.

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/vehaedal/claim_rewards_v2" \
  -H "Content-Type: application/json" \
  -d '{"signerAddress":"0xYOUR_ADDRESS","periods":["1","2"]}'
```

**claim_rewards_v2_epoch_1**

This route claims haSUI rewards for fixed period 1 using the configured Rewards distributor. Select an eligible account and verify the returned transaction.

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/vehaedal/claim_rewards_v2_epoch_1" \
  -H "Content-Type: application/json" \
  -d '{"signerAddress":"0xYOUR_ADDRESS"}'
```

## Response

- **add_stake / add_to_existing_stake / extend_existing_lock / start_decay / stop_decay / unstake_and_claim / claim_rewards_v2 / claim_rewards_v2_epoch_1**: on HTTP 200, the body is `{"txBytes":"<base64>"}`; use `jq -r '.txBytes'` to extract it. On non-200, parse `.msg` if the body is valid JSON; otherwise return the plain-text body with the HTTP status.
- **get_vehaedal_list**: on HTTP 200, the body is `{"list":[{"objectId":"...","fields":{...}}, ...]}`. Use `jq -r '.list'` to obtain the list, then take the desired entry's `objectId` as the `vehaedalObj` for subsequent calls.

## MoveAbort error codes

Use the following mapping only for this package's `staking` module, after checking package, module and function. Reward-distribution and `staking_pool` errors have different mappings; see the contract rules. In particular, rewards code 9 means `EPauseClaim`, not `ELockExpired`.

| Code | Constant Name | Description |
|------|---------------|-------------|
| 0 | `N/A` | `start_decay`: token is already in decaying state. `stop_decay` / `unstake_and_claim`: token is not in decaying state. |
| 1 | `EInvalidLockDuration` | Invalid duration, insufficient full weeks for add/stop-decay, or extension exceeds 52 remaining full weeks |
| 2 | `EInvalidOwner` | Caller is not the owner of the veHAEDAL token |
| 3 | `ENoTokensToUnstake` | No tokens available to unstake (`unstaked amount` is 0) |
| 4 | `EInvalidAmount` | Invalid amount, must be greater than 0 |
| 5 | `ELockNotExpired` | Redemption is earlier than the UTC day containing lock_end_time |
| 6 | `ELockShouldBeExpired` | Lock period should be expired |
| 7 | `EMinStakeAmount` | Stake amount is below the minimum required |
| 8 | `EDataNotMatchProgram` | Pool version exceeds the program version; check is pool.version <= current_version |
| 9 | `ELockExpired` | Lock period has already expired |
