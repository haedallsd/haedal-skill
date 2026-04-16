# Haedal Skills

This is a skill pack for AI agents. Its purpose is to help prepare transactions for **Haedal Protocol** use cases.

It currently includes 3 skills:

- `haedal-hasui`: handles haSUI actions such as `stake`, `withdraw`, `instant withdraw`, and `claim`
- `haedal-hawal`: handles haWAL actions such as `stake`, `withdraw`, `instant withdraw`, and `claim`
- `haedal-vehaedal`: handles veHAEDAL actions such as locking, adding to a position, extending lock time, claiming rewards, and decay control

## The Most Important Thing First

**These 3 skills are used to help an agent obtain the `tx bytes` needed for a transaction.**

They are **not wallets**, and they **do not sign transactions by themselves**.

If you want to actually complete an on-chain transaction, you will need:

1. An agent that supports skills, such as **OpenClaw**, **Hermes**, and others
2. The Haedal skills from this repository installed in that agent
3. A **signing wallet skill**
4. Common wallet skills such as **OKX Wallet** and **ClawWallet**

You can think of the flow like this:

`Haedal skill prepares the transaction data` -> `wallet skill signs it` -> `the transaction is completed on-chain`

Without a wallet skill for signing, the agent can only help prepare the transaction, but cannot actually complete it.

## Who This Is For

This project is for you if you want to use natural language to ask an agent to do things like:

- "Stake 100 SUI into haSUI"
- "Redeem 50 haWAL"
- "Lock HAEDAL for 52 weeks"
- "Claim my Haedal rewards"

## How To Use

### Step 1: Prepare an agent

You need an agent that supports skills.

For example:

- OpenClaw
- Hermes
- Codex
- Cursor

If you do not have an agent ready yet, prepare that first and then come back to this project.

### Step 2: Install the Haedal skills through conversation

These 3 Haedal skills can be installed by simply talking to your agent.

You can say:

- `read https://github.com/haedallsd/haedal-skill/blob/main/haedal-hasui/SKILL.md and set up`
- `read https://github.com/haedallsd/haedal-skill/blob/main/haedal-hawal/SKILL.md and set up`
- `read https://github.com/haedallsd/haedal-skill/blob/main/haedal-vehaedal/SKILL.md and set up`

If you want to know which skills are available in this repo, you can just ask the agent.

### Step 3: Install and connect a signing wallet skill

If your goal is to actually complete transactions, this step is essential.

Common wallet skills include:

- OKX Wallet
- ClawWallet

`OKX Wallet` and `ClawWallet` can also be installed as skills.

You can say:

- `read https://www.clawwallet.cc/skills/SKILL.md and set up`
- `read https://github.com/okx/onchainos-skills/blob/main/skills/okx-agentic-wallet/SKILL.md and set up`

If this step is missing or incomplete, the usual result is:

- the agent understands your request
- the agent can generate `tx bytes`
- but the transaction cannot actually be signed and sent

### Step 4: Use natural language commands

After setup, you can directly say things like:

- `Stake 100 SUI into haSUI`
- `Redeem 50 haWAL`
- `Stake HAEDAL locked for 52 weeks`
- `Claim my Haedal rewards`

If the action needs more information, the agent will usually ask follow-up questions such as:

- wallet address
- which validator to use
- the `NFTObj` needed for reward claiming
- the object ID needed for veHAEDAL actions

This is normal and does not mean something is wrong.

## What Each Skill Does

### `haedal-hasui`

Best for:

- staking SUI into haSUI
- withdrawing or instant withdrawing
- claiming related rewards

### `haedal-hawal`

Best for:

- staking WAL into haWAL
- withdrawing or instant withdrawing
- claiming related rewards

### `haedal-vehaedal`

Best for:

- locking HAEDAL
- adding more tokens to an existing position
- extending lock duration
- starting or stopping decay
- claiming rewards
- unstaking and claiming after expiry

## FAQ

### 1. Why can't I complete a transaction even though the skill is installed?

The most common reasons are not with the Haedal skill itself, but:

- you do not have a usable agent ready yet
- the agent has not been restarted
- a signing wallet skill has not been installed
- the wallet skill has not been properly authorized to sign for the agent

### 2. What does the skill do, and what does the wallet do?

- skill: calls the Haedal API and helps obtain the `tx bytes` needed for the transaction
- wallet: signs the transaction and sends it on-chain

### 3. Do I need to know how to code?

Not necessarily.

If your agent and wallet are already set up, in most cases you can simply describe what you want, and the agent will guide you through the steps.

## One-Line Summary

Haedal is not a wallet. It is an **agent skill pack for Haedal**.

It helps your agent understand Haedal actions and prepare transaction data. If you want to actually complete an on-chain transaction, you still need to use it together with an **agent + wallet skill for signing**.

## License

MIT
