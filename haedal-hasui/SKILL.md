---
name: haedal-hasui
description: |
  Use when the user mentions staking SUI, withdrawing, claiming rewards, or operating haSUI on Haedal Protocol (a DeFi/LST protocol on SUI).
  Trigger phrases: "haedal","stake sui"，"stake hasui", "withdraw hasui", "withdraw sui"，"instant withdraw hasui", "instant withdraw sui", "claim hasui rewards","claim sui rewards".
  This skill calls https://skillsapi.haedal.xyz/api/v1/hasui/* via curl POST. On HTTP 200 returns txBytes (base64); on non‑200 returns msg.
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(curl:*), Bash(jq:*)
license: MIT
metadata:
  author: haedal
  version: '0.1.2'
---

# Haedal haSUI

**Haedal Protocol** is a DeFi ecosystem on the [SUI](https://sui.io) blockchain. **haSUI** is its liquid staking token (LST): stake native SUI to receive haSUI, earn staking rewards, and optionally choose validators. This skill calls the Haedal Skills API to perform stake, withdraw, instant withdraw, and claim operations via curl — no custom scripts required.

Call the haSUI HTTP APIs directly with curl.

**API request `amount` is a human-readable decimal string**: send `"amount":"20"` for 20 tokens, without multiplying by decimals. `stake` spends SUI; `withdraw` and `withdraw_instant` spend haSUI. Object response amounts are raw integer units; epoch numbers and millisecond timestamps are not token amounts.

## Contract rules and transaction handoff

Read [contract rules](references/contract-rules.md) before preparing a transaction. They cover minimum amounts, fees, ticket maturity. Use current chain state for balances, fees, pauses and eligibility.

HTTP 200 with `txBytes` means an unsigned transaction was prepared. Before handing it to a signing wallet, decode and check the sender, network, Move targets, input asset/amount, recipient and expected asset changes, then run an unsigned checked simulation. If the required tooling or state is unavailable, report that verification is incomplete and do not describe the transaction as ready to sign. Signing and broadcasting require the user's authorization.

## Validator when staking

Only the **stake** method requires a `validator` (validator node `node_id`):

- Passing **`0x0`** means "use the default validator".
- If the user **does not provide a validator**: show a reference list of validators and let the user choose. The user can respond with **index 1–5** or the **validator name**. Map this to the corresponding `node_id` in the table and include it in the request.
- The reference list is defined in [`references/validators.md`](references/validators.md) (mapping between name and node_id, indexed 1–5).
- Treat this as a static reference, not proof of current validator availability. Check the selected address against current Sui active validators. The contract uses protocol selection for `0x0` or an inactive address; disclose a fallback instead of claiming the requested validator was used.

Typical flow: the user says "stake hasui" without specifying a validator → present 5 reference validators (index + name) → the user replies with "1" or "Ankr", etc. → use the corresponding `node_id` to call stake.

## Claiming delayed redemption and NFTObj

The **claim** endpoint requires **address** and **NFTObj**, the `staking::UnstakeTicket` ID created by a delayed withdrawal. It redeems the ticket's SUI amount and consumes the ticket; it is not a separate staking-reward claim. If the user says "claim hasui rewards", clarify whether they mean an existing delayed redemption before using this flow.

1. **Ask the user** whether they want help finding the object id (if the user already knows the NFTObj, you can send it directly).
2. **If discovery is needed**: call **get_unstake_tickets_list** with the user `address`. On HTTP 200 the response body is:
   ```json
   {
     "list": [
       {
         "objectId": "0x...",
         "type": "0x...::staking::UnstakeTicket",
         "version": "...",
         "fields": {
           "claim_epoch": "646",
           "claim_timestamp_ms": "1737135496590",
           "id": { "id": "0x..." },
           "st_amount": "1000000000",
           "sui_amount": "1045887315",
           "unstake_timestamp_ms": "1737085604741"
         }
       }
     ]
   }
   ```
3. **Validate and present tickets**: check the type and current ownership, display raw amounts alongside decimal-formatted amounts, and show `claim_epoch` and `claim_timestamp_ms` with timezone. A listed ticket is not necessarily mature: current Sui epoch must reach `claim_epoch` AND its epoch-start timestamp must reach `claim_timestamp_ms`. Also check claim pause state; do not build a knowingly premature claim.
4. **If there are multiple entries in the list**: let the user choose which one to claim (or take the first entry if only one exists) and use that entry's `objectId` as NFTObj.
5. **Then call claim**: `POST /api/v1/hasui/claim` with body `{"address":"0x...","NFTObj":"<objectId from previous step>"}`.

Flow summary: delayed redemption requested → obtain the ticket ID → validate type, ownership and maturity → user selects a ticket if needed → call claim(address, NFTObj). Apply these checks even when the user supplies NFTObj directly. An empty list means there is no ticket to claim.

## Base URL

`https://skillsapi.haedal.xyz/api/v1/hasui`

## Request body

| Method                   | Required fields           | Notes |
|--------------------------|---------------------------|-------|
| stake                    | address, amount, validator | Spend SUI; minimum 1 SUI |
| withdraw                 | address, amount           | Spend haSUI; create delayed-redemption ticket |
| withdraw_instant         | address, amount           | Spend haSUI; receive SUI after configured service fee |
| claim                    | address, **NFTObj**       | Redeem a mature UnstakeTicket for SUI |
| get_unstake_tickets_list | address                   | Query the UnstakeTicket list for this address to obtain NFTObj |

## curl examples

**stake**

Pass `validator` as `0x0` to use the default validator, or use a node_id from [`references/validators.md`](references/validators.md) (user can choose by index 1–5 or by name).

```bash
# Use the default validator
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hasui/stake" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS","amount":"100","validator":"0x0"}'

# Or specify an explicit validator node_id (for example Haedal Protocol)
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hasui/stake" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS","amount":"100","validator":"0xc8a57a7ae3b814afc15a907d963a288454b2c0f1a323fd556cb2d56d85a94583"}'
```

**withdraw**

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hasui/withdraw" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS","amount":"100"}'
```

**withdraw_instant**

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hasui/withdraw_instant" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS","amount":"100"}'
```

**get_unstake_tickets_list** (discover UnstakeTickets; independently check maturity before claiming)

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hasui/get_unstake_tickets_list" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS"}'
```

When the response status is 200, the body contains `{"list":[{"objectId":"...","type":"...","version":"...","fields":{...}}, ...]}`. Present key fields (`objectId`, `st_amount`, `sui_amount`, `claim_epoch`, `claim_timestamp_ms`) to the user, then use the selected `list[].objectId` as the **NFTObj** for the claim call.

**claim** (requires NFTObj, which can be obtained from get_unstake_tickets_list → list[].objectId)

```bash
curl -s -w "\n%{http_code}" -X POST "https://skillsapi.haedal.xyz/api/v1/hasui/claim" \
  -H "Content-Type: application/json" \
  -d '{"address":"0xYOUR_ADDRESS","NFTObj":"0xOBJECT_ID_FROM_LIST"}'
```

## Response

- **stake / withdraw / withdraw_instant / claim**: on HTTP 200, the body is `{"txBytes":"<base64>"}`; use `jq -r '.txBytes'` to extract it. On non-200, parse `.msg` if the body is valid JSON; otherwise return the plain-text body with the HTTP status.
- **get_unstake_tickets_list**: on HTTP 200, the body is `{"list":[{"objectId","type","version","fields":{...}}, ...]}`; use `jq -r '.list'` to obtain the list. Each entry's `fields` contains `st_amount`, `sui_amount`, `claim_epoch`, `claim_timestamp_ms`, `unstake_timestamp_ms`. Take the desired entry's `objectId` as the NFTObj for claim.

## MoveAbort error codes

Use the following mapping only for errors from this package's `staking` module, after checking package, module and function. Dependency/module errors may reuse the same numbers.

| Code | Constant Name | Description |
|------|---------------|-------------|
| 1 | `EDataNotMatchProgram` | Data does not match the program |
| 2 | `EStakedSuiRewardsNotMatched` | Staked SUI rewards do not match |
| 3 | `EInvalidStakeParas` | Invalid staking parameters |
| 4 | `EStakeNotEnoughSui` | Input is below the 1 SUI staking minimum |
| 5 | `EStakeNoStsuiMint` | No stSUI minted during staking |
| 6 | `EUnstakeNormalTicketLocking` | Normal unstake ticket is still in locking period |
| 7 | `EUnstakeNotEnoughSui` | Not enough SUI to fulfill unstake |
| 8 | `EUnstakeExceedMaxSuiAmount` | Gross SUI equivalent is below 1 SUI or exceeds total protocol SUI |
| 11 | `EUnstakeNotZeroStSui` | Unstake stSUI amount must not be zero |
| 12 | `EStakePause` | Staking is paused |
| 13 | `EUnstakePause` | Unstaking is paused |
| 16 | `EUnstakeNeedAmountIsNotZero` | Unable to obtain the full SUI redemption amount |
| 17 | `EClaimPause` | Claiming is paused |
| 18 | `EValidatorCountNotMatch` | Validator count does not match |
| 19 | `EValidatorNotFound` | Validator not found |
| 20 | `EInjectRewardsTooLow` | Injected rewards amount is too low |
