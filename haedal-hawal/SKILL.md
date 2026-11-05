---
name: haedal-hawal
description: |
  Use when the user mentions staking WAL, withdrawing hawal, claiming rewards, or operating haWAL on Haedal Protocol (a DeFi/LST protocol on SUI).
  Trigger phrases: “stake hawal”,"stake hawal", "withdraw hawal", "withdraw wal","instant withdraw hawal","instant withdraw wal","claim hawal rewards","claim wal rewards" .
  This skill calls https://skillsapi.haedal.xyz/api/v1/hawal/* via curl POST. On HTTP 200 returns txBytes (base64); on non‑200 returns msg.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(curl:*), Bash(jq:*)
license: MIT
metadata:
  author: haedal
  version: '0.1.2'
---

# Haedal haWAL

**Haedal Protocol** is a DeFi ecosystem on the [SUI](https://sui.io) blockchain. **haWAL** is its liquid staking token (LST) for WAL: stake WAL to receive haWAL, earn staking rewards, and optionally choose validators. This skill calls the Haedal Skills API to perform stake, withdraw, instant withdraw, and claim operations via curl — no custom scripts required.

Call the haWAL HTTP APIs directly with curl.

**API request `amount` is a human-readable decimal string**: send `"amount":"20"` for 20 tokens, without multiplying by decimals. `stake` spends WAL; `withdraw` and `withdraw_instant` spend haWAL. Object response amounts are raw integer units; epoch numbers and millisecond timestamps are not token amounts.

## Contract rules and transaction handoff

Read [contract rules](references/contract-rules.md) before preparing a transaction. They cover minimum amounts, fees, Walrus epochs, ticket maturity. Use current chain state for balances, fees, pauses and eligibility.

HTTP 200 with `txBytes` means an unsigned transaction was prepared. Before handing it to a signing wallet, decode and check the sender, network, Move targets, input asset/amount, recipient and expected asset changes, then run an unsigned checked simulation. If the required tooling or state is unavailable, report that verification is incomplete and do not describe the transaction as ready to sign. Signing and broadcasting require the user's authorization.

## Validator when staking

Only the **stake** method requires a `validator` (validator node `node_id`):

- Passing **`0x0`** means "use the default validator".
- If the user **does not provide a validator**: show a reference list of validators and let the user choose. The user can respond with **index 1–10** or the **validator name**. Map this to the corresponding `node_id` from the table and include it in the request.
- The reference list is defined in [`references/validators.md`](references/validators.md) (mapping between name and node_id, indexed 1–10).
- Treat this as a static reference, not proof of current validator availability. Validate the selected Walrus node/pool ID against current Walrus state. `0x0` requests automatic validator selection; confirm that an explicit nonzero ID is currently available before using it.

Typical flow: the user says "stake hawal" without specifying a validator → present 10 reference validators (index + name) → the user replies with "1" or "Nansen", etc. → use the corresponding `node_id` to call stake.

## Claiming delayed redemption and NFTObj

The **claim** endpoint requires **address** and **NFTObj**, the `walstaking::UnstakeTicket` ID created by a delayed withdrawal. It redeems the ticket's WAL amount and consumes the ticket; it is not a separate staking-reward claim. If the user says "claim hawal rewards", clarify whether they mean an existing delayed redemption before using this flow.

1. **Ask the user** whether they want help finding the object id (if the user already knows the NFTObj, you can send it directly).
2. **If discovery is needed**: call **get_unstake_tickets_list** with the user `address`. On HTTP 200 the response body is:
   ```json
   {
     "list": [
       {
         "objectId": "0x...",
         "type": "0x...::walstaking::UnstakeTicket",
         "version": "...",
         "fields": {
           "claim_epoch": 27,
           "claim_timestamp_ms": "1774364404744",
           "hawal_amount": "700000000000",
           "id": { "id": "0x..." },
           "unstake_timestamp_ms": "1772871784320",
           "wal_amount": "700635659164"
         }
       }
     ]
   }
   ```
3. **Validate and present tickets**: check the type and current ownership, display raw amounts alongside decimal-formatted amounts, and show `claim_epoch` and `claim_timestamp_ms` with timezone. A listed ticket is not necessarily mature: the current Walrus epoch must reach `claim_epoch` AND Clock's timestamp must reach `claim_timestamp_ms`. Read the epoch from current Walrus state; its business `version` is not an epoch. Also check claim pause state; do not substitute Sui epoch or build a knowingly premature claim.
4. **If there are multiple entries in the list**: let the user choose which one to claim (or take the first entry if only one exists) and use that entry's `objectId` as NFTObj.
5. **Then call claim**: `POST /api/v1/hawal/claim` with body `{"address":"0x...","NFTObj":"<objectId from previous step>"}`.

Flow summary: delayed redemption requested → obtain the ticket ID → validate type, ownership and maturity → user selects a ticket if needed → call claim(address, NFTObj). Apply these checks even when the user supplies NFTObj directly. An empty list means there is no ticket to claim.

## Base URL

`https://skillsapi.haedal.xyz/api/v1/hawal`

## Request body

| Method                   | Required fields           | Notes |
|--------------------------|---------------------------|-------|
| stake                    | address, amount, validator | Spend WAL; minimum 1 WAL |
| withdraw                 | address, amount           | Spend haWAL; create delayed-redemption ticket |
| withdraw_instant         | address, amount           | Spend haWAL; receive WAL after configured service fee |
| claim                    | address, **NFTObj**       | Redeem a mature UnstakeTicket for WAL |
| get_unstake_tickets_list | address                   | Query the UnstakeTicket list for this address to obtain NFTObj |

## curl examples

**stake**

Pass `validator` as `0x0` to use the default validator, or use a node_id from [`references/validators.md`](references/validators.md) (user can choose by index 1–10 or by name).

```bash
# Use the default validator
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hawal/stake" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS","amount":"100","validator":"0x0"}'

# Or specify an explicit validator node_id (for example Nansen)
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hawal/stake" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS","amount":"100","validator":"0x7b3ba6de2ae58283f60d5b8dc04bb9e90e4796b3b2e0dea75569f491275242e7"}'
```

**withdraw**

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hawal/withdraw" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS","amount":"100"}'
```

**withdraw_instant**

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hawal/withdraw_instant" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS","amount":"100"}'
```

**get_unstake_tickets_list** (discover UnstakeTickets; independently check maturity before claiming)

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hawal/get_unstake_tickets_list" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS"}'
```

When the response status is 200, the body contains `{"list":[{"objectId":"...","type":"...","version":"...","fields":{...}}, ...]}`. Present key fields (`objectId`, `hawal_amount`, `wal_amount`, `claim_epoch`, `claim_timestamp_ms`) to the user, then use the selected `list[].objectId` as the **NFTObj** for the claim call.

**claim** (requires NFTObj, which can be obtained from get_unstake_tickets_list → list[].objectId)

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hawal/claim" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS","NFTObj":"0xOBJECT_ID_FROM_LIST"}'
```

## Response

- **stake / withdraw / withdraw_instant / claim**: on HTTP 200, the body is `{"txBytes":"<base64>"}`; use `jq -r '.txBytes'` to extract it. On non-200, parse `.msg` if the body is valid JSON; otherwise return the plain-text body with the HTTP status.
- **get_unstake_tickets_list**: on HTTP 200, the body is `{"list":[{"objectId","type","version","fields":{...}}, ...]}`; use `jq -r '.list'` to obtain the list. Each entry's `fields` contains `hawal_amount`, `wal_amount`, `claim_epoch`, `claim_timestamp_ms`, `unstake_timestamp_ms`. Take the desired entry's `objectId` as the NFTObj for claim.

## MoveAbort error codes

Use the following mapping only for errors from this package's `walstaking` module, after checking package, module and function. Walrus and other dependency/module errors may reuse the same numbers; preserve the full abort location for diagnosis.

| Code | Constant Name | Description |
|------|---------------|-------------|
| 1 | `EDataNotMatchProgram` | Data version does not match the current program version |
| 2 | `EStakeNotEnoughWal` | Stake amount is below the minimum staking threshold (1 WAL) |
| 3 | `EStakeNoHaWalMint` | Calculated haWAL mint amount is zero |
| 4 | `EUnstakeNormalTicketLocking` | Unstake ticket is still locked (claim epoch/time not reached) |
| 5 | `EUnstakeExceedMaxWalAmount` | Unstake WAL amount exceeds total available WAL |
| 6 | `EUnstakeNotZeroHawal` | Unstake input haWAL amount must not be zero |
| 7 | `EStakePause` | Staking is currently paused |
| 8 | `EUnstakePause` | Unstaking is currently paused |
| 9 | `EUnstakeNeedAmountIsNotZero` | Unable to obtain the full WAL redemption amount |
| 10 | `EClaimPause` | Claim is currently paused |
| 11 | `EValidatorCountNotMatch` | Provided validator count does not match existing validator count |
| 12 | `EValidatorNotFound` | Validator not found in the validator list |
| 13 | `EStakeActiveValidatorsNotFound` | No active validators found for staking |
| 14 | `EUnstakeOutOfRange` | Unstake timestamp is out of the valid epoch time range |
| 15 | `EStakeActiveValidatorsIsNull` | Active validators list is empty |
| 16 | `EUnstakeExceedMinWalAmount` | Unstake WAL amount is below the minimum staking threshold |
