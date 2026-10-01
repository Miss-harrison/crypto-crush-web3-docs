# Crypto Crush — Proposed Wallet Integration

## Overview

This document defines the proposed wallet integration for Crypto Crush.

The wallet system allows players to connect a Solana-compatible wallet to their Crypto Crush account for Web3-related functionality, including receiving eligible USDT withdrawals.

> **Status:** Proposed technical specification. The wallet architecture described here is not a confirmed production implementation.

---

## Integration Objectives

The proposed wallet integration should:

- Allow players to connect a supported Solana wallet
- Verify wallet ownership before enabling withdrawals
- Associate a verified wallet with a Crypto Crush player account
- Provide clear wallet connection and verification states
- Support USDT withdrawal destinations on Solana
- Prevent the application from accessing or storing player private keys or seed phrases

---

## Wallet Connection Flow

```text
Player selects "Connect Wallet"
        ↓
Compatible wallet provider opens
        ↓
Player approves connection
        ↓
Application receives public wallet address
        ↓
Backend creates verification challenge
        ↓
Player signs challenge using wallet
        ↓
Backend verifies signature
        ↓
Wallet marked as verified
        ↓
Wallet linked to player account
```

Connecting a wallet and verifying ownership should be treated as separate steps.

---

## Wallet Ownership Verification

A public wallet address alone does not prove that the player controls the wallet.

The proposed verification process uses a signed challenge.

### Step 1 — Request Challenge

```http
POST /wallets/challenge
```

### Example Request

```json
{
  "wallet_address": "<solana_wallet_address>",
  "network": "solana"
}
```

### Example Response

```json
{
  "challenge_id": "challenge_7841",
  "message": "Verify wallet ownership for Crypto Crush.",
  "expires_in": 300
}
```

The production challenge should include unique, time-limited data to reduce replay risk.

---

### Step 2 — Player Signs Challenge

The player's wallet prompts the player to sign the verification message.

Signing the message demonstrates control of the wallet without exposing the player's private key.

The application must never request:

- Private keys
- Seed phrases
- Recovery phrases

---

### Step 3 — Verify Signature

```http
POST /wallets/verify
```

### Example Request

```json
{
  "challenge_id": "challenge_7841",
  "wallet_address": "<solana_wallet_address>",
  "signature": "<wallet_signature>"
}
```

### Example Response

```json
{
  "wallet_id": "wallet_204",
  "network": "solana",
  "verification_status": "verified"
}
```

The backend should verify that:

- The challenge exists
- The challenge has not expired
- The challenge has not already been used
- The signature corresponds to the submitted wallet address

---

## Wallet States

A wallet connection may have one of the following proposed states:

| State | Description |
|---|---|
| `pending` | Wallet has been submitted but ownership is not verified |
| `verified` | Wallet ownership has been successfully verified |
| `rejected` | Verification failed |
| `disconnected` | Wallet is no longer active for the player account |

Only a verified wallet should be eligible for withdrawal processing.

---

## Wallet Information

The application may store information required to manage the wallet association, such as:

```json
{
  "wallet_id": "wallet_204",
  "player_id": "player_1024",
  "network": "solana",
  "wallet_address": "<solana_wallet_address>",
  "verification_status": "verified",
  "verified_at": "2026-10-01T15:00:00Z"
}
```

The application should not store wallet secrets.

---

## Withdrawal Destination

When requesting a withdrawal, the player selects a verified wallet associated with the account.

The system should validate:

- Wallet verification status
- Solana network compatibility
- Supported withdrawal asset
- Valid destination address
- Player withdrawal eligibility

For the proposed Crypto Crush reward model, the supported settlement asset is USDT on Solana.

---

## Changing a Wallet

Changing a withdrawal wallet should require additional verification.

Proposed flow:

1. Player requests a wallet change.
2. Existing authentication session is verified.
3. New wallet is connected.
4. New ownership challenge is generated.
5. Player signs the challenge.
6. Backend verifies the signature.
7. New wallet becomes eligible for future withdrawals.

Additional security controls may be required before a newly changed wallet can receive a withdrawal.

---

## Failure Scenarios

### Wallet Connection Rejected

The player rejects the wallet connection request.

**Expected result:** No wallet is associated with the account.

### Invalid Signature

The submitted signature cannot be verified against the wallet address.

**Expected result:** Verification fails and the wallet remains unverified.

### Expired Challenge

The verification challenge exceeds its permitted lifetime.

**Expected result:** The player must request a new challenge.

### Unsupported Network

The connected wallet or requested transaction uses an unsupported network.

**Expected result:** The operation is rejected before withdrawal processing.

### Blockchain Service Unavailable

Solana or the configured infrastructure provider is temporarily unavailable.

**Expected result:** The application should preserve the player's account state and allow the operation to be retried safely.

---

## Security Requirements

The proposed wallet integration should follow these principles:

- Never request or store private keys
- Never request seed or recovery phrases
- Verify wallet ownership before withdrawals
- Use unique, expiring verification challenges
- Prevent challenge reuse
- Validate wallet addresses server-side
- Protect wallet-management endpoints with authentication
- Log security-relevant wallet events
- Require additional review for suspicious wallet changes or withdrawal activity

---

## Implementation Note

This document defines a proposed wallet-integration approach for Crypto Crush.

Supported wallet providers, signing libraries, Solana RPC infrastructure, custody model, transaction-signing responsibilities, and production security controls must be selected and reviewed during engineering implementation.
