# Crypto Crush — Proposed Rewards System

## Overview

This document defines the proposed reward-processing model for Crypto Crush.

Under the current product concept, players earn **$0.005 for each eligible completed level** and become eligible to request a withdrawal after reaching a minimum reward balance of **$5.00**.

The proposed architecture keeps frequent gameplay reward accounting off-chain while using the Solana network for eligible USDT withdrawals.

> **Status:** Proposed technical specification. This reward architecture does not represent a confirmed production implementation.

---

## Reward Lifecycle

```text
Player Starts Level
        ↓
Game Session Created
        ↓
Player Completes Level
        ↓
Completion Submitted
        ↓
Backend Validates Session
        ↓
Reward Eligibility Checked
        ↓
$0.005 Reward Created
        ↓
Reward Ledger Updated
        ↓
Player Balance Updated
        ↓
Withdrawal Threshold Checked
```

---

## Reward Value

The current product requirement defines:

```text
Reward per eligible completed level: $0.005
Minimum withdrawal threshold: $5.00
Withdrawal asset: USDT
Settlement network: Solana
```

Reward values should be controlled by backend configuration rather than trusted from values submitted by the mobile client.

---

## Reward Eligibility

A reward should only be created after the corresponding game session has been validated.

Proposed validation checks include:

- Player is authenticated
- Game session exists
- Session belongs to the authenticated player
- Level is eligible for a reward
- Level completion is valid
- Session has not already generated a reward
- Completion event has not already been processed

The mobile client should not have authority to directly increase a player's reward balance.

---

## Reward Record

Each successful reward event should create a traceable ledger entry.

Example:

```json
{
  "reward_id": "reward_501",
  "player_id": "player_1024",
  "session_id": "session_84f21",
  "level_id": 43,
  "amount_usd": 0.005,
  "status": "available",
  "created_at": "2026-10-01T14:34:12Z"
}
```

---

## Reward States

| Status | Description |
|---|---|
| `pending` | Reward is awaiting validation |
| `available` | Validated reward is included in available balance |
| `reserved` | Reward amount has been allocated to an active withdrawal |
| `withdrawn` | Reward value has been successfully settled |
| `rejected` | Reward failed validation |

---

## Player Balance

The player's reward balance should be derived from validated reward and withdrawal records.

A proposed balance response may include:

```json
{
  "available_balance_usd": 5.35,
  "reserved_balance_usd": 0.00,
  "withdrawn_total_usd": 10.00,
  "minimum_withdrawal_usd": 5.00,
  "withdrawal_eligible": true
}
```

### Available Balance

Validated rewards that have not been reserved or withdrawn.

### Reserved Balance

Funds associated with a withdrawal currently being processed.

### Withdrawn Total

Rewards that have already been successfully settled.

---

## Duplicate Reward Prevention

The system must prevent a single eligible game session from generating multiple rewards.

A proposed approach is to maintain a unique relationship between:

```text
game_session_id → reward_id
```

If a completion request is repeated for a session that has already generated a reward, the API should not create another reward.

Example error:

```json
{
  "error": {
    "code": "REWARD_ALREADY_CLAIMED",
    "message": "A reward has already been processed for this game session."
  }
}
```

---

## Withdrawal Eligibility

A player becomes eligible to request a withdrawal when:

```text
available_balance >= minimum_withdrawal_threshold
```

For the current proposed model:

```text
available_balance >= $5.00
```

Eligibility alone should not automatically initiate a blockchain transaction.

The player must explicitly submit a withdrawal request.

---

## Balance Reservation

When an eligible withdrawal request is accepted, the requested reward amount should be reserved before blockchain processing begins.

Example:

```text
Available Balance: $7.00
Withdrawal Request: $5.00

Before processing:
Available: $7.00
Reserved: $0.00

After reservation:
Available: $2.00
Reserved: $5.00
```

This prevents the same balance from being used in multiple simultaneous withdrawals.

---

## Successful Withdrawal

After the USDT transaction is confirmed on Solana:

1. Withdrawal status becomes `confirmed`.
2. Reserved reward value becomes `withdrawn`.
3. Transaction reference is stored.
4. Player withdrawal history is updated.
5. Updated balance becomes available to the client.

---

## Failed Withdrawal

A blockchain transaction may fail or remain unresolved.

The system should not silently remove the player's reward balance.

Depending on the failure state, the system may:

- Keep the amount reserved while transaction status is investigated
- Release the reserved amount back to available balance after confirmed failure
- Retry an eligible transaction safely
- Escalate uncertain transaction states for review

The system must avoid creating duplicate blockchain payments during retries.

---

## Off-Chain Reward Ledger

Routine gameplay rewards are proposed to remain off-chain.

This means completing a level does **not** create a Solana transaction for every $0.005 reward.

Instead:

```text
Gameplay
   ↓
Validated Reward Ledger
   ↓
Accumulated Balance
   ↓
Withdrawal Request
   ↓
Blockchain Settlement
```

This design reduces unnecessary blockchain interactions while retaining Web3 settlement for withdrawals.

---

## Auditability

Reward activity should be traceable from gameplay through settlement.

Relevant identifiers may include:

- Player ID
- Game Session ID
- Level ID
- Reward ID
- Withdrawal ID
- Wallet ID
- Blockchain transaction signature

This allows a reward to be traced from the originating game session to a completed withdrawal.

---

## Security Considerations

The proposed reward system should:

- Never trust reward amounts submitted by the client
- Validate game sessions server-side
- Prevent duplicate reward creation
- Protect reward endpoints with authentication
- Prevent simultaneous spending of the same reward balance
- Record reward and withdrawal state changes
- Apply anti-abuse and anti-cheat controls
- Treat blockchain transaction confirmation separately from transaction submission

---

## Implementation Note

This reward model is a proposed technical design based on the Crypto Crush product concept.

Production reward economics, anti-cheat mechanisms, fraud controls, accounting treatment, withdrawal policies, and blockchain settlement architecture require product, engineering, security, and applicable legal/compliance review before implementation.
