# Crypto Crush — Proposed API Reference

## Overview

This document defines a proposed REST API for communication between the Crypto Crush mobile client and backend services.

> **Status:** Proposed API specification. The endpoints, schemas, and authentication model documented here are not confirmed production implementations.

## Base URL

Example development base URL:

```text
https://api.example.com/v1
```

The production API domain should be determined during implementation.

---

## Authentication

Protected endpoints require an authenticated player session.

Example authorization header:

```http
Authorization: Bearer <access_token>
Content-Type: application/json
```

Requests without valid authorization should return an authentication error.

---

# 1. Player Profile

## Get Player Profile

```http
GET /players/{player_id}
```

Returns information associated with the authenticated player.

### Example Response

```json
{
  "player_id": "player_1024",
  "username": "cryptoplayer",
  "current_level": 42,
  "reward_balance_usd": 1.75,
  "wallet_connected": true
}
```

### Possible Responses

| Status | Meaning |
|---|---|
| 200 | Player returned successfully |
| 401 | Authentication required |
| 403 | Player is not authorized to access the resource |
| 404 | Player not found |

---

# 2. Game Sessions

## Create Game Session

```http
POST /game-sessions
```

Creates a unique game session before an eligible level begins.

### Example Request

```json
{
  "level_id": 43
}
```

### Example Response

```json
{
  "session_id": "session_84f21",
  "level_id": 43,
  "status": "active",
  "started_at": "2026-10-01T14:30:00Z"
}
```

---

## Complete Game Session

```http
POST /game-sessions/{session_id}/complete
```

Submits a level-completion event for validation.

### Example Request

```json
{
  "score": 18450
}
```

### Example Response

```json
{
  "session_id": "session_84f21",
  "status": "validated",
  "reward_eligible": true,
  "reward_amount_usd": 0.005
}
```

A game session should not generate the same reward more than once.

---

# 3. Rewards

## Get Reward Balance

```http
GET /rewards/balance
```

Returns the authenticated player's reward balance and withdrawal eligibility.

### Example Response

```json
{
  "available_balance_usd": 5.35,
  "minimum_withdrawal_usd": 5.00,
  "withdrawal_eligible": true
}
```

---

## Get Reward History

```http
GET /rewards/history
```

Returns reward transactions associated with the authenticated player.

### Example Response

```json
{
  "rewards": [
    {
      "reward_id": "reward_501",
      "session_id": "session_84f21",
      "amount_usd": 0.005,
      "status": "credited",
      "created_at": "2026-10-01T14:34:12Z"
    }
  ]
}
```

---

# 4. Wallets

## Register Wallet

```http
POST /wallets
```

Associates a Solana wallet address with the authenticated player.

Registering an address should not by itself prove wallet ownership. Production implementation should use an appropriate wallet-signature verification flow.

### Example Request

```json
{
  "network": "solana",
  "wallet_address": "<solana_wallet_address>"
}
```

### Example Response

```json
{
  "wallet_id": "wallet_204",
  "network": "solana",
  "wallet_address": "<solana_wallet_address>",
  "verification_status": "pending"
}
```

Private keys or seed phrases must never be submitted through this endpoint.

---

# 5. Withdrawals

## Create Withdrawal Request

```http
POST /withdrawals
```

Creates a withdrawal request for an eligible player.

### Example Request

```json
{
  "amount_usd": 5.00,
  "asset": "USDT",
  "network": "solana",
  "wallet_id": "wallet_204"
}
```

### Validation

Before accepting the request, the system should verify:

- Player authentication
- Available reward balance
- Minimum withdrawal threshold
- Wallet verification status
- Supported asset
- Supported network
- Absence of conflicting withdrawal requests

### Example Response

```json
{
  "withdrawal_id": "wd_9021",
  "amount_usd": 5.00,
  "asset": "USDT",
  "network": "solana",
  "status": "pending"
}
```

---

## Get Withdrawal Status

```http
GET /withdrawals/{withdrawal_id}
```

Returns the current status of a withdrawal.

### Example Response

```json
{
  "withdrawal_id": "wd_9021",
  "status": "confirmed",
  "transaction_signature": "<solana_transaction_signature>"
}
```

### Proposed Withdrawal States

```text
pending
approved
processing
confirmed
failed
rejected
```

---

# 6. Leaderboard

## Get Leaderboard

```http
GET /leaderboard
```

Returns ranked player data.

### Optional Query Parameters

| Parameter | Description |
|---|---|
| period | Ranking period such as `weekly` or `monthly` |
| limit | Maximum number of results |

### Example

```http
GET /leaderboard?period=weekly&limit=20
```

### Example Response

```json
{
  "period": "weekly",
  "players": [
    {
      "rank": 1,
      "username": "player_one",
      "completed_levels": 125
    }
  ]
}
```

---

# Standard Error Response

Errors should use a consistent response structure.

```json
{
  "error": {
    "code": "INSUFFICIENT_REWARD_BALANCE",
    "message": "The available reward balance is below the requested withdrawal amount."
  }
}
```

## Example Error Codes

| Code | Description |
|---|---|
| AUTH_REQUIRED | Valid authentication is required |
| INVALID_GAME_SESSION | Game session could not be validated |
| REWARD_ALREADY_CLAIMED | Reward has already been processed |
| INVALID_WALLET | Wallet information is invalid |
| WALLET_NOT_VERIFIED | Wallet ownership has not been verified |
| BELOW_WITHDRAWAL_THRESHOLD | Balance does not meet the withdrawal threshold |
| INSUFFICIENT_REWARD_BALANCE | Requested amount exceeds available balance |
| WITHDRAWAL_ALREADY_PROCESSING | Another conflicting withdrawal is being processed |
| BLOCKCHAIN_TRANSACTION_FAILED | Web3 transaction could not be completed |

---

## API Design Notes

This proposed API separates gameplay operations from blockchain settlement.

The backend remains responsible for validating game activity and maintaining the reward ledger, while blockchain-specific processing occurs through the Web3 integration layer.

Final endpoint design, authentication mechanisms, rate limits, anti-cheat controls, schemas, and production security requirements require engineering and security review.
