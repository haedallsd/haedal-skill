---
name: haedal-hawal
description: Use this skill when the user asks to "stake hawal", "withdraw hawal", "instant withdraw hawal", or "claim hawal rewards". The haWAL module calls /api/v1/hawal/* via curl POST, sending only address, amount (and validator). On HTTP 200 it returns txBytes; on non‑200 it returns msg.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(curl:*), Bash(jq:*)
model: opus
license: MIT
metadata:
  author: haedal
  version: '0.1.0'
---

# Haedal haWAL

Call the haWAL HTTP APIs directly with curl, without any custom scripts.

**All numeric parameters are human‑readable amounts**: pass `amount` and similar quantity fields as human‑readable values, without multiplying by decimals. For example, to stake 20 tokens, simply send `"amount":"20"`.

## Validator when staking

Only the **stake** method requires a `validator` (validator node `node_id`):

- Passing **`0x0`** means "use the default validator".
- If the user **does not provide a validator**: show a reference list of validators and let the user choose. The user can respond with **index 1–10** or the **validator name**. Map this to the corresponding `node_id` from the table and include it in the request.
- The reference list is defined in [`references/validators.md`](references/validators.md) (mapping between name and node_id, indexed 1–10).

Typical flow: the user says "stake hawal" without specifying a validator → present 10 reference validators (index + name) → the user replies with "1" or "Nansen", etc. → use the corresponding `node_id` to call stake.

## Claiming rewards and NFTObj

The **claim** endpoint requires **address** and **NFTObj** (the NFT object ID that holds the rewards). When the user says "claim hawal rewards":

1. **Ask the user** whether they want help finding the object id (if the user already knows the NFTObj, you can send it directly).
2. **If discovery is needed**: call **get_unstake_tickets_list** with the user `address`. On HTTP 200 the response body is `{"list":[{"objectId":"...","type":"...","version":"..."}, ...]}`. Use the `objectId` from `list` as **NFTObj**.
3. **If there are multiple entries in the list**: let the user choose which one to claim (or follow a predefined rule such as "take the first entry") and use that entry's `objectId` as NFTObj.
4. **Then call claim**: `POST /api/v1/hawal/claim` with body `{"address":"0x...","NFTObj":"<objectId from previous step>"}`.

Flow summary: the user says "claim hawal rewards" → ask whether to discover the object id → if yes, call get_unstake_tickets_list(address) → present the list (or let the user select one entry) → take the chosen `objectId` as NFTObj → call claim(address, NFTObj).

## Base URL

`https://skillsapi.haedal.xyz/api/v1/hawal`

## Request body

| Method                   | Required fields           | Notes |
|--------------------------|---------------------------|-------|
| stake                    | address, amount, validator | No object field |
| withdraw                 | address, amount           | No object field |
| withdraw_instant         | address, amount           | No object field |
| claim                    | address, **NFTObj**       | Requires NFT object ID, see the "Claiming rewards" flow above |
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

**get_unstake_tickets_list** (query the UnstakeTicket list that can be claimed, used to obtain NFTObj)

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hawal/get_unstake_tickets_list" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS"}'
```

When the response status is 200, the body looks like: `{"list":[{"objectId":"0x...","type":"...","version":"..."}, ...]}`. Use `list[].objectId` as the **NFTObj** for the claim call.

**claim** (requires NFTObj, which can be obtained from get_unstake_tickets_list → list[].objectId)

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hawal/claim" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS","NFTObj":"0xOBJECT_ID_FROM_LIST"}'
```

## Response

- **stake / withdraw / withdraw_instant / claim**: on HTTP 200, the body is `{"txBytes":"<base64>"}`; use `jq -r '.txBytes'` to extract it. On non‑200, use `jq -r '.msg'` to return the error reason to the user.
- **get_unstake_tickets_list**: on HTTP 200, the body is `{"list":[{"objectId","type","version"}, ...]}`; use `jq -r '.list'` to obtain the list, then take each entry's `objectId` as the NFTObj for claim.

## MoveAbort error codes

When a dry-run or build call fails with `MoveAbort(..., <code>)`, use the following mapping:

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
| 9 | `EUnstakeNeedAmountIsNotZero` | Unstake process failed: remaining `need_amount` is not zero |
| 10 | `EClaimPause` | Claim is currently paused |
| 11 | `EValidatorCountNotMatch` | Provided validator count does not match existing validator count |
| 12 | `EValidatorNotFound` | Validator not found in the validator list |
| 13 | `EStakeActiveValidatorsNotFound` | No active validators found for staking |
| 14 | `EUnstakeOutOfRange` | Unstake timestamp is out of the valid epoch time range |
| 15 | `EStakeActiveValidatorsIsNull` | Active validators list is empty |
| 16 | `EUnstakeExceedMinWalAmount` | Unstake WAL amount is below the minimum staking threshold |
