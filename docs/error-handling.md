# Crypto Crush — Proposed Error Handling

## Overview

This document defines the proposed error-handling approach for Crypto Crush across the mobile application, backend APIs, reward system, wallet integration, and Web3 settlement layer.

The objective is to provide predictable error responses while preventing failures from creating incorrect reward balances or duplicate blockchain transactions.

> **Status:** Proposed technical specification. Error codes and recovery behavior require engineering validation before production implementation.

---

## Error Response Format

API errors should follow a consistent structure.

```json
{
  "error": {
    "code": "BELOW_WITHDRAWAL_THRESHOLD",
    "message": "The available reward balance does not meet the minimum withdrawal threshold."
  }
}
```

A production implementation may additionally include:

- Request ID
- Timestamp
- Validation details
- Support reference

Sensitive internal information should not be exposed through client-facing error messages.

---

## HTTP Status Codes

| Status | Usage |
|---|---|
| `400` | Invalid request or validation failure |
| `401` | Authentication required or invalid |
| `403` | Authenticated user is not permitted to perform the operation |
| `404` | Requested resource does not exist |
| `409` | Request conflicts with existing application state |
| `422` | Request is valid in format but cannot be processed |
| `429` | Too many requests |
| `500` | Unexpected server error |
| `503` | Required service temporarily unavailable |

---

# Authentication Errors

## AUTH_REQUIRED

Returned when an operation requires authentication.

```json
{
  "error": {
    "code": "AUTH_REQUIRED",
    "message": "Valid authentication is required."
  }
}
```

## ACCESS_DENIED

Returned when an authenticated player attempts to access a resource or operation they are not permitted to use.

---

# Game Session Errors

## INVALID_GAME_SESSION

Returned when a submitted game session cannot be validated.

Possible causes include:

- Session does not exist
- Session does not belong to the player
- Session is no longer valid
- Completion information fails validation

## SESSION_ALREADY_COMPLETED

Returned when a completion operation is submitted for a session that has already been finalized.

---

# Reward Errors

## REWARD_ALREADY_CLAIMED

Returned when an eligible game session has already generated its reward.

The system must not create another reward record.

## REWARD_NOT_ELIGIBLE

Returned when the submitted game activity does not qualify for a reward.

---

# Wallet Errors

## INVALID_WALLET

Returned when wallet information cannot be validated.

## WALLET_NOT_VERIFIED

Returned when a player attempts a withdrawal using a wallet that has not completed ownership verification.

## INVALID_WALLET_SIGNATURE

Returned when the wallet signature does not successfully verify the ownership challenge.

## WALLET_CHALLENGE_EXPIRED

Returned when the verification challenge has expired.

A new challenge should be generated rather than reusing the expired challenge.

## UNSUPPORTED_NETWORK

Returned when an operation requests a blockchain network not supported by the application.

---

# Withdrawal Errors

## BELOW_WITHDRAWAL_THRESHOLD

Returned when the available balance does not meet the required minimum withdrawal threshold.

## INSUFFICIENT_REWARD_BALANCE

Returned when the requested withdrawal exceeds the player's available balance.

## WITHDRAWAL_ALREADY_PROCESSING

Returned when the requested operation conflicts with an existing withdrawal state.

## WITHDRAWAL_ALREADY_SETTLED

Returned when the same withdrawal has already been successfully processed.

A settled withdrawal must not be paid again.

---

# Web3 / Blockchain Errors

## BLOCKCHAIN_SERVICE_UNAVAILABLE

Returned when required blockchain infrastructure is temporarily unavailable.

The withdrawal should remain in a recoverable state rather than being incorrectly marked as completed.

## BLOCKCHAIN_TRANSACTION_FAILED

Returned when a submitted blockchain transaction is confirmed to have failed.

## TRANSACTION_CONFIRMATION_PENDING

Used when a transaction has been submitted but its final status has not yet been established.

The application should not treat this state as either confirmed or definitively failed.

## PROGRAM_PAUSED

Returned when the proposed Solana settlement program has been placed into a paused state.

New settlements should not proceed until authorized operation resumes.

---

## Error Handling by Layer

### Mobile Client

The client should:

- Display understandable messages
- Avoid exposing technical secrets
- Preserve user input where appropriate
- Prevent accidental repeated financial requests
- Allow safe retry when permitted

### Backend

The backend should:

- Validate requests
- Return consistent error structures
- Log relevant failures
- Preserve financial state correctly
- Prevent duplicate processing

### Web3 Integration Layer

The Web3 service should distinguish between:

```text
Transaction preparation failure
Transaction submission failure
Transaction submitted / confirmation pending
Transaction confirmed
Transaction confirmed as failed
```

These states should not be treated as equivalent.

### Solana Program

The proposed program should reject invalid instructions without performing unauthorized settlement.

Examples include:

- Unauthorized signer
- Duplicate withdrawal
- Invalid token
- Invalid destination account
- Invalid amount
- Paused program

---

## Retry Strategy

Not every failed operation should automatically be retried.

### Safe Retry Examples

Potentially retryable operations may include:

- Temporary API failure
- Temporary RPC unavailability
- Read requests that do not change financial state

### Controlled Retry Required

Financial operations require additional verification before retrying.

Before retrying a withdrawal, the system should determine:

```text
Was a transaction submitted?
        ↓
Does a transaction signature exist?
        ↓
What is its blockchain status?
        ↓
Has this withdrawal already been settled?
        ↓
Is retry safe?
```

This reduces the risk of duplicate payments.

---

## Idempotency

Financially sensitive API operations should support idempotent behavior where appropriate.

For example, repeated submission of the same withdrawal request should not create multiple withdrawals or multiple settlements.

A production API may use an idempotency key or equivalent mechanism to identify repeated requests.

---

## Logging

Error logs should capture sufficient information for troubleshooting, such as:

- Request identifier
- Player identifier where appropriate
- Game session ID
- Reward ID
- Withdrawal ID
- Error code
- Transaction signature where applicable
- Timestamp

Logs must not contain:

- Private keys
- Seed phrases
- Authentication secrets
- Other protected credentials

---

## User-Facing vs Internal Errors

Internal technical failures should be translated into understandable user-facing messages.

For example:

```text
Internal:
BLOCKCHAIN_SERVICE_UNAVAILABLE

User-facing:
"Withdrawal processing is temporarily unavailable. Your reward balance has not been lost."
```

Detailed diagnostic information should remain within authorized system logs.

---

## Recovery Principle

A technical failure must not silently create an incorrect financial state.

When the final outcome of a withdrawal is uncertain, the system should preserve sufficient state for reconciliation before releasing funds, retrying settlement, or marking the operation as completed.

---

## Implementation Note

This error-handling model is part of the proposed Crypto Crush technical documentation.

Final error codes, retry policies, observability tooling, blockchain confirmation strategy, and incident-response procedures require engineering and security review before production implementation.
