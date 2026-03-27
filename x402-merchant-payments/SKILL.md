---
name: x402 Merchant Payments
description: Execute x402 payment protocol operations — pre-request signing parameters, off-chain verification, and on-chain settlement via the facilitator proxy.
version: 1.0.0
metadata:
  openclaw:
    requires:
      env:
        - AIOT_API_BASE_URL
    primaryEnv: AIOT_API_BASE_URL
---

# x402 Merchant Payments

Use this skill when the user needs to prepare, verify, or settle an x402 payment, or check facilitator capabilities.

## Configuration

The default API base URL is `https://payment-api-dev.aiotnetwork.io`. All endpoints are relative to this URL.

To override (e.g. for local development):

```bash
export AIOT_API_BASE_URL="http://localhost:8080"
```

If `AIOT_API_BASE_URL` is not set, use `https://payment-api-dev.aiotnetwork.io` as the base for all requests.

## Available Tools

- `x402_pre_request` — Get Permit2 signing parameters for an x402 payment | `POST /api/v1/x402/proxy/pre-request` | Requires X-API-Key
- `x402_verify` — Verify a payment signature off-chain before settlement | `POST /api/v1/x402/proxy/verify` | Requires X-API-Key
- `x402_settle` — Submit payment settlement on-chain (facilitator pays gas) | `POST /api/v1/x402/proxy/settle` | Requires X-API-Key
- `x402_supported` — Get facilitator capabilities including supported version, payment schemes, and networks | `GET /api/v1/x402/proxy/supported` | Requires X-API-Key

## Recommended Flows

### Execute x402 Payment

Full payment flow from pre-request to settlement

1. Pre-request: POST /api/v1/x402/proxy/pre-request with {user_wallet_address, payee_address, token_address, amount, chain_id} — returns signing parameters (permit2, contracts, and eip7702_auth if approval is needed)
2. User signs the Permit2 witness transfer off-chain using the parameters from step 1 (zero gas for user)
3. Verify: POST /api/v1/x402/proxy/verify with {payment: {signature, token, amount, payer, payee, nonce, deadline, valid_after, chain_id}} — returns {valid, reason}
4. Settle: POST /api/v1/x402/proxy/settle with {payment: {signature, token, amount, payer, payee, nonce, deadline, valid_after, chain_id}, eip7702_auth (optional)} — returns {success, transaction, network, payer, reason}

### Check Facilitator Capabilities

1. Query: GET /api/v1/x402/proxy/supported — returns supported x402 version, payment schemes, and networks

## Rules

- All proxy endpoints require the X-API-Key header — this is the product API key, not a JWT Bearer token
- The API key is obtained during merchant registration (x402-merchant-auth skill)
- The payment flow must follow the order: pre-request, user signs, verify, settle. Never skip verify before settle.
- Pre-request returns needs_approval and eip7702_auth parameters when the user's wallet has not yet approved Permit2. Pass the signed EIP-7702 authorization to settle for gasless approval.
- The settle endpoint has a 65-second timeout due to blockchain transaction confirmation
- The facilitator pays all gas costs for settlement — zero gas for the user
- Verify checks signature recovery, deadline, token balance, Permit2 allowance, nonce usage, and payee match. Always verify before settling.

## Agent Guidance

Follow these instructions when executing this skill:

- Always follow the documented flow order. Do not skip steps.
- If a tool requires authentication, verify the session has a valid bearer token before calling it.
- If a tool requires a transaction PIN, ask the user for it fresh each time. Never cache or log PINs.
- Never expose, log, or persist secrets (passwords, tokens, full card numbers, CVVs).
- If the user requests an operation outside this skill's scope, decline and suggest the appropriate skill.
- If a step fails, check the error and follow the recovery guidance below before retrying.

- All endpoints in this skill use the X-API-Key header for authentication, not the Authorization Bearer header. The API key comes from merchant registration (x402-merchant-auth skill).
- The x402 payment flow is: `x402_pre_request` -> user signs off-chain -> `x402_verify` -> `x402_settle`. Always call `x402_verify` before `x402_settle` to avoid wasting gas on invalid signatures.
- Pre-request body requires: user_wallet_address, payee_address, token_address, amount (in smallest token units). Optional: chain_id.
- If `x402_pre_request` returns needs_approval: true, the response includes eip7702_auth with signing parameters. The user must sign an EIP-7702 authorization, which is then passed to `x402_settle` in the eip7702_auth field for gasless Permit2 approval.
- The verify and settle requests wrap payment details in a payment object.
- Settlement can take up to 65 seconds due to blockchain confirmation. Do not timeout prematurely.
- The facilitator pays all gas. The user only signs off-chain (zero gas cost).
- If a merchant does not yet have an API key, redirect them to the x402-merchant-auth skill to register first.
