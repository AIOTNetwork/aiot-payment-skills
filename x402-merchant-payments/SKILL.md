---
name: x402 Merchant Payments
description: x402 payment protocol proxy — create orders, request payment parameters, verify signatures, settle on-chain, and query facilitator capabilities.
version: 1.0.0
metadata:
  openclaw:
    requires:
      env:
        - AIOT_API_BASE_URL
    primaryEnv: AIOT_API_BASE_URL
---

# x402 Merchant Payments

Use this skill when the user needs to create payment orders, get signing parameters, verify user signatures, settle payments on-chain, or check facilitator capabilities.

All proxy endpoints require an `X-API-Key` header with the product's API key (obtained during registration or via key rotation in the x402-merchant-products skill).

## Configuration

The default API base URL is `https://payment-api-dev.aiotnetwork.io`. All endpoints are relative to this URL.

To override (e.g. for local development):

```bash
export AIOT_API_BASE_URL="http://localhost:8080"
```

If `AIOT_API_BASE_URL` is not set, use `https://payment-api-dev.aiotnetwork.io` as the base for all requests.

## Available Tools

- `x402_create_order` — Create a payment order and get a requestPayment URL | `POST /api/v1/x402/proxy/create-order` | Requires X-API-Key
- `x402_request_payment` — Get signing parameters for an existing order | `GET /api/v1/x402/proxy/request-payment` | Requires X-API-Key
- `x402_pre_request` — *(Deprecated: use create-order + request-payment)* Get Permit2 signing parameters | `POST /api/v1/x402/proxy/pre-request` | Requires X-API-Key
- `x402_verify` — Verify a payment signature off-chain before settlement | `POST /api/v1/x402/proxy/verify` | Requires X-API-Key
- `x402_settle` — Submit payment settlement on-chain (facilitator pays gas) | `POST /api/v1/x402/proxy/settle` | Requires X-API-Key
- `x402_supported` — Get facilitator capabilities including supported version, payment schemes, and networks | `GET /api/v1/x402/proxy/supported` | Requires X-API-Key

## Recommended Flows

### Order-Based Flow (Primary)

The recommended flow for processing x402 payments:

1. Create Order: POST /api/v1/x402/proxy/create-order with {payee_address, token_address, amount, merchant_id, order_id, path, chain_id (optional)} — returns order details + request_payment_url
2. Request Payment: GET /api/v1/x402/proxy/request-payment?merchant_id=...&order_id=...&user_address=... — returns Permit2 signing parameters
3. User signs the Permit2 witness transfer off-chain using the parameters from step 2 (zero gas for user)
4. Verify: POST /api/v1/x402/proxy/verify with {payment: {signature, token, amount, payer, payee, nonce (string), deadline (int64), valid_after (int64, optional), chain_id (optional)}, order_id} — returns {valid, reason}
5. Settle: POST /api/v1/x402/proxy/settle with {payment: {signature, token, amount, payer, payee, nonce (string), deadline (int64), valid_after (int64, optional), chain_id (optional)}, order_id, eip7702_auth (optional)} — returns {success, transaction, network, payer, reason}

### Legacy Pre-Request Flow (Deprecated)

> **Deprecated:** Use the order-based flow above for new integrations.

1. Pre-request: POST /api/v1/x402/proxy/pre-request with {user_wallet_address, payee_address, token_address, amount, order_id, path, chain_id (optional)} — returns signing parameters (permit2, contracts, and eip7702_auth if approval is needed)
2. User signs the Permit2 witness transfer off-chain using the parameters from step 1 (zero gas for user)
3. Verify: POST /api/v1/x402/proxy/verify with {payment, order_id} — returns {valid, reason}
4. Settle: POST /api/v1/x402/proxy/settle with {payment, order_id, eip7702_auth (optional)} — returns {success, transaction, network, payer, reason}

### Check Facilitator Capabilities

1. Query: GET /api/v1/x402/proxy/supported — returns supported x402 version, payment schemes, and networks

## Rules

- All proxy endpoints require the X-API-Key header — this is the product API key, not a JWT Bearer token
- The API key is obtained during merchant registration (x402-merchant-auth skill) or via key rotation (x402-merchant-products skill)
- Use the order-based flow (create-order + request-payment) for new integrations. pre-request is deprecated.
- The payment flow must follow the order: create order, request payment, user signs, verify, settle. Never skip verify before settle.
- Create-order requires: payee_address, token_address, amount, merchant_id, order_id, path. Optional: chain_id (defaults to facilitator's default chain).
- Request-payment requires query params: merchant_id, order_id, user_address.
- Pre-request returns needs_approval and eip7702_auth parameters when the user's wallet has not yet approved Permit2. Pass the signed EIP-7702 authorization to settle for gasless approval.
- The settle endpoint has a 65-second timeout due to blockchain transaction confirmation
- The facilitator pays all gas costs for settlement — zero gas for the user
- Verify checks signature recovery, deadline, token balance, Permit2 allowance, nonce usage, and payee match. Always verify before settling.
- amount is always in the token's smallest unit (e.g., wei for 18-decimal tokens, micro-units for 6-decimal tokens like USDC)
- order_id must be unique per payment — reusing an order ID will fail

## Agent Guidance

Follow these instructions when executing this skill:

- Always follow the documented flow order. Do not skip steps.
- If a tool requires authentication, verify the session has a valid bearer token before calling it.
- If a tool requires a transaction PIN, ask the user for it fresh each time. Never cache or log PINs.
- Never expose, log, or persist secrets (passwords, tokens, full card numbers, CVVs).
- If the user requests an operation outside this skill's scope, decline and suggest the appropriate skill.
- If a step fails, check the error and follow the recovery guidance below before retrying.

- **Primary flow:** `x402_create_order` -> `x402_request_payment` -> user signs off-chain -> `x402_verify` -> `x402_settle`. Always call `x402_verify` before `x402_settle` to avoid wasting gas on invalid signatures.
- All endpoints in this skill use the X-API-Key header for authentication, not the Authorization Bearer header. The API key comes from merchant registration (x402-merchant-auth skill) or key rotation (x402-merchant-products skill).
- When creating an order, all of payee_address, token_address, amount, merchant_id, order_id, and path are required. chain_id is optional and defaults to the facilitator's default chain.
- When requesting payment, pass merchant_id, order_id, and user_address as query parameters (not body).
- `x402_pre_request` is deprecated — only use it for legacy integrations that have not migrated to the order-based flow.
- Pre-request body requires: user_wallet_address, payee_address, token_address, amount (in smallest token units), order_id, path. Optional: chain_id.
- If `x402_pre_request` or `x402_request_payment` returns needs_approval: true, the response includes eip7702_auth with signing parameters. The user must sign an EIP-7702 authorization, which is then passed to `x402_settle` in the eip7702_auth field for gasless Permit2 approval.
- The verify and settle requests wrap payment details in a payment object and require a top-level order_id field. In the payment object, nonce is a string, deadline and valid_after are int64 (unix timestamp).
- Settlement can take up to 65 seconds due to blockchain confirmation. Do not timeout prematurely.
- The facilitator pays all gas. The user only signs off-chain (zero gas cost).
- Use `x402_supported` to check which chains and tokens the facilitator supports before creating an order.
- If a merchant does not yet have an API key, redirect them to the x402-merchant-auth skill to register or x402-merchant-products skill to create a product first.
