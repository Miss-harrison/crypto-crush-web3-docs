# Crypto Crush — Proposed Withdrawal Flow

## Overview

This document defines the proposed end-to-end withdrawal process for Crypto Crush.

The withdrawal flow converts eligible off-chain gameplay rewards into a USDT settlement to a verified player wallet through the Solana network.

> **Status:** Proposed technical specification. This workflow does not represent a confirmed or deployed production implementation.

---

## Withdrawal Requirements

Before a player can request a withdrawal, the proposed system should confirm that:

- The player is authenticated
- The player has reached the $5.00 minimum withdrawal threshold
- The requested amount does not exceed the available reward balance
- A verified Solana wallet is connected
- The requested asset is supported
- The player does not have a conflicting withdrawal already being processed

---

## End-to-End Flow

```text
Player
  ↓
Requests Withdrawal
  ↓
Withdrawal API
  ↓
Authentication Check
  ↓
Reward Balance Check
  ↓
$5 Threshold Check
  ↓
Verified Wallet Check
  ↓
Reserve Reward Balance
  ↓
Create Withdrawal Record
  ↓
Web3 Integration Service
  ↓
Authorized Settlement Request
  ↓
Solana Program Validation
  ↓
USDT Token Transfer
  ↓
Solana Confirmation
  ↓
Backend Reconciliation
  ↓
Withdrawal Completed
```

---

## Step 1 — Player Requests Withdrawal

The authenticated player initiates a withdrawal through the Crypto Crush application.

Proposed endpoint:

```http
POST /withdrawals
```

Example request:

```json
{
  "amount_usd": 5.00,
  "asset": "USDT",
  "network": "solana",
  "wallet_id": "wallet_204"
}
```

The client requests the withdrawal but does not authorize the blockchain settlement itself.

---

## Step 2 — Validate Player

The backend validates the player's authenticated session and confirms that the withdrawal request belongs to the authenticated account.

If authentication fails, processing stops.

Example error:

```json
{
  "error": {
    "code": "AUTH_REQUIRED",
    "message": "Valid authentication is required."
  }
}
```

---

## Step 3 — Validate Reward Balance

The reward service retrieves the player's available balance.

Example:

```text
Available balance: $7.25
Requested withdrawal: $5.00
Minimum threshold: $5.00
```

Because the requested amount meets the threshold and does not exceed the available balance, the request may continue.

---

## Step 4 — Verify Wallet

The selected wallet must:

- Belong to the authenticated player account
- Have completed ownership verification
- Be compatible with Solana
- Provide a valid destination for the supported USDT token

An unverified wallet must not be used for withdrawal settlement.

---

## Step 5 — Reserve Reward Balance

Before blockchain processing begins, the requested amount is reserved.

Example:

```text
Before withdrawal

Available: $7.25
Reserved:  $0.00

After $5.00 reservation

Available: $2.25
Reserved:  $5.00
```

This prevents the same reward balance from being used in another simultaneous withdrawal.

---

## Step 6 — Create Withdrawal Record

The backend creates a unique withdrawal record.

Example:

```json
{
  "withdrawal_id": "wd_9021",
  "player_id": "player_1024",
  "wallet_id": "wallet_204",
  "amount_usd": 5.00,
  "asset": "USDT",
  "network": "solana",
  "status": "approved"
}
```

The unique withdrawal ID is used to maintain traceability throughout settlement.

---

## Step 7 — Prepare Web3 Transaction

The Web3 integration service receives the approved withdrawal.

Its proposed responsibilities include:

- Retrieving approved withdrawal information
- Validating settlement parameters
- Preparing the Solana transaction
- Providing the required program accounts
- Submitting the authorized settlement instruction

The mobile client should not have access to treasury signing credentials.

---

## Step 8 — Solana Program Validation

The proposed Solana program validates the withdrawal before allowing settlement.

Checks may include:

- Authorized settlement signer
- Unique withdrawal identifier
- Withdrawal not previously processed
- Supported token mint
- Valid destination token account
- Valid withdrawal amount
- Program operational status

If validation fails, the token transfer should not proceed.

---

## Step 9 — USDT Settlement

After successful validation, the approved USDT amount is transferred from the configured settlement source to the player's valid destination token account.

Conceptually:

```text
Crypto Crush Settlement Treasury
             ↓
      Solana Program
             ↓
        USDT Transfer
             ↓
     Player Token Account
```

---

## Step 10 — Transaction Monitoring

Submitting a blockchain transaction does not automatically mean settlement has succeeded.

The backend should record the transaction signature and monitor its status.

Proposed withdrawal state:

```text
processing
```

The system waits for the required confirmation state before finalizing the withdrawal.

---

## Step 11 — Successful Confirmation

After successful blockchain confirmation:

```text
Withdrawal → confirmed
Reserved Reward → withdrawn
Transaction Signature → stored
Player Balance → updated
```

Example response:

```json
{
  "withdrawal_id": "wd_9021",
  "status": "confirmed",
  "asset": "USDT",
  "amount_usd": 5.00,
  "transaction_signature": "<solana_transaction_signature>"
}
```

---

## Failed Withdrawal

A withdrawal may fail because of:

- Invalid transaction parameters
- Insufficient settlement funds
- Blockchain/RPC failure
- Program validation failure
- Invalid destination account
- Unsupported token configuration

A failed transaction must not automatically result in the player's reserved balance being permanently lost.

The system should determine whether the transaction definitively failed before releasing reserved rewards.

---

## Retry Handling

Blockchain retries require special care because repeating a payment operation can create duplicate settlement.

Before retrying, the system should check:

1. Whether a transaction signature already exists
2. Whether the previous transaction was confirmed
3. Whether the withdrawal ID has already been processed on-chain
4. Whether the reward amount remains reserved
5. Whether retrying can occur safely

The same withdrawal must not be paid twice.

---

## Proposed Withdrawal States

| Status | Description |
|---|---|
| `pending` | Withdrawal request received |
| `approved` | Application validation completed |
| `processing` | Blockchain settlement is in progress |
| `confirmed` | USDT settlement successfully confirmed |
| `failed` | Settlement failed |
| `rejected` | Withdrawal failed application validation |
| `review` | Withdrawal requires investigation before further action |

---

## Traceability

A completed withdrawal should be traceable across:

```text
Player ID
   ↓
Reward Records
   ↓
Withdrawal ID
   ↓
Wallet ID
   ↓
Solana Transaction Signature
```

This provides a clear relationship between gameplay rewards and blockchain settlement.

---

## Security Considerations

The withdrawal process should:

- Require authenticated requests
- Use verified wallets
- Prevent duplicate withdrawals
- Reserve balances before settlement
- Restrict settlement authorization
- Validate token and destination accounts
- Protect signing credentials
- Record state transitions
- Separate transaction submission from confirmation
- Support investigation of uncertain transaction states

---

## Implementation Note

This withdrawal workflow is a proposed technical design for the Crypto Crush development stage.

Production implementation requires engineering validation, security review, treasury and key-management design, blockchain infrastructure decisions, and applicable legal/compliance review before handling real player funds.
